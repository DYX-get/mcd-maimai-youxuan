#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""麦麦优选智能体（maimai-youxuan）命令行演示程序

麦当劳程序员节创意开发大赛参赛作品 · 单人参赛

特性
----
* 通过 stdio 直连官方 mcd-mcp MCP Server，调用官方工具，未自定义任何 MCP 参数
* 覆盖四大模块：核心点餐搭配 / 门店筛选 / 堂食 VS 麦麦送比价 / 限定商品库存查询
* 只做查询与计价，**不调用任何下单、支付类工具**
* 仅使用 Python 标准库，无需安装第三方依赖

用法示例
--------
    python maimai_youxuan.py tools
    python maimai_youxuan.py stores --city 上海 --keyword 松江 --open24
    python maimai_youxuan.py menu --store 1450038 --grep 龙焰
    python maimai_youxuan.py coupons --store 1450038
    python maimai_youxuan.py price --store 1450038 --item 9900005462:1 --item 501205:1
    python maimai_youxuan.py stock --store 1450271 --keyword 月堡
    python maimai_youxuan.py compare --store 1450038 --address-id <地址ID> \
        --dinein-item 9900005462:1 --delivery-item 9900002996:1

环境变量
--------
    MCP_TOKEN   麦当劳 MCP 访问令牌（也可写入 mcp-config.example.json 同级 .env）
