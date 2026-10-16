# MCP_INTEGRATION.md — 麦当劳 MCP 集成说明

本文档说明「麦麦优选智能体」实际使用的麦当劳官方 MCP Server（`mcd-mcp`）、每个模块调用的 Tool、参数传递方式与返回结果处理。全部使用官方 MCP 工具与官方参数规则，**未自定义、未伪造任何 MCP 参数**。

## 一、使用的 MCP Server

| 项      | 说明                                                                |
| ------ | ----------------------------------------------------------------- |
| Server | `mcd-mcp`（麦当劳中国官方 MCP Server）                                     |
| 接入方式   | MCP 标准协议，配置见 `mcp-config.example.json`（凭证用 `${MCP_TOKEN}` 环境变量占位） |
| 调用原则   | 只调用查询/计价类工具；**从不调用 `create-order`**（比赛 Demo 不下单不支付）               |

## 二、使用的 Tool 总览

| Tool                       | 用途              | 本项目场景                    |
| -------------------------- | --------------- | ------------------------ |
| `query-nearby-stores`      | 到店场景门店查询        | 模块1门店定位、模块2门店筛选、彩蛋模块门店匹配 |
| `delivery-query-addresses` | 查询用户配送地址列表      | 模块3麦麦送侧前置                |
| `delivery-create-address`  | 创建配送地址          | 模块3用户无地址时引导（可选）          |
| `delivery-query-stores`    | 外送可配送门店查询       | 模块1/3 麦麦送侧门店+beCode      |
| `query-meals`              | 餐品列表（菜单）        | 全部模块：商品 code 映射、库存判定     |
| `query-meal-detail`        | 餐品详情（套餐组成/特调）   | 模块1用户询问餐品组成时（可选）         |
| `query-store-coupons`      | 指定门店+渠道可用优惠券    | 模块1/3 优惠券匹配              |
| `query-my-coupons`         | 用户卡包优惠券         | 用户主动询问"我有什么券"时展示（可选）     |
| `calculate-price`          | 计价（含优惠、配送费、打包费） | 模块1/3 价格计算               |
| `list-nutrition-foods`     | 常见餐品营养成分        | 模块1 营养信息被动展示             |

## 三、渠道参数规则（所有渠道类 Tool 通用）

| 渠道       | beType | orderType | beCode                        |
| -------- | ------ | --------- | ----------------------------- |
| 堂食自取     | 1      | 1         | **不传**（传了报错）                  |
| 得来速      | 5      | 1         | 来自 `query-nearby-stores` 返回   |
| 麦麦送（麦乐送） | 2      | 2         | 来自 `delivery-query-stores` 返回 |

价格单位：`calculate-price` 返回字段单位为**分**，展示时 ÷100 转为元。

## 四、各模块调用链

### 模块 1【多方案点餐搭配】

```
query-nearby-stores (beType=1, searchType=2, city, keyword)
  └→ 取 data[].storeCode（beType=5 时同时取 beCode）
query-meals (storeCode, beType=1, orderType=1)
  └→ categories[].meals[].code + meals{code} 名称/价格映射：
     将用户清单逐项映射到商品 code；tags 含「单品」→ 情况B（仅单点）；
     菜单找不到 → 情况A（无库存）候选，待计价确认
query-store-coupons (storeCode, beType, orderType)
  └→ data[].couponId / couponCode / products[].productCode：
     仅匹配到清单商品 code 的券才可绑定
calculate-price (storeCode, beType, orderType, items=[{productCode, quantity, couponId?, couponCode?}])
  └→ data.price（实付，分）、data.originalPrice（原价）、data.discount（优惠）、
     data.productList：为空或缺项 → 确认情况A「当前门店暂无库存」
     （验证实例：不可售商品 9900016371 计价返回 productList=[]、price=0）
list-nutrition-foods ()
  └→ 按方案内商品名匹配，汇总总热量（kJ）与钠含量（mg），随方案被动展示
```

- 方案1 价格偏高判断：若存在可承接同样餐品的更优惠套餐/随心配组合，则重排后再次 `calculate-price` 验证，生成方案2，标注删减/替换项及（拼单时）成员归属。
- 多人拼单拆账：按 `data.productList[].subtotal` 逐项归属成员，优惠 `discount` 按各成员小计占比分摊，输出每人应付金额。

