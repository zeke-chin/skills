# 秒杀创建能力边界

## 当前结论

当前精确搜索 `seckill-create` 没有返回秒杀专用创建命令。相近的 `activity-create` 不能表达商品/SKU、秒杀价、秒杀库存、每人限购和订单关闭等规则；`discount-create` 是限时折扣，也不是秒杀。

因此当前 woscli 不能创建秒杀活动。

## 标准路径

1. 精确搜索：

   ```text
   woscli search "seckill-create 秒杀创建" --category marketing --limit 20 -f json
   ```

2. 只有出现描述明确为秒杀创建、且 help 含秒杀价、库存和限购规则的专用 create command，才重新做闭环。
3. 当前结果不满足时，报告 Catalog 创建缺口，不询问活动时间、价格或库存。
4. 在专用 create command 出现并重新闭环前，不调用 `goods shop-goods-list` / `shop-goods-get` 选品，也不让用户提供商品内部 ID。

## 失败时

- search 返回 `discount-create`：说明它是限时折扣，不能替代秒杀。
- search 返回 `activity-create`：其 help 没有秒杀规则字段，立即停止。
- 用户愿意改做限时折扣：只有用户明确接受变更活动类型后，才转到限时折扣 reference。

## 不要

- 不要默认把“秒杀”改写成“限时折扣”。
- 不要用短时间窗口加折扣来伪造秒杀。
- 不要在无专用 command 时生成看似完整的秒杀参数。
