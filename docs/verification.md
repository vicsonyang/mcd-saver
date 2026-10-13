# 真实调用验证记录

本文记录本项目在 2026-10-13 完成的一次完整**只读**链路验证。

- 全部调用均为只读，**不领券、不下单、不消耗积分、不修改任何账号状态**。
- 门店为示例门店（上海人民广场附近），非特定个人地址。
- 涉及账号的返回值只记录**字段形状**，不记录具体数值；券码、券 ID 一律脱敏。

---

## 一、调用汇总

| # | Tool | 入参（脱敏） | 返回摘要 |
|---|---|---|---|
| 1 | `now-time-info` | 无 | 服务器时间 `2026-10-13 16:34:47`，时区 `GMT+08:00`，`dayOfWeek=TUESDAY` |
| 2 | `query-nearby-stores` | `beType=1, searchType=2, city=上海, keyword=人民广场` | 5 家门店，首店 `storeCode=1450713`，`reservation=true` 且带 `reservationTimeOptions` |
| 3 | `query-meals` | `storeCode=1450713, orderType=1, beType=1` | 14 个分类、100+ 餐品；返回 `currentPrice` / `originalPrice` / `discountType` / `canWithOrder` |
| 4 | `query-store-coupons` | `storeCode=1450713, orderType=1, beType=1` | 该门店此刻可用券 3 张（含渠道与时段过滤后的结果） |
| 5 | `calculate-price` | 候选 A：`9900005466` ×1 | 实付 `3650` 分，`takeWayList` = 堂食 / 外带 |
| 6 | `calculate-price` | 候选 B：`1100` ×1 + `4810` ×1 + `9900008751` ×1 | 实付 `4850` 分，`productList` 三行明细齐全 |
| 7 | `calculate-price` | 候选 C：`9900005453` ×1 | 实付 `2800` 分 |
| 8 | `calculate-price` | 券挂载：`9900012387` ×1 + `couponId` / `couponCode` | 原价 `2700` 分 → 实付 `1350` 分，`discount=1350` |

---

## 二、关键验证点

### 2.1 单位陷阱确认（最重要）

同一次验证中拿到了三套不同的单位规则，证实了 [`references/field-manual.md`](../references/field-manual.md) 的结论：

| 来源 | 实际返回值 | 单位 |
|---|---|---|
| 菜单 `currentPrice` | `"36.5"` | **元，字符串** |
| 算价 `price` | `3650` | **分，整数** |
| 券抵扣 `discount` | `1350` | **分，整数** |

混用会导致**金额放大 100 倍**或缩小 100 倍。本项目在展示层统一换算为元并保留两位小数。

### 2.2 `data.price` 的口径

候选 A 返回：

```
productPrice  = 3650
deliveryPrice = 0
packingPrice  = 0
tablewarePrice = 0
price         = 3650
```

到店自取场景无运费与打包费，`price` 等于商品价——与字段手册中「`price` 已包含商品价、运费、打包费与餐具费」的结论一致。因此**最小化 `price` 就是四项费用的联合最优**，无需额外加权。

### 2.3 券挂载生效

候选 D（券挂载）返回：

```
productOriginalPrice = 2700
productPrice         = 1350
discount             = 1350
price                = 1350
productList[0] = { productName: "中薯条买一送一", quantity: 1,
                   originalSubtotal: 2700, subtotal: 1350 }
```

`originalSubtotal = 2700` 对应 2 份中薯条（买一送一），`subtotal = 1350` 为实际支付。
说明**券必须通过 `couponId` + `couponCode` 挂在对应 `items[]` 项上**，服务端才会计算抵扣；券不能整包传入，也不能挂在无关商品上。

### 2.4 `productList` 的构成可信度

候选 B 返回三行明细，与请求的三个商品逐项对应：

```
巨无霸      ×1  originalSubtotal 2550  subtotal 2550
中薯条      ×1  originalSubtotal 1350  subtotal 1350
可乐        ×1  originalSubtotal  950  subtotal  950
```

数量与构成均与请求一致——这正是本项目第 6 步「算价返回必须与请求逐项核对」可以落地的基础。

### 2.5 门店搜索返回预约餐段

`query-nearby-stores` 返回的门店带 `reservationTimeOptions`，给出各日期的餐段划分（夜市 / 宵夜 / 早餐 / 午餐 / 下午茶）及具体时间窗口。本项目**以该字段为准，不自行臆断餐段划分**。

---

## 三、候选对比结果

门店：`storeCode=1450713`（上海黄浦华旭国际大厦餐厅）· 到店自取 · 一人份 · 目标商品：巨无霸

| 候选 | 构成 | 原价 | 实付 |
|---|---|---:|---:|
| **A 巨无霸三件套** | `9900005466` ×1 | ¥36.50 | **¥36.50** |
| B 单点自己拼 | `1100` + `4810` + `9900008751` | ¥48.50 | ¥48.50 |
| C 麦香鸡中套餐 | `9900005453` ×1 | ¥28.00 | ¥28.00 |
| D 券挂载验证 | `9900012387` ×1 + 券 | ¥27.00 | **¥13.50** |

**结论**：在「想吃巨无霸」这个约束下，A 比 B 省 ¥12.00；C 虽然更便宜但未命中目标商品（属于「同类目不同款不算命中」），只能作为备选。

---

## 四、未执行的验证（如实标注）

以下能力**尚未在真实下单中验证**，本项目在文档与输出中均标注为「部分验证」或「未验证」，不宣称已验证：

| 能力 | 状态 | 说明 |
|---|---|---|
| `create-order` 是否返回支付链接 | 未验证 | 验证截止时未创建真实订单；无链接时输出订单号与 App 入口，**不拼造 URL** |
| `create-order` 的幂等行为 | 未验证 | 策略为「失败后先查 `order-list` 再决定是否重试」，不做裸重试 |
| 麦乐送场景的运费与打包费口径 | 部分验证 | 字段手册已记录结构，本次未跑真实外送算价 |
| 得来速（`beType=5`）场景 | 未验证 | `beCode` 传递规则来自官方工具说明 |
| 积分兑换路径 | 未验证 | 本次未消耗积分 |

---

## 五、复现方法

```bash
# 1. 配置 mcd-mcp 连接器（见 README 第三节）
# 2. 发起一次只读探测，并写入脱敏结果文件
#    内容模板见 references/setup-guide.md 第四节
# 3. 运行环境检查
python scripts/check_environment.py --capability 最优组合试算 --probe-results ./probe-results.json
```

复现时请勿在对话或文件中留下真实 Token。
