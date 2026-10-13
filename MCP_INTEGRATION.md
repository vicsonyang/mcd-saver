# 麦当劳 MCP 集成说明

本项目**真实调用**麦当劳中国 MCP 服务完成点餐省钱优化。本文说明实际使用的 MCP Server、Tool、调用流程与业务价值。

---

## 一、接入信息

| 项 | 值 |
|---|---|
| MCP Server | `mcd-mcp`（麦当劳中国官方 MCP Server） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | Streamable HTTP |
| 认证方式 | 请求头 `Authorization: Bearer <MCD_MCP_TOKEN>` |
| 限流 | 每个 Token 每分钟最多 600 次请求，超出返回 429 |
| 开放平台 | https://open.mcd.cn/mcp |
| 使用指南 | https://github.com/M-China/mcd-mcp-server |

**认证安全边界**：Token 由使用者自行申请并配置在 WorkBuddy 的连接器中，**本项目的仓库、代码与文档中不含任何真实 Token**，[`mcp-config.example.json`](mcp-config.example.json) 仅使用占位符。麦当劳会员授权由连接器侧承载，本 Skill 不接触密码、令牌或 Cookie。

**错误码处理**：

| code | 含义 | 本项目的行为 |
|---|---|---|
| 401 | Token 无效 / 过期 / 未提供 | 判定为 `unavailable`，展示恢复步骤，不降级为猜测数据 |
| 429 | 触发限流（>600 次/分钟） | 本 Skill 通过「先剪枝再算价」把调用量压到最低，避免触发；触发时退避等待 |

---

## 二、实际使用的 Tool 清单

共 **16 个**工具，按在流程中的角色分组。

### 2.1 时间基准

| Tool | 用途 | 入参要点 |
|---|---|---|
| `now-time-info` | 取服务器时间与星期。**星期是券过滤的必需输入**，不是可有可无的基础信息 | 无入参 |

### 2.2 门店与地址

| Tool | 用途 | 入参要点 |
|---|---|---|
| `query-nearby-stores` | 到店场景搜索门店 | `beType`、`searchType` 必填；`searchType=2` 时 `city`、`keyword` 必填。`beType=1` 到店自提、`beType=5` 得来速（返回 `beCode`，后续必传） |
| `delivery-query-addresses` | 取用户可配送地址列表 | 无入参，返回 `addressId` |
| `delivery-query-stores` | 按地址查可配送门店 | `beType`、`addressId` 必填；返回 `storeCode`、`beCode` 与 `reservationTimeOptions`（餐段定义） |

### 2.3 菜单

| Tool | 用途 | 入参要点 |
|---|---|---|
| `query-meals` | 取门店菜单与**服务端返回的常点清单** | `storeCode`、`orderType`、`beType` 必填；非自提另需 `beCode` |
| `query-meal-detail` | 核实「套餐里到底有什么」 | 入参名是 **`code`**（不是 `productCode`）；仅在用户要求改搭配且候选已进入前三时调用 |

### 2.4 优惠券

| Tool | 用途 | 入参要点 |
|---|---|---|
| `auto-bind-coupons` | 一键领取麦麦省当前可领的券 | 无入参 |
| `available-coupons` | 查看可领取券列表。**在领券前后各调一次**，用「可领取」条数差值判定本次新领到几张 | 无入参 |
| `query-store-coupons` | **唯一可信的可用券来源**：服务端已按门店 + 渠道过滤，并给出每张券适用的 `productCode` | `storeCode`、`orderType`、`beType` 必填 |
| `query-my-coupons` | 卡包券文本（标题、有效期含星期时段、标签）。**不能用于算价**，只做提醒与过滤 | `page`、`pageSize` |

### 2.5 算价与下单

