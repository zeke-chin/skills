# 一口价创建能力边界

## 当前结论

当前 `marketing` Catalog 能找到：

- `oneprice-detail`：查询已有一口价活动详情。
- `oneprice-goods-detail`：查询已有一口价活动商品。

当前找不到 `oneprice-create`。精确搜索 `oneprice-create` 只返回上述查询命令和其它类型的创建命令，因此本类型不能通过当前 woscli 创建。

## 标准路径

1. 执行一次精确搜索确认 Catalog 是否已更新：

   ```text
   woscli search "oneprice-create 一口价创建" --category marketing --limit 20 -f json
   ```

2. 只有结果出现描述明确为“一口价创建”的专用 create command，才读取其 help 并重新验证创建闭环。创建命令成功且返回活动 ID 时，确认准确 `activityType` 后按 `post-create-activity-result.md` 回读四字段；没有 ID 不安排等待或自动查找。
3. 仍只有 `oneprice-detail` 和 `oneprice-goods-detail` 时，直接报告“当前 woscli 只能查询已有一口价活动，不能创建”，不要向用户收集活动参数。
4. 在专用 create command 出现并重新闭环前，不调用 `goods shop-goods-list` / `shop-goods-get` 选品，也不让用户提供商品内部 ID。

## 失败时

- search 返回通用 `activity-create`：它要求预先存在的 `activity-id`，且没有商品/SKU 和固定价规则字段，不是一口价创建。
- search 返回限时折扣：不要用折扣价格模拟一口价。

## 不要

- 不要猜 `oneprice-create` 并调用。
- 不要用 `discount-create` 或 `activity-create` 冒充一口价。
- 不要因为只有详情命令就让用户提供活动 ID 来“继续创建”。
