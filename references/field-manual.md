# 接口字段手册

调用麦当劳点餐接口前先读本文件。字段名来自真实只读调用，单位陷阱是本 Skill 最容易出错的地方。

## 一、时间与账号

`now-time-info`：无入参。返回包含服务器时间与 `dayOfWeek`。`dayOfWeek` 是星期券过滤的必需输入。

`query-my-account`：无入参。返回：

```
availablePoint            可用积分（小数，不是整数）
accumulativePoint         累计积分
currentMouthExpirePoint   本月将过期积分
nextMouthExpirePoint      下月将过期积分
expiredPoint              已过期积分
frozenPoint               冻结积分
usedPoint                 已用积分
```

积分是小数，兑换门槛判定不能按整数假设。

## 二、门店与地址

### 判定顺序：问渠道 + 门店（一次问完）

门店与渠道靠**问用户**确定，本 Skill **不使用系统级定位**（不调 CoreLocation、不调 PowerShell Geolocator）。顺序与形态：

```
1. 首轮提问 · 渠道与门店一起问（同一条消息）
   「这次是到店自取，还是麦乐送？顺便告诉我门店名或大致地址」
2. 用户答得不全 → 只补缺的那一项
   到店自取 → 缺门店就问门店名或大致地址
   麦乐送   → 缺地址就问送到哪个地址
3. 用户答得模糊 → 搜附近门店，摆出前几家让用户选一次
4. 用户明确不愿提供 → 档案默认值（最后手段，且必须标注未经核对）
```

用户已在本次话里明说渠道或门店时，对应的问题跳过，不重复问。

### 渠道与门店的解答分叉

| 渠道 | 门店/地址处理 | 用户给模糊地址时 |
|---|---|---|
| 到店自取 | `query-nearby-stores(searchType=2, beType=1, keyword=门店名, city=城市)` 核实或搜附近门店 | 摆出前几家让用户选 |
| 麦乐送 | `delivery-query-addresses` 取 `addressId` → `delivery-query-stores(beType=2, addressId)` | 匹配不到地址就问用户确认 |

`query-nearby-stores`：`beType` 与 `searchType` 必填。`searchType=1` 搜收藏餐厅（默认），`searchType=2` 按位置/关键词搜索（此时 `city` 与 `keyword` 必填）。`beType=1` 到店自提返回的门店无 `beCode`；`beType=5` 得来速返回 `beCode`，后续必须传。收藏餐厅为空时返回 `code=600050`，必须走 `searchType=2` 地址搜索兜底，不能当作致命错误。

`delivery-query-addresses`：无入参，返回地址列表与 `addressId`。

`delivery-query-stores`：`beType` 与 `addressId` 必填。返回门店 `storeCode`、`beCode`，以及 `reservationTimeOptions`（餐段定义：夜市、早餐、午餐、下午茶）。餐段划分以该字段为准，不要自行臆断。

### 地址带城市名

按地址搜门店时，`city` 与 `keyword` 都要给。城市名缺失时先静默补全，不追问用户：

```
档案 defaultCity → 仍无才问
```

**可以静默推断的只有城市名这一个字段**，门店与地址本身不可推断，必须来自用户。

### 搜不到门店的兜底

```
搜索结果为空        → 问用户换一个说法或换个门店
结果不唯一（同商圈多家）→ 摆出前几家让用户选一次，把选择写入档案；
                        不得自行挑最近的，选错店会导致整单价格全错
用户明确不愿提供     → 才回落到档案默认值，并标注门店未经核对
```

### 三条硬约束

```
1. 门店编号一律取自接口返回的 storeCode 与 beCode，
   不得把地址文本或门店名当作编号使用

2. 门店结果必须回显（门店名 + 地址），让用户有机会纠正。
   选错店是唯一会让整单作废的错误，值得多写一行

3. 来源必须回显。用的是用户明说的、按地址搜索选定的还是档案默认值，
   要在输出里标明。档案默认值额外标注「门店来自档案，未与你的实际位置核对」
```

## 三、菜单

`query-meals`：`storeCode`、`orderType`、`beType` 必填，非自提时另需 `beCode`。返回：

```
categories[].{name, meals[].{code, tags}}
meals.{code}.{name, image, currentPrice, originalPrice,
               discountType, canWithOrder,
               withOrder.{membershipCode, specId}}
frequent.{code, tags}         服务端返回的常点餐品
```

要点：

- `currentPrice` 与 `originalPrice` 是**元，字符串**，例如 `"34.9"`
- `frequent` 是零操作的关键：用户不开口也知道常吃什么
- 菜单里会混入棒球帽、痒痒挠等非餐饮商品，必须按类目剔除
- `canWithOrder=true` 的商品是随单购降价项；`withOrder.specId` 对应下单入参的 `membershipSpecId`（**名字不一致**）；`cardId` 与 `cardType` 在返回里没有，来源未知，缺失时跳过该维度

`query-meal-detail`：仅在用户主动要求改搭配、且该候选已进入前三时使用。工具约定明确写着不要主动展开特调选项列表。含 `unselectedKey` 的特调组，组内所有项都必须传：选中用 `selectedKey`，未选用 `unselectedKey`。