### 模块 2【门店筛选（轻量化）】

```
query-nearby-stores (beType=1, searchType=2, city, keyword)
  └→ data[]：storeName / address / distance / storeCode / businessStatus /
     businessStartTime / businessEndTime / reservation
筛选逻辑（仅用返回字段）：
  - 24小时营业：businessStartTime="00:00" 且 businessEndTime="23:59"
    （验证实例：深圳华强北餐厅 1420077 为 00:00-23:59，判定 24h）
  - 甜品站/停车位：接口无设施字段，仅按名称/地址关键字推断并如实标注
query-meals (storeCode, beType=1, orderType=1)
  └→ 限定款库存：tags（上新/限定/限量/周边）+ 商品名匹配 code；
     有 code+价格 → 在售；无 → 未上架/暂无库存
     （验证实例：1420077 菜单中「旋旋水手帽」521929，tags 含「汪苏泷限量周边」）
```

### 模块 3【堂食 VS 麦麦送比价】

```
堂食侧：
  query-meals (storeCode, beType=1, orderType=1)            → 堂食 code 与价格
  query-store-coupons (storeCode, beType=1, orderType=1)    → 堂食券
  calculate-price (beType=1, orderType=1, items)            → 无配送费总价

麦麦送侧：
  delivery-query-addresses ()
    └→ data.addresses[].addressId 交给用户选择
  delivery-query-stores (addressId, beType=2)
    └→ data[].storeCode + data[].beCode
       （验证实例：1450098 / beCode 145009802）
  query-meals (storeCode, beType=2, orderType=2, beCode)
    └→ 外送专用菜单：⚠️ 商品 code 与堂食不同（外送为 99000029xx/90xxxx 系列，
       套餐 tag 含「外送价格」），必须在此外送菜单中重新映射 code；
       复用堂食 code 计价会报 {"success":false,"code":60006,"message":"商品已售罄或不可售…"}
       （验证实例：外送板烧套餐=9900002996 ¥42，薯条=501205 ¥15.5）
  query-store-coupons (storeCode, beType=2, orderType=2, beCode) → 麦麦送券（独立券体系）
  calculate-price (beType=2, orderType=2, beCode, items)
    └→ data.deliveryPrice：配送费（分），并入麦麦送总价
       data.packingPrice：打包服务费（分）
       data.price：实付总价 = 餐品 + 配送 + 打包 − 优惠
       （验证实例：餐品 5750 + 配送 600 + 打包 270 = 6620 分 = 66.20 元）
```

输出对比明细：① 堂食（原价−优惠=总价，无配送费）② 麦麦送（原价−外卖券+配送费+打包费=实付），附客观场景参考，不替用户做决定。金额全部来自当次实时查询。

### 彩蛋模块【限定商品库存独立查询】

```
query-nearby-stores (beType=1, searchType=2, city, keyword)  → 用户只给门店名时定位
query-meals (storeCode, beType=1, orderType=1)               → tags/名称匹配限定款 code
  └→ 在售（报 code+价格）｜未上架/暂无库存｜多候选列出让用户确认
```

## 五、返回结果通用处理规则

1. **成功判定**：`success=true && code=200`；否则把 `code`/`message` 原样告知用户并给下一步建议（如 60006 → 提示换商品或检查渠道参数）。
2. **价格**：全部分→元换算后展示，保留一位小数；优惠分「原价 / 优惠抵扣 / 实付」三行呈现。
3. **库存判定**：以 `query-meals` 是否含该 code 为初筛，以 `calculate-price` 的 `productList` 是否包含该项为最终确认；两者不一致时以计价结果为准并提示「暂无库存/不可售」。
4. **优惠券**：`query-store-coupons` 返回的券仅当 `products[].productCode` 与清单商品匹配时才绑定到对应 items 项（传 `couponId` + `couponCode`）；不匹配的券只展示不硬绑。
5. **实时性**：所有价格、库存、券均当场查询，不做缓存估算；数据仅供参考，不构成交易承诺。
