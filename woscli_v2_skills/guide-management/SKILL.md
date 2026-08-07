---
description: '用 woscli 查询导购、导购绑定客户、业绩明细、素材和导购任务，并创建已验证的自定义图文素材。 用户提到导购名单/手机号、名下客户、导购业绩、导购线索、素材、分享数据、导购任务或客户跟进话术时使用；
  命中场景先读对应 playbook，缺少等价能力时明确边界。纯客户资料检索请用 crm-ops。

  '
name: guide-management
---


# guide-management（导购）

## Category 绑定

- 主 category：`team`
- search 默认：`woscli search "..." --category team -f json`
- 跨 category：仅当 `references/` 步骤写明（如 `customer` 反查客户、`auth` 组织节点）

## 决策协议

1. 命中下方场景索引 → Read 对应 `references/...`，按步骤执行
2. 未命中 → `woscli search "<对象 动作 近义词>" --category team -f json`
3. 参数不明或失败 → `woscli team <command> --help -f json`
4. 禁止无目标 `woscli team --help` 翻页

## 场景索引

| 意图关键词 | 文件 |
| --- | --- |
| 查导购 / 导购手机号 / 启用导购 / 名下客户 / 绑定客户 | `references/guider-and-customer-lookup.md` |
| 导购业绩 / 业绩明细 / 导购销售 / 按导购查订单 | `references/guide-performance-query.md` |
| 导购素材 / 内容素材 / 创建图文素材 / 分享收藏数据 | `references/guide-material-ops.md` |
| 导购任务 / 任务模板 / 执行进度 / 任务客户 | `references/guide-task-query.md` |

### 当前已知边界

- 当前 Catalog 没有老版“导购线索列表”的等价筛选能力，不能按未查看/未跟进/已跟进/已成交状态查询。`guider-customer-list` 只表示绑定关系，不是线索。
- 老版 `agent-guide sceneRecommendContent` 的商品/活动/优惠券推荐话术不属于当前 `team` category。可以查询客户、素材和任务作为事实输入，但不能声称有等价的一键推荐命令。
- `guider-customer-list` 在最新报告中有 passed 与 failed Case 混合；当前只读实跑成功，但不要扩展成加好友来源或新增好友统计。

## 横切约定

- 需要稳定解析时使用 `--output-format json`（或 `-f json`）
- 分页默认 `page=1`、`page-size=20`，除非场景另有说明
- 写操作：只读定位 → 变更 → 只读复核；破坏性操作需用户确认
- 导购、客户、素材和任务标识必须从当前上下文的只读结果获得，不复制 eval/示例值
- 手机号属于个人信息：仅用于定位并默认脱敏展示；稳定关联优先使用 `guiderWid`
- 场景未覆盖的能力走 search，不要猜 command 名
- 会员/客户主检索归 `crm-ops`；店铺组织主路径归 `merchant-store`