**入参名是 `code`，不是 `productCode`。** 必填 `storeCode`、`orderType`、`beType`、`code`；外送场景另需 `beCode`。传 `productCode` 会被直接拒绝。

**这也是核实「套餐里到底有什么」的唯一手段。** 用户明说了目标商品（如「我想吃巨无霸」）时，必须逐个候选调本接口，看 `rounds[].choices` 里 `isDefault=1` 的主堡是哪一款，再决定要不要推。三条理由：

```
1. query-meals 只给菜名与价格，不给套餐子项，看不出里面有什么
2. 类目名不可信。「巨无霸 牛鱼肉堡」这一栏同时收着巨无霸、不素之霸、
   培根蔬萃双层牛堡等多款，名称带「牛」的产品主堡未必是巨无霸
3. 算价返回的 productList[].productName 只能看到套餐名与打包费，
   套餐名同样可能起得有歧义（「牛气满满套餐」不含巨无霸）
```

便宜一些的替代：算价返回的 `productList` 里，套餐子项会各自单列一行，能看出构成；但确认不了主堡具体是哪款时，仍要调本接口。

## 四、券

`available-coupons`：无入参，返回**文本**，每条只有标题与状态（已领取／可领取），**没有券 ID**。领券成败只能用"可领取"条数的前后差值判定。

`query-my-coupons`：`page`、`pageSize`。返回**文本**：标题、优惠、有效期（含星期时段，例如"周二 05:00-10:29"）、领取时间、标签（到店专用／外送专用／今日到期）。同样**没有券 ID**，不能用于算价，只能做提醒与过滤。

`query-store-coupons`：`storeCode`、`orderType`、`beType` 必填，非自提另需 `beCode`。返回：

```
data[].{title, couponId, couponCode, tradeDateTime,
        products[].{productName, productCode}}
```

这是**唯一可信的可用券来源**，已被服务端按门店和渠道过滤。`products[].productCode` 让每张券适用哪些商品精确已知，不需要盲试。

返回空数组是常态：券带渠道标签和星期时段，全不匹配时就是零张。

## 五、算价

`calculate-price`：`storeCode`、`orderType`、`beType`、`items[]` 必填；非自提另需 `beCode`；可传 `needTableware`。`items[]` 每项为 `productCode`、`quantity`，券以 `couponId` 加 `couponCode` 挂在这一项上。套餐传两层：`roundList` → `comboItemList`。

返回：

```
data.productOriginalPrice    商品原价        分
data.productPrice            商品现价        分
data.deliveryOriginalPrice   运费原价        分
data.deliveryPrice           运费现价        分
data.packingOriginalPrice    打包费原价      分
data.packingPrice            打包费现价      分
data.tablewarePrice          餐具费现价      分
data.originalPrice           原价合计        分
data.discount                优惠金额        分
data.price                   实付价          分
data.productList[].{productCode, productName, quantity,
                    originalSubtotal, subtotal}
data.takeWayList[].{code, title, subtitle}      到店与得来速下单必传来源
data.mealAssistanceList[].{code, name}          团餐用
data.enjoyed.{amountType, realDiscount, balance}
data.enjoyable.{amountType, realDiscount, balance}
```

三条必须记住：

1. `data.price` **已包含商品价、配送费、打包费与餐具费**。实测商品 3490 加运费 600 加打包 190 等于 4280，与 `price` 一致。
2. `enjoyable.balance` 是门槛差额，单位为分，直接告诉还差多少能享受优惠。`amountType` 取值：1 特价、2 折扣、3 立减。`realDiscount` 单位是**元，数字**。
3. `productList` 会混入虚拟行项，例如编码 920174 的"打包服务费"。解析商品清单时必须过滤非餐品行项。

## 六、下单与订单

`create-order`：`storeCode`、`orderType`、`beType`、`items[]` 必填。条件必填：`beCode`（非自提）、`takeWayCode`（orderType 等于 1，取自 `takeWayList[].code`）、`addressId`（orderType 等于 2）、`needTableware`、`remark`（外送备注，不超过 50 字）。

是否返回支付链接**尚未验证**。无链接时输出订单号与 App 完成支付入口，不得拼造 URL。

`query-order`：按订单号查询订单状态。

## 七、积分商城

`mall-points-products`：`catRuleIds` 传 `"1>4"`。`mall-product-detail`：按 `spuId` 查详情，取 `skuId`、`spuCategory`、`shopId`。`mall-create-order`：`spuCategory` 等于 "2" 时必须传 `addressId`，逐笔串行执行。

需选门店场次的兑换（生日派对、主题派对、体验营、品鉴会、读书会）走 `party-order-create`，本 Skill 不做。

## 八、单位陷阱

| 来源 | 单位与类型 | 示例 |
|---|---|---|
| 菜单的 `currentPrice`、`originalPrice` | 元，字符串 | `"34.9"` |
| 算价的金额字段 | 分，整数 | `3490` |
| `realDiscount` | 元，数字 | `7.5` |

三套接口三种规则，混用必出金额错误。所有对外展示的金额统一换算成元并保留两位小数。