| Tool | 用途 | 入参要点 |
|---|---|---|
| `calculate-price` | **核心工具**：服务端算价。返回四项费用拆解与实付价 | `storeCode`、`orderType`、`beType`、`items[]` 必填；券以 `couponId` + `couponCode` 挂在对应 `items[]` 项上；套餐传 `roundList` → `comboItemList` |
| `create-order` | 创建订单，返回订单号与支付入口 | 条件必填：`beCode`（非自提）、`takeWayCode`（orderType=1，取自算价返回的 `takeWayList[].code`）、`addressId`（orderType=2） |
| `order-list` | 查询历史订单。**建单异常时先查它**，判断是否已生成，避免重复下单 | 无必填入参 |
| `query-order` | 按订单号查订单状态 | — |
| `cancel-order` | 取消订单。发现重复的未支付订单时用于清理 | — |

### 2.6 账号资产

| Tool | 用途 | 入参要点 |
|---|---|---|
| `query-my-account` | 积分账户：可用积分、**本月/下月将过期积分**等 | 无入参。注意积分字段是**小数**，不是整数 |

---

## 三、调用流程

九步流程，其中**第 3 步的五个调用必须并发发起**——它们之间没有数据依赖，串行会白白多两轮往返。

```
第 1 步  取服务器时间
         └─ now-time-info                                  【必调，券过滤的前置】

第 2 步  定场景（渠道 → 门店/地址）
         ├─ 到店：query-nearby-stores
         └─ 麦乐送：delivery-query-addresses → delivery-query-stores
         └─► 输出 storeCode / beCode

第 3 步  领券与拉资源（一次并发，五项同时发起）
         ├─ auto-bind-coupons
         ├─ query-meals
         ├─ query-store-coupons
         ├─ query-my-account
         └─ query-my-coupons
         └─ available-coupons 前后各一次，用条数差值判定领券结果

第 4 步  券可用性过滤
         四道过滤：渠道标签 → 适用星期 → 时段窗口 → 是否过期
         过滤后为零张是常态，走「无券价最优」分支

第 5 步  生成候选（最多呈现三个）
         参数：人数、预算、口味偏好、忌口 —— 一律不强制提供
         缺参时的降级来源：用户明说 → 服务端常点清单 → 本地档案

第 6 步  服务端算价
         先剪枝到最终呈现数量，再算价（这是全流程最耗时的一步）
         互不依赖的算价并发发起
         返回结果与请求逐项核对，异常候选作废

第 7 步  呈现
         券汇报行 → 门店行（含来源回显）→ 候选正文 → 搜索过程行

第 8 步  确认
         明确肯定才算确认；方案十分钟内有效

第 9 步  复价与建单
         完全相同的参数再算一次 → 价格一致才 create-order
         建单异常：先 order-list 查是否已生成，再用 cancel-order 清理重复单
```

### 3.1 为什么这样编排

| 设计 | 原因 |
|---|---|
| 第 3 步五项并发 | 五项无数据依赖，`query-meals` 需要的 `storeCode` 在第 2 步已拿到 |
| 先剪枝再算价 | `calculate-price` 是全流程最耗时的一步，把候选池全算一遍会直接变成用户的等待时间 |
| 建单前必复价 | 从看到方案到确认之间价格可能已变，默默用新值下单等于欺骗 |
| 建单异常先查 `order-list` | 写接口裸重试会生成重复订单；查一次比赔一单成本低 |
| 券按商品编码挂载 | 券只能挂在单个商品上，整包传入会被拒绝 |

---

## 四、三个必须记住的接口口径

这是本 Skill 最容易出错的地方，也是 MCP_INTEGRATION 里最值得写清的细节：

| 来源 | 单位与类型 | 示例 |
|---|---|---|
| 菜单的 `currentPrice` / `originalPrice` | **元，字符串** | `"36.5"` |
| 算价的全部金额字段（`price`、`discount`、`productPrice`…） | **分，整数** | `3650` |
| `enjoyed` / `enjoyable` 的 `realDiscount` | **元，数字** | `7.5` |

三套接口三种规则，混用必出金额错误。本项目所有对外展示的金额统一换算成元并保留两位小数。

另外两条实测结论：

