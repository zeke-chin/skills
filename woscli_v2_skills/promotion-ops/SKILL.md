---
description: '用户要求创建、预览或配置营销活动，询问营销活动创建能力，或提到限时折扣、满减满折、 满减邮、满赠、N 元 N 件、一口价、拼团、订单换购、秒杀等促销活动时使用；创建后需要
  activityId、belongVidType、promotionStatus、belongVid 等活动标识与状态时也必须使用。

  '
name: promotion-ops
---


# promotion-ops（营销 · 活动创建）

## Category 绑定

- 主 category：`marketing`
- search 默认：`woscli search '...' --category marketing -f json`
- 跨 category：仅当 `references/` 步骤写明（例如创建前用 `goods` 定位/校验商品）；不把商品运营收进本 skill

映射总表见 `distill_skills/docs/business-skill-category-map.md`。

## 决策协议

1. 命中下方场景索引 → Read 对应 `references/...`，按步骤执行
2. 未命中 → `woscli search '<对象 动作 近义词>' --category marketing -f json`
3. 参数不明或失败 → `woscli marketing <command> --help -f json`
4. 禁止无目标 `woscli marketing --help` 翻页

## 场景索引

| 意图关键词 | 文件 |
| --- | --- |
| 限时折扣 / 单品打折 / 商品八折 / 创建折扣活动 | `references/limited-time-discount-create.md` |
| 满减满折 / 满 100 减 10 / 满额立减 | `references/full-discount-create.md` |
| 满减邮 / 满额包邮 / 满 99 包邮 | `references/free-freight-create.md` |
| 满赠 / 买满送 / 满额赠品 | `references/full-gift-create.md` |
| N 元 N 件 / 任选两件 20 元 / 多件一口价 | `references/n-for-n-create.md` |
| 一口价 / 固定价 | `references/fixed-price-create-boundary.md` |
| 拼团 / 多人成团 | `references/group-buy-create-boundary.md` |
| 订单换购 / 订单加价购 | `references/order-redemption-create-boundary.md` |
| 秒杀 / 整点抢购 | `references/seckill-create-boundary.md` |
| 支持哪些活动 / 活动能力总览 | `references/activity-type-capability-boundaries.md` |
| 默认值 / 参数怎么填 / 用户没给参数 / 自动补参数 | `references/activity-form-defaults.md` |
| 创建后结果 / activityId / belongVidType / promotionStatus / belongVid | `references/post-create-activity-result.md` |

当前创建 references 只覆盖各文件写明的固定时间、全部组织、全部人群、线上订单和单层保守规则。满件、满折、循环优惠、周期活动、部分人群等未覆盖变体，按决策协议走 `search`，不要套用相近参数。

旧版 skill 的 9 种前端表单不等于当前 `marketing` 的 9 条创建命令。具体映射和实测状态见能力边界 reference；没有对应 create command 时不要用通用 `activity-create` 冒充。

## 横切约定

- 需要稳定解析时使用 `--output-format json`（或 `-f json`）
- 分页默认 `page=1`、`page-size=20`，除非场景另有说明
- URL、`image_ref://...`、JSON 和其它字面量必须作为单个 shell 参数传入；URL / image_ref 优先使用 Bash 单引号，禁止裸拼进命令
- 创建参数优先从用户描述、BasicInfo、商品详情和 reference 默认值补齐；不逐字段追问，也不让用户提供内部 ID
- 已锁定准确 command 但用户没有给可选业务值时，读取 `references/activity-form-defaults.md`；当前 command help、当前环境实跑和场景 reference 的取值优先于旧表单默认
- 用户明确说“创建 / 新建 / 帮我创建”即已授权本次创建：**只读定位 → 正式创建一次 → 检查创建响应 → 按 `references/post-create-activity-result.md` 回读四字段 → 向用户返回**，不要再追加摘要确认
- 用户明确说“预览 / 草稿 / 先别提交”时只执行只读定位和参数补全，返回摘要后停止，不调用创建 command
- 只有活动类型无法判断，或自动选品后仍没有可用商品/SKU、身份上下文缺失等无法安全补齐的核心条件，才合并成一个问题
- 当前营销创建 command 不暴露 `--wid`；操作人 `wid` 由 Gateway 从会话身份注入。不要让用户提供 `wid`，也不要把它手工加入创建参数
- 创建 command 只正式调用一次。命令成功结束且响应没有顶层 `error`、`success=false`、非空失败明细等明确业务失败时，立即视为创建成功；`activityId` 是可选结果，返回 `{}` 或没有活动 ID 仍立刻报告成功
- 创建成功且响应含 `data.activityId` 时，按 `references/post-create-activity-result.md` 调用一次 `activity-detail`，返回 `activityId`、`belongVidType`、`promotionStatus`、`belongVid`；不得触发第二次创建
- 创建成功但没有 `activityId` 时，说明当前接口无法继续安全取得四字段；禁止猜 ID、按标题模糊定位或重复创建
- 除统一 `activity-detail` 外，不为补四字段调用 goods-detail、list、`activity-pair-list` 或 `activity-goods-recommend-list`；也不 `sleep` 等待活动开始
- HTTP 5xx、工具超时、连接中断或无法取得创建响应不属于“成功但无 ID”；此时写入状态不确定，直接报告卡点，禁止换 UUID 或重复创建
- JSON 对象/数组参数必须作为完整 JSON 字符串传入；单元素数组也必须保留外层 `[...]`
- 场景未覆盖的能力走 search，不要猜 command 名
- 商品列表/改价/库存归 `product-ops`；本 skill 只在 playbook 写明时临时调用 `goods`
- 跨到 `goods` 选品时严格使用当前活动 reference 写出的完整命令；`shop-goods-list` 的当前商户上下文由 Gateway 注入，只用 `goods-status=0` 表示上架，禁止追加该命令未暴露的 `--store-id`、`--basic-info-vid-type` 或其它营销命令参数
