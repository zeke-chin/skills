# 营销活动创建能力总览

## 当前能力表

| 用户活动类型 | 当前专用能力 | 创建响应收口 | 对应文件 |
| --- | --- | --- | --- |
| 限时折扣 | `discount-create` | 通常返回 ID；用 `activityType=3` 回读四字段 | `limited-time-discount-create.md` |
| 满减满折 | `fulldiscount-create` | 返回 ID；用 `activityType=1` 回读四字段 | `full-discount-create.md` |
| 满减邮 | `freefreight-create` | 通常返回 ID；用 `activityType=2` 回读四字段 | `free-freight-create.md` |
| 满赠 | `gift-create` | 通常返回 ID；用 `activityType=32` 回读四字段 | `full-gift-create.md` |
| 一口价 | `oneprice-detail`、`oneprice-goods-detail` | 只有查询，没有 `oneprice-create` | `fixed-price-create-boundary.md` |
| 拼团 | 无拼团专用 create | 通用 `activity-create` 不能表达拼团规则 | `group-buy-create-boundary.md` |
| N 元 N 件 | `nynj-create` | 通常返回 ID；用 `activityType=12` 回读四字段 | `n-for-n-create.md` |
| 订单换购 | `redemption-order-detail` | `redemption-create` 是单品换购，没有订单换购 create | `order-redemption-create-boundary.md` |
| 秒杀 | 无秒杀专用 create | 通用活动或限时折扣都不能替代秒杀 | `seckill-create-boundary.md` |

表中的创建模板只适用于各 reference 写明的固定时间、保守单层规则。满件、打折、循环优惠、多层阶梯、部分人群、部分门店等变体不自动沿用。

## 标准路径

1. 精确识别活动类型。`单品换购` 不等于 `订单换购`，`限时折扣` 不等于 `秒杀`。
2. 命中表格中的文件后直接读取并执行，不再做无目标 search。
3. 只有当前类型处于创建缺口时，按对应 boundary 文件做一次精确 search，确认 Catalog 是否更新。
4. 有专用创建 SOP 时，使用该文件内联的完整选品命令执行：只读选品 → 正式创建一次 → 检查创建响应。不要从营销创建参数反推商品列表参数。
5. 创建成功且响应含 `data.activityId` 时，按 `post-create-activity-result.md` 调用一次统一 `activity-detail`，回读 `activityId`、`belongVidType`、`promotionStatus`、`belongVid`。
6. 返回 `{}` 或没有 ID 时仍按创建成功处理，但明确四字段不可安全取得；不等待、不猜 ID，也不重复创建。

## 已验证参数模式

- 可创建类型默认使用全部组织、全部人群、线上订单、固定时间和单层保守优惠。
- 可创建类型统一用 `goods-status=0` 查询上架商品；`goods shop-goods-list` 不接受 `store-id`、`basic-info-vid-type`，当前商户上下文由 Gateway 注入。组织参数只按各 `marketing` command 的 help/reference 传入。
- JSON 对象和数组必须作为一个完整参数传入，并保留正确外壳。
- command 需要图片时，用户明确给出的 URL 优先；否则直接使用 `activity-form-defaults.md` 中的固定 HTTPS URL。
- 用户明确说“创建”即授权正式写入；用户说“预览/草稿”时只补齐参数并返回摘要，不调用创建 command。

## 创建响应与默认后续动作

| 活动类型 | 正式创建响应 | 默认后续动作 |
| --- | --- | --- |
| 限时折扣 | 通常含 `data.activityId` | 用 `activityType=3` 回读并返回四字段 |
| 满减满折 | 通常含 `data.activityId`、`data.success` | `success=false` 才失败；成功时用 `activityType=1` 回读并返回四字段 |
| 满减邮 | 通常含 `data.activityId` | 用 `activityType=2` 回读并返回四字段 |
| 满赠 | 通常含 `data.activityId` | 用 `activityType=32` 回读并返回四字段 |
| N 元 N 件 | 通常含 `data.activityId`、`data.success` | `success=false` 才失败；成功时用 `activityType=12` 回读并返回四字段 |

## 失败时

- search 只返回相近活动：报告没有准确 create command，不继续拼装。
- 正式写入成功且没有业务失败明细：返回 ID 时按统一路径回读四字段，未返回 ID 也视为创建成功。
- 正式写入返回 `{}` 或没有 ID：禁止等待、标题模糊定位、候选关联和重试；明确四字段不可安全取得。

## 不要

- 不要调用无目标 `woscli marketing --help` 翻整页。
- 不要用通用 `activity-create` 代替具体促销创建。
- 不要把旧版前端表单字段当成当前 woscli 参数。
- 不要因为默认值可补齐，就把缺失专用 create 的活动说成可以创建。
- 不要在没有准确专用 create command 的活动中提前调用商品列表或让用户提供商品内部 ID。
