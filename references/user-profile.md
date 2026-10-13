# 用户档案

档案记录长期偏好与授权状态。字段缺失即按缺省值处理，不向用户追问。

## 一、档案结构

```
autoBindCoupons      领券授权状态：未询问过为空，已授权为 true，已拒绝为 false
locationAuthorized   位置授权状态：本 Skill 不再使用技术定位，该字段已废弃，保留以免旧档案解析出错
defaultCity          默认城市（补全搜索关键词的城市名；位置取不到且用户不愿说时的兜底）
defaultAddressId     默认地址编号（用户明确不愿提供地址或门店名时才用）
defaultStoreCode     默认门店编号，同上
defaultBeCode        与默认门店配对的业务编码
defaultChannel       默认渠道：自取、得来速、麦乐送
defaultStoreConfirmedAt  档案默认门店最后一次被确认的时间，用于时效判断
budget               默认预算上限，为空表示不设上限
exclude              忌口清单
trustMode            信任模式开关，默认关闭
trustModeLimit       信任模式的实付上限
trustModeStores      信任模式的门店白名单
lastOrder            上次成功下单的场景记录，供下次沿用
```

### lastOrder · 上次下单记录

```
lastOrder.channel        渠道：到店自取 / 得来速 / 麦乐送
lastOrder.storeCode      门店编号
lastOrder.beCode         业务编码（到店自取为空）
lastOrder.storeName      门店名，回显用
lastOrder.addressId      配送地址编号（仅麦乐送）
lastOrder.addressText    配送地址文本，回显用（仅麦乐送）
lastOrder.orderedAt      下单时间，用于时效判断
```

**写入时机**：每次 `create-order` 成功创建订单后写入，覆盖旧值。

**只有建单成功才写。** 推荐了方案但用户没下单、或建单失败时，一律不写——
`lastOrder` 是「上次下单记录」，不是「上次问过的门店」。没下单就预写，会让下次流程误判为
「有历史可沿用」，从而跳过两次提问，把用户带到他从没选过的门店。

**读取时机**：第 2 步定场景时。`lastOrder` 非空且 `orderedAt` 距今不超过三十天，就直接沿用它，跳过两次提问。

**沿用规则**：

```
1. 本次明说的渠道或门店/地址，覆盖 lastOrder 对应的那一项
2. 沿用时必须在门店行回显「按上次下单的门店与渠道」，并附可改的提示
3. orderedAt 超过三十天 → 不静默沿用，把历史值作为第二问的默认选项摆出来确认一次
```

`defaultChannel` 只在用户本次未明说、且第一问也没得到回答时兜底。渠道的**第一顺位是用户本次的回答**，第二顺位是 `lastOrder.channel`，第三位才是 `defaultChannel`。

门店字段的定位是**最后手段**，不是首选。每次进流程先按「明说 → 沿用历史 → 问渠道与门店/地址」走，
只有用户明确表示不愿提供地址或门店名时，才读这里的默认门店。

**人数不入档案。** 用户没说人数时一律按一人处理，不记录、不沿用历史值。
档案里的历史人数与用户此刻几个人吃无关，按它推断等于拿旧数据冒充当前需求。
详见 pricing-playbook.md 第三节。

默认门店带时效：`defaultStoreConfirmedAt` 距今超过三十天时，不再直接沿用，
降级为「需要在输出里标注门店未经核对」，并建议用户下次提供一次地址。
超出时效不影响流程继续，只影响可信度标注。

## 二、券叠加结论缓存

以"券 A 加券 B"为键记录实测结论：

```
key          两张券的编号组合
result       可叠加、互斥、部分生效三者之一
checkedAt    实测时间
```

遇到同一对券直接用结论，不重复试探。条目超过三十天视为失效，需重新实测。

## 三、填写规则

- 用户明说的信息当次生效，并询问是否需要长期记住
- 只在首次涉及某项时问一次，答案写入档案，此后不再询问
- 用户说"别再问了"视为永久跳过该环节
- 不做任何跨店偷偷换店的行为，换门店必须先问
- 档案不记录任何凭据、令牌或密码

## 四、当前档案值

> ⚠️ **本仓库为公开仓库，此处保持空白模板，不含任何真实档案值。**
> 实际使用时，档案由 Skill 在你的本机首次运行时建立并写入本地副本，
> 请勿把填写了真实地址、门店或账号信息的档案提交到任何公开仓库。

```
autoBindCoupons   空            尚未询问领券授权
defaultCity       空
defaultAddressId  空
defaultStoreCode  空
defaultBeCode     空
defaultStoreConfirmedAt  空      尚未经用户主动确认，仅作最后手段
defaultChannel    空
budget            空           未设上限，不做预算硬过滤
exclude           空           无已知忌口
trustMode         false         默认关闭
lastOrder         空           尚无成功下单记录
```

`defaultStoreCode` 只在用户明确表示不愿提供地址或门店名时启用。用户答了渠道与门店/地址时一律以用户答案为准，不读这两个字段。

叠加结论缓存统一记录在 coupon-rules.md 第七节，不在此重复维护。
