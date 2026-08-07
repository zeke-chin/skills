# 查询客户消费统计与 CDP 标签字典

## 何时用

已经确认客户 `wid`，要补充累计消费次数、金额、件数、最近消费等统计；或要查看当前组织可用的 CDP 标签定义时使用。

## 标准路径

1. 批量查询已知客户的消费统计：

   ```text
   woscli cdp consumption-list --ukey-list '[<WID_1>,<WID_2>]' -f json
   ```

   单次 1～100 个用户 ID。用返回的 `ukey` 与请求列表关联，读取总消费次数/金额、件数、最近下单时间等实际存在的字段。

2. 用户问标签定义时，先从当前 BasicInfo 或 `auth org-list` 确认组织节点，再查询标签字典：

   ```text
   woscli cdp cdp-tag-list \
     --vid <VID> \
     --vid-type <VID_TYPE> \
     --has-cover-number true \
     --hide-enabled true \
     --page 1 --page-size 50 \
     -f json
   ```

   根据 `totalCount` 继续分页。返回的是标签及属性定义，不是某个客户已经拥有的标签。

## 已验证参数模式

- `consumption-list` 是 CRM 连锁 PE 及以上版本的高级接口；版本不满足时应报告能力限制。
- 消费统计是对已知用户 ID 的补充查询，不是按消费条件从全库筛客户的接口。
- `cdp-tag-list` 的标签 ID/属性 ID 批量查询上限为 20；普通分页 `page-size` 使用 50 已验证。

## 结果解释

- 某个 `ukey` 未返回记录时只能说明当前接口没有该用户的统计记录，不能把所有指标补成 0。
- 金额、次数、件数和售后是不同口径；只展示用户实际要求的统计项。
- 标签覆盖人数是标签定义侧统计，不能据此断言目标客户命中标签。

## 失败时

- `consumption-list` 报版本或权限错误：保留 CRM 基础资料结果并明确高级接口不可用，不要改用猜测字段。
- 标签列表为空：确认 `vid`、`vid-type` 来自当前上下文，并尝试 `hide-enabled=false`；不要复制历史 Case 的节点 ID。
- 用户要按消费、积分或余额条件圈选整个人群：定向
  `woscli search "<筛选条件 人群 提取>" --category cdp -f json`，本 reference 不替代人群任务。

