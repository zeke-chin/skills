# 拼团创建能力边界

## 当前结论

当前精确搜索 `groupon-create` 没有返回拼团专用创建命令。相近结果中的 `activity-create` 只是通用活动元数据登记，字段只有产品实例、预先存在的活动 ID、名称、图片、说明、时间、状态和组织节点；它不能表达商品/SKU、团价、成团人数、单次开团有效期等拼团规则。

因此当前 woscli 不能创建拼团活动。

## 标准路径

1. 精确搜索：

   ```text
   woscli search "groupon-create 拼团创建" --category marketing --limit 20 -f json
   ```

2. 只有出现描述明确为拼团创建、且 help 含商品/SKU、团价和成团规则的专用命令，才重新做创建闭环。
3. 当前结果不满足时，直接报告 Catalog 缺口，不继续组装参数，也不让用户逐项提供团价或成团人数。
4. 在专用 create command 出现并重新闭环前，不调用 `goods shop-goods-list` / `shop-goods-get` 选品，也不让用户提供商品内部 ID。

## 失败时

- search 返回 `activity-create`：读取 help 后确认缺少拼团规则即停止。
- search 返回满赠、满减满折等相近促销：类型不一致，不能替代。

## 不要

- 不要把通用活动登记说成拼团创建。
- 不要把 N 元 N 件或限时折扣包装成拼团。
- 不要在没有专用 command 时向用户收集内部产品实例 ID。
