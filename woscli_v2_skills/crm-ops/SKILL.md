---
description: '用 woscli 处理 CRM 常见只读运营：按手机号、姓名或昵称定位客户，查询客户资料、 会员卡与权益、积分、储值余额及流水，并补充已知客户的消费统计。
  用户提到查客户、会员、会员卡、积分、余额、储值、客户消费或 customer 域 SOP 时使用； 导购关系和导购业绩使用 guide-management。

  '
name: crm-ops
---

# crm-ops（CRM）

## Category 绑定

- 主 category：`customer`
- search 默认：`woscli search "<对象 动作 近义词>" --category customer -f json`
- 跨 category：仅使用场景 reference 明示的 `auth` 组织定位和 `cdp` 消费/标签查询

## 决策协议

1. 命中场景索引 → Read 对应 `references/...`，按步骤执行
2. 未命中 → `woscli search "<对象 动作 近义词>" --category customer -f json`
3. 已锁定 command 但参数不明或失败 → `woscli customer <command> --help -f json`
4. 禁止无目标 `woscli customer --help` 翻页，也不要对 playbook 已写明的参数重复 help

## 场景索引

| 意图关键词 | 文件 |
| --- | --- |
| 手机号换 wid / 姓名或昵称搜客户 / 客户资料 / 会员卡 / 会员方案 / 会员权益 | `references/customer-and-membership-lookup.md` |
| 当前积分 / 可用积分 / 积分流水 / 积分变动 | `references/point-query.md` |
| 当前余额 / 本金 / 赠金 / 储值流水 / 余额变动 | `references/stored-value-query.md` |
| 已知客户消费统计 / 消费次数或金额 / CDP 标签字典 | `references/customer-consumption-and-tags.md` |

## 横切约定

- 稳定解析使用 `-f json`；分页默认 `page=1`、`page-size=20`
- 精确客户查询以当前上下文解析出的 `wid` 为主；`user-search` 不支持按手机号或 `wid` 过滤
- 客户手机号、证件号、生日、地址等属于个人信息；默认只展示完成任务所需字段并脱敏
- 本 skill 仅沉淀已验证的只读路径；冻结、导入、积分/余额调整等写操作不在这些 playbook 内
- 企业微信好友状态、跟进策略和导购绑定关系不等于 CRM 基础资料；相关需求定向 search，不能从字段缺失推断状态