1. **`data.price` 已包含商品价、运费、打包费与餐具费**。实测：商品 3490 + 运费 600 + 打包 190 = 4280，与 `price` 一致。因此「最小化实付价」天然就是四者的联合最优，不需要额外加权。
2. **`productList` 会混入虚拟行项**（例如编码 `920174` 的「打包服务费」），解析商品清单时必须过滤非餐品行项。

以及一条接口行为提示：`query-nearby-stores(searchType=1)` 搜收藏餐厅为空时返回 `code=600050`，这是**正常状态而非错误**，必须回落到 `searchType=2` 地址搜索。

---

## 五、业务价值

### 5.1 直接价值：把「凭感觉点」换成「算过再点」

以 [`docs/verification.md`](docs/verification.md) 记录的真实一次调用为例（到店自取，一人份，目标商品为巨无霸）：

| 点法 | 实付 | 差额 |
|---|---:|---:|
| 巨无霸 + 中薯条 + 可乐（单点自己拼） | ¥48.50 | — |
| 巨无霸三件套 | **¥36.50** | **省 ¥12.00** |
| 中薯条买一送一券（挂券后） | ¥13.50（原 ¥27.00） | **省 ¥13.50** |

一次点单，人脑需要比较的组合在十几个到几十个之间；本项目把它们交给服务端算价，用**真实价格**排序。这带来三个可验证的收益：

1. **省掉「人肉枚举」**：不用在套餐、单点、券、积分之间反复试算。
2. **避免「看似省钱其实更贵」**：加购型券（`canWithOrder=true`）挂上去必须额外买卡，本项目会把「相对裸单的实付增量」明确写出来，净收益 ≤ 0 就不推荐。
3. **不漏掉「本来就能领的券」**：`auto-bind-coupons` 静默补齐可领券，领到几张报几张，领到零张时**不会宣称「已为你自动领券」**。

### 5.2 对麦当劳 MCP 的能力验证

本项目对 MCP 的使用深度超出了「一问一答」：

- **组合使用 16 个工具**，横跨时间、门店、地址、菜单、券、算价、下单、订单、账号九类能力
- **把 `calculate-price` 当作求解器而非计算器**：同一候选做多次不同券挂载方式的试算，再由本地排序选出最优
- **正反向用接口**：`available-coupons` 用前后差值反推领券结果；`order-list` 用查询来避免写接口的重复副作用
- **接口口径建模**：把「元/分/元」三套单位规则写成字段手册，避免金额错误

---

## 六、验证记录

2026-10-13 完成的只读链路实测（不含任何写操作，不消耗积分、不下单）：

| # | Tool | 结果 |
|---|---|---|
| 1 | `now-time-info` | ✅ 返回服务器时间 2026-10-13 16:34:47 GMT+8 |
| 2 | `query-nearby-stores` | ✅ 返回 5 家门店（含 `storeCode`、预约餐段） |
| 3 | `query-meals` | ✅ 返回 14 个分类、100+ 餐品（含 `currentPrice` / `originalPrice`） |
| 4 | `query-store-coupons` | ✅ 返回该门店此刻可用券 3 张 |
| 5 | `calculate-price` ×3 | ✅ 返回四项费用拆解与实付价，用于候选对比 |

明细见 [`docs/verification.md`](docs/verification.md)。

---

## 七、未使用的能力

以下工具本项目**不使用**，此处如实列出以说明能力边界：

`list-nutrition-foods`（不做营养建议）、`query-meal-assistance`（不做企业团餐）、`query-party-city` / `query-party-store` / `query-partystore-date` / `query-partystore-session` / `party-order-create`（不做派对预约）、`campaign-calendar`（不做活动日历）、`mall-points-products` / `mall-product-detail` / `mall-create-order` / `mall-order-list` / `mall-order-detail`（不做商城兑换）、`query-lottery-info` / `draw-lottery` / `query-my-prizes`（不做抽奖）、`delivery-create-address`（不代用户建地址）。
