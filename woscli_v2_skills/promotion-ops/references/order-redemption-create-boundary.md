# 订单换购创建能力边界

## 当前结论

当前 Catalog 有：

- `redemption-order-detail`：查询已有订单换购活动。
- `redemption-create`：描述明确为“单品换购创建活动”。

没有订单换购 create command。单品换购与订单换购的门槛、规则归属和活动类型不同，不能因为都含“换购”就互换。

## 标准路径

1. 精确搜索：

   ```text
   woscli search "redemption-order-create 订单换购创建" --category marketing --limit 20 -f json
   ```

2. 若只返回 `redemption-create` 和 `redemption-order-detail`，报告“当前只能创建单品换购、查询订单换购，不能创建订单换购”，立即停止。
3. 只有出现描述明确为订单换购创建的命令，才读取 help 并重新验证创建闭环。创建命令成功且返回活动 ID 时，确认准确 `activityType` 后按 `post-create-activity-result.md` 回读四字段；没有 ID 不安排等待或自动查找。
4. 在订单换购专用 create command 出现并重新闭环前，不调用 `goods shop-goods-list` / `shop-goods-get` 选品，也不让用户提供商品内部 ID。

## 失败时

- 用户只说“换购”且没有说明单品或订单：只问一次准确类型。
- 用户已明确说“订单换购”：不要再问类型，也不要调用单品换购命令。
- search 只有详情命令：说明只能查询已有活动，不要求用户提供 ID 以继续创建。

## 不要

- 不要把 `redemption-create` 映射成订单换购。
- 不要把旧表单的满元/满件门槛直接拼进单品换购请求。
- 不要用通用 `activity-create` 替代。
