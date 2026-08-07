---
description: '用 woscli 查询店铺组织、门店详情、员工和门店业绩。 用户提到查门店/组织、门店地址或营业时间、员工、店铺经营概览、门店业绩， 或询问商户列表、店铺待办是否可查时使用；命中已验证场景就先读对应
  playbook， 其它能力定向 search 并说明边界。不做商品改价、发货或会员检索。

  '
name: merchant-store
---


# merchant-store（店铺）

## Category 绑定

- 主 category：`auth`
- search 默认：`woscli search "..." --category auth -f json`
- 跨 category：仅当 `references/` 步骤写明（例如 `team` 门店业绩）；不扩大为本 skill 默认职责

## 决策协议

1. 命中下方场景索引 → Read 对应 `references/...`，按步骤执行
2. 未命中 → `woscli search "<对象 动作 近义词>" --category auth -f json`
3. 参数不明或失败 → `woscli auth <command> --help -f json`
4. 禁止无目标 `woscli auth --help` 翻页

## 场景索引

| 意图关键词 | 文件 |
| --- | --- |
| 查门店 / 组织列表 / 门店详情 / 地址 / 营业时间 / 查员工 | `references/org-and-employee-lookup.md` |
| 门店业绩 / 门店 GMV / 店铺经营数据 / 门店排名 | `references/store-performance-query.md` |

当前 `auth` 严格通过能力不含“可访问商户列表”和“店铺待办”。遇到这两类旧版能力，先定向 search 检查 Catalog 是否已新增等价 command；仍无匹配就明确说明缺口，不要拿组织列表或业绩明细冒充。

## 横切约定

- 需要稳定解析时使用 `--output-format json`（或 `-f json`）
- 分页默认 `page=1`、`page-size=20`，除非场景另有说明
- 写操作：只读定位 → 变更 → 只读复核；破坏性操作需用户确认
- 组织、商户和员工标识优先从当前上下文或只读查询获得，不复制 eval/示例里的 ID
- 回复默认展示名称、类型、状态等业务字段；手机号和内部标识只在用户确有需要时最小化展示
- 场景未覆盖的能力走 search，不要猜 command 名
- 商品 / 订单 / 客户 / 导购主路径分别交给 `product-ops`、后续 order skill、`crm-ops`、`guide-management`