"""

from __future__ import annotations

import argparse
import json
import os
import shutil
import subprocess
import sys
import threading
import time
from pathlib import Path
from typing import Any, Dict, Iterable, List, Optional

CENT_PER_YUAN = 100
DEFAULT_CONFIG = Path(__file__).with_name("mcp-config.example.json")
ORDER_TOOLS = ("create-order", "main-order", "pay", "refund")


# --------------------------------------------------------------------------- #
# MCP stdio 客户端
# --------------------------------------------------------------------------- #
class McpError(RuntimeError):
    """MCP 调用异常。"""


class McpStdioClient:
    """极简 MCP stdio 客户端：JSON-RPC 2.0，按行分隔。"""

    def __init__(self, command: str, args: Iterable[str] = (), env: Optional[Dict[str, str]] = None,
                 timeout: float = 40.0) -> None:
        full_env = os.environ.copy()
        full_env.update(env or {})
        self._timeout = timeout
        self._next_id = 0
        self._stderr: List[str] = []
        self._proc = subprocess.Popen(
            [command, *args],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            env=full_env,
            text=True,
            encoding="utf-8",
            errors="replace",
            bufsize=1,
        )
        threading.Thread(target=self._drain_stderr, daemon=True).start()

    # -- 内部工具 ---------------------------------------------------------- #
    def _drain_stderr(self) -> None:
        try:
            for line in self._proc.stderr:  # type: ignore[union-attr]
                self._stderr.append(line.rstrip())
                if len(self._stderr) > 300:
                    del self._stderr[0]
        except Exception:  # pragma: no cover - 进程退出时忽略
            pass

    def _write(self, payload: Dict[str, Any]) -> None:
        if self._proc.stdin is None or self._proc.poll() is not None:
            raise McpError("MCP 服务端未运行。" + self._stderr_tail())
        self._proc.stdin.write(json.dumps(payload, ensure_ascii=False) + "\n")
        self._proc.stdin.flush()

    def _stderr_tail(self) -> str:
        tail = "\n".join(self._stderr[-10:])
        return f"\n服务端输出：\n{tail}" if tail else ""

    def _read(self) -> Optional[Dict[str, Any]]:
        if self._proc.stdout is None:
            return None
        line = self._proc.stdout.readline()
        if not line:
            raise McpError("MCP 服务端连接已关闭。" + self._stderr_tail())
        line = line.strip()
        if not line:
            return None
        try:
            return json.loads(line)
        except json.JSONDecodeError:
            return None  # 忽略非 JSON 日志行

    # -- 协议方法 ---------------------------------------------------------- #
    def request(self, method: str, params: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        self._next_id += 1
        rid = self._next_id
        self._write({"jsonrpc": "2.0", "id": rid, "method": method, "params": params or {}})
        deadline = time.time() + self._timeout
        while time.time() < deadline:
            msg = self._read()
            if msg is None:
                continue
            if msg.get("id") == rid:
                if "error" in msg:
                    raise McpError(f"{method} 调用失败：{msg['error']}")
                return msg.get("result") or {}
        raise McpError(f"{method} 调用超时（{self._timeout:.0f}s）")

    def notify(self, method: str, params: Optional[Dict[str, Any]] = None) -> None:
        self._write({"jsonrpc": "2.0", "method": method, "params": params or {}})

    def initialize(self) -> Dict[str, Any]:
        result = self.request("initialize", {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {"name": "maimai-youxuan", "version": "1.0.0"},
        })
        self.notify("notifications/initialized")
        return result

    def list_tools(self) -> List[Dict[str, Any]]:
        return list(self.request("tools/list").get("tools") or [])

    def call_tool(self, name: str, arguments: Dict[str, Any]) -> Any:
        for banned in ORDER_TOOLS:
            if banned in name:
                raise McpError(f"已拦截下单/支付类工具调用：{name}（本项目只做规划，不下单）")
        result = self.request("tools/call", {"name": name, "arguments": arguments})
        return self._unwrap(result)

    @staticmethod
    def _unwrap(result: Dict[str, Any]) -> Any:
        """把 MCP tool 返回内容解成本地 JSON（失败时回退原始文本）。"""
        if isinstance(result.get("structuredContent"), (dict, list)):
            return result["structuredContent"]
        chunks = [c.get("text", "") for c in result.get("content") or [] if isinstance(c, dict)]
        text = "\n".join(chunks).strip()
        if not text:
            return result
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            return {"raw": text}

    def close(self) -> None:
        try:
            if self._proc.stdin:
                self._proc.stdin.close()
        except Exception:
            pass
        try:
            self._proc.terminate()
            self._proc.wait(timeout=5)
        except Exception:
            self._proc.kill()


# --------------------------------------------------------------------------- #
# 配置加载
# --------------------------------------------------------------------------- #
def resolve_command(command: str) -> str:
    """在 Windows 上兼容 npx / npx.cmd 这类可执行文件命名。"""
    found = shutil.which(command)
    if found is None and os.name == "nt":
        for suffix in (".cmd", ".exe", ".bat"):
            found = shutil.which(command + suffix)
            if found:
                break
    if found is None:
        raise SystemExit(
            f"未找到 MCP 启动命令「{command}」。请确认已安装对应的 MCP Server，"
            f"或在 mcp-config.example.json 中改成可执行文件的绝对路径。"
        )
    return found


def load_server(config_path: Path = DEFAULT_CONFIG) -> McpStdioClient:
    """从 mcp-config.example.json 读取连接参数，并注入 MCP_TOKEN。"""
    if not config_path.exists():
        raise SystemExit(f"未找到 MCP 配置文件：{config_path}")
    cfg = json.loads(config_path.read_text(encoding="utf-8"))
    servers = cfg.get("mcpServers") or {}
    if not servers:
        raise SystemExit("配置文件中缺少 mcpServers 字段。")
    name, spec = next(iter(servers.items()))
    command = spec.get("command")
    if not command:
        raise SystemExit(f"mcpServers.{name} 缺少 command 字段。")

    env = dict(spec.get("env") or {})
    for key, value in list(env.items()):
        if isinstance(value, str) and "${" in value:
            var = value.strip("${}")
            env[key] = os.environ.get(var, "")
            if not env[key]:
                print(f"⚠️  环境变量 {var} 为空，请在启动前设置（如 export {var}=<你的 Token>）。",
                      file=sys.stderr)

    client = McpStdioClient(resolve_command(command), spec.get("args") or [], env)
    client.initialize()
    return client


# --------------------------------------------------------------------------- #
# 渠道参数规则（官方 MCP 参数，不做自定义）
# --------------------------------------------------------------------------- #
def channel_args(channel: str, be_code: Optional[str] = None) -> Dict[str, Any]:
    """堂食自取 / 得来速 / 麦麦送 的 beType、orderType、beCode 组合。"""
    if channel == "dinein":
        return {"beType": 1, "orderType": 1}
    if channel == "drivethru":
        if not be_code:
            raise SystemExit("得来速渠道必须提供 --be-code。")
        return {"beType": 5, "orderType": 1, "beCode": be_code}
    if channel == "delivery":
        if not be_code:
            raise SystemExit("麦麦送渠道必须提供 --be-code（由 delivery-query-stores 获取）。")
        return {"beType": 2, "orderType": 2, "beCode": be_code}
    raise SystemExit(f"未知渠道：{channel}")


def yuan(cents: Any) -> str:
    """官方接口价格单位为分，展示需 ÷100。"""
    try:
        return f"¥{int(cents) / CENT_PER_YUAN:.1f}"
    except (TypeError, ValueError):
        return "¥—"


def is_24h(store: Dict[str, Any]) -> bool:
    return store.get("businessStartTime") == "00:00" and store.get("businessEndTime") == "23:59"


def parse_items(raw_items: List[str]) -> List[Dict[str, Any]]:
    items = []
    for raw in raw_items:
        code, _, qty = raw.partition(":")
        items.append({"productCode": code.strip(), "quantity": int(qty or 1)})
    if not items:
        raise SystemExit("至少需要一个 --item 商品编码，格式 商品code:数量。")
    return items


# --------------------------------------------------------------------------- #
# 业务模块
# --------------------------------------------------------------------------- #
class MaiMaiYouXuan:
    def __init__(self, client: McpStdioClient) -> None:
        self.mcp = client

    # 模块 2：门店筛选 ---------------------------------------------------- #
    def stores(self, city: str, keyword: str = "", limit: int = 10) -> List[Dict[str, Any]]:
        args: Dict[str, Any] = {"searchType": 2, "city": city}
        if keyword:
            args["keyword"] = keyword
        data = self.mcp.call_tool("query-nearby-stores", args)
        rows = data.get("data") or data.get("stores") or data.get("list") or []
        if isinstance(rows, dict):
            rows = rows.get("list") or []
        return rows[:limit]

    # 模块 1 / 3：菜单、优惠券、计价 --------------------------------------- #
    def menu(self, store_code: str, channel: str = "dinein", be_code: Optional[str] = None,
             keyword: str = "") -> List[Dict[str, Any]]:
        args = {"storeCode": store_code, **channel_args(channel, be_code)}
        data = self.mcp.call_tool("query-meals", args)
        rows = data.get("data") or data.get("productList") or []
        if isinstance(rows, dict):
            rows = rows.get("productList") or rows.get("list") or []
        if keyword:
            rows = [r for r in rows if keyword in str(r.get("productName", ""))
                    or keyword in json.dumps(r.get("tags") or [], ensure_ascii=False)]
        return rows

    def coupons(self, store_code: str, channel: str = "dinein") -> List[Dict[str, Any]]:
        order_type = 2 if channel == "delivery" else 1
        data = self.mcp.call_tool("query-store-coupons",
                                  {"storeCode": store_code, "orderType": order_type})
        rows = data.get("data") or data.get("couponList") or []
        if isinstance(rows, dict):
            rows = rows.get("couponList") or rows.get("list") or []
        return rows

    def price(self, store_code: str, items: List[Dict[str, Any]], channel: str = "dinein",
              be_code: Optional[str] = None) -> Dict[str, Any]:
        args = {"storeCode": store_code, "productList": items, **channel_args(channel, be_code)}
        return self.mcp.call_tool("calculate-price", args)

    # 模块 3：麦麦送链路 --------------------------------------------------- #
    def delivery_addresses(self) -> List[Dict[str, Any]]:
        data = self.mcp.call_tool("delivery-query-addresses", {})
        rows = data.get("data") or data.get("addressList") or []
        if isinstance(rows, dict):
            rows = rows.get("list") or []
        return rows

    def delivery_stores(self, address_id: str) -> List[Dict[str, Any]]:
        data = self.mcp.call_tool("delivery-query-stores",
                                  {"addressId": address_id, "beType": 2})
        rows = data.get("data") or data.get("storeList") or []
        if isinstance(rows, dict):
            rows = rows.get("list") or []
        return rows

    # 彩蛋模块：限定商品库存 ------------------------------------------------ #
    def stock(self, store_code: str, keyword: str) -> List[Dict[str, Any]]:
        hits = self.menu(store_code, keyword=keyword)
        result = []
        for item in hits:
            result.append({
                "name": item.get("productName"),
                "code": item.get("productCode"),
                "price": yuan(item.get("price")),
                "tags": item.get("tags"),
                "status": "在售",
            })
        return result


# --------------------------------------------------------------------------- #
# 命令行输出
# --------------------------------------------------------------------------- #
def print_stores(rows: List[Dict[str, Any]], only_24h: bool = False) -> None:
    if only_24h:
        rows = [r for r in rows if is_24h(r)]
    if not rows:
        print("未查询到符合条件的门店。")
        return
    header = f"{'门店名称':<34}{'距离':>9}  {'营业时间':<14}{'storeCode':<10}"
    print(header)
    print("-" * 78)
    for row in rows:
        name = str(row.get("storeName", ""))[:32]
        distance = row.get("distance")
        dist = f"{distance} m" if isinstance(distance, int) else str(distance or "-")
        hours = f"{row.get('businessStartTime', '-')}-{row.get('businessEndTime', '-')}"
        flag = " 24h" if is_24h(row) else ""
        print(f"{name:<34}{dist:>9}  {hours + flag:<14}{row.get('storeCode', '-'):<10}")


def print_menu(rows: List[Dict[str, Any]], limit: int = 40) -> None:
    if not rows:
        print("未匹配到商品（可能该门店未上架/暂无库存）。")
        return
    for row in rows[:limit]:
        tags = row.get("tags") or []
        tag_text = "/".join(map(str, tags)) if isinstance(tags, list) else str(tags)
        print(f"{row.get('productName', '-')[:28]:<30}{yuan(row.get('price')):>8}  "
              f"{row.get('productCode', '-'):<12}{tag_text}")
    if len(rows) > limit:
        print(f"...（共 {len(rows)} 项，已显示前 {limit} 项）")


def print_price(result: Dict[str, Any], title: str) -> float:
    if not result or result.get("success") is False:
        print(f"{title}：计价失败 {result}")
        return 0.0
    total = result.get("totalPrice") or result.get("payPrice") or 0
    delivery = result.get("deliveryPrice") or 0
    packing = result.get("packingPrice") or 0
    items = result.get("productList") or []
    print(f"\n【{title}】")
    for item in items:
        qty = item.get("quantity", 1)
        print(f"  - {item.get('productName', '-')[:26]:<28}x{qty}  {yuan(item.get('price'))}")
    if delivery or packing:
        print(f"  配送费 {yuan(delivery)}｜打包费 {yuan(packing)}")
    print(f"  实付合计：{yuan(total)}")
    return int(total) / CENT_PER_YUAN


# --------------------------------------------------------------------------- #
# 入口
# --------------------------------------------------------------------------- #
def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="麦麦优选智能体命令行演示（只做规划，不下单）")
    sub = parser.add_subparsers(dest="command", required=True)

    sub.add_parser("tools", help="列出 MCP Server 提供的官方工具")

    p_stores = sub.add_parser("stores", help="模块2：附近门店筛选")
    p_stores.add_argument("--city", required=True)
    p_stores.add_argument("--keyword", default="")
    p_stores.add_argument("--open24", action="store_true", help="仅显示 24 小时门店")
    p_stores.add_argument("--limit", type=int, default=10)

    p_menu = sub.add_parser("menu", help="模块1：门店菜单查询")
    p_menu.add_argument("--store", required=True)
    p_menu.add_argument("--channel", default="dinein", choices=["dinein", "drivethru", "delivery"])
    p_menu.add_argument("--be-code")
    p_menu.add_argument("--grep", default="")
    p_menu.add_argument("--limit", type=int, default=40)

    p_coupon = sub.add_parser("coupons", help="模块1：门店优惠券查询")
    p_coupon.add_argument("--store", required=True)
    p_coupon.add_argument("--channel", default="dinein", choices=["dinein", "delivery"])

    p_price = sub.add_parser("price", help="单渠道计价")
    p_price.add_argument("--store", required=True)
    p_price.add_argument("--channel", default="dinein", choices=["dinein", "drivethru", "delivery"])
    p_price.add_argument("--be-code")
    p_price.add_argument("--item", action="append", default=[], help="商品code:数量")

    p_stock = sub.add_parser("stock", help="彩蛋：限定商品库存查询")
    p_stock.add_argument("--store", required=True)
    p_stock.add_argument("--keyword", required=True)

    p_cmp = sub.add_parser("compare", help="模块3：堂食 VS 麦麦送比价")
    p_cmp.add_argument("--store", required=True, help="堂食门店 storeCode")
    p_cmp.add_argument("--address-id", default="", help="麦麦送收货地址ID，留空则自动取第一条")
    p_cmp.add_argument("--dinein-item", action="append", default=[], help="堂食商品code:数量")
    p_cmp.add_argument("--delivery-item", action="append", default=[], help="外送商品code:数量（与堂食不同）")
    return parser


def main(argv: Optional[List[str]] = None) -> int:
    args = build_parser().parse_args(argv)
    try:
        client = load_server()
    except McpError as exc:
        raise SystemExit(f"MCP 连接失败：{exc}") from None
    helper = MaiMaiYouXuan(client)
    try:
        if args.command == "tools":
            for tool in client.list_tools():
                print(f"{tool.get('name'):<28}{tool.get('description', '')[:60]}")
            return 0

        if args.command == "stores":
            print_stores(helper.stores(args.city, args.keyword, args.limit), args.open24)

        elif args.command == "menu":
            print_menu(helper.menu(args.store, args.channel, args.be_code, args.grep), args.limit)

        elif args.command == "coupons":
            rows = helper.coupons(args.store, args.channel)
            if not rows:
                print("当前门店无可领取优惠券。")
            for row in rows:
                print(f"{str(row.get('couponName', '-'))[:30]:<32}"
                      f"{row.get('couponType', '')}  {row.get('couponCode', '')}")

        elif args.command == "price":
            items = parse_items(args.item)
            print_price(helper.price(args.store, items, args.channel, args.be_code), "单渠道计价")

        elif args.command == "stock":
            rows = helper.stock(args.store, args.keyword)
            print_menu(rows) if rows else print("未上架或暂无库存，建议换门店或稍后再查。")

        elif args.command == "compare":
            dinein_items = parse_items(args.dinein_item)
            delivery_items = parse_items(args.delivery_item) if args.delivery_item else dinein_items

            address_id = args.address_id
            if not address_id:
                addresses = helper.delivery_addresses()
                if not addresses:
                    print("账户下没有可用收货地址，请先在麦当劳官方渠道添加。")
                    return 1
                address_id = str(addresses[0].get("addressId") or addresses[0].get("id"))
            stores = helper.delivery_stores(address_id)
            if not stores:
                print("该地址暂无可配送门店。")
                return 1
            d_store = stores[0]
            d_code = str(d_store.get("storeCode"))
            d_be = str(d_store.get("beCode"))

            dinein_total = print_price(
                helper.price(args.store, dinein_items, "dinein"), "① 堂食（到店自取）")
            delivery_total = print_price(
                helper.price(d_code, delivery_items, "delivery", d_be),
                f"② 麦麦送（{d_store.get('storeName', d_code)} 配送）")

            diff = delivery_total - dinein_total
            print(f"\n结论：堂食 {dinein_total:.1f} 元 vs 麦麦送 {delivery_total:.1f} 元，"
                  f"{'堂食更省' if diff >= 0 else '麦麦送更省'} {abs(diff):.1f} 元。")
            print("提示：两渠道券体系独立；外送商品 code 与堂食不同，需分别映射。本程序不下单不支付。")
        return 0
    finally:
        client.close()


if __name__ == "__main__":
    sys.exit(main())
