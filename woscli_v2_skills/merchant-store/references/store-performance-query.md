# 查询门店业绩

## 何时用

用户要按时间查看门店销售/发货/归属业绩、汇总门店 GMV 或做门店排名时使用。

该命令返回业绩订单明细，不是完整经营看板，也不包含店铺待办、流量、库存等全部指标。

## 创建查询口径

执行前确认：

- 时间范围：`yyyy-MM-dd HH:mm:ss`
- 业绩类型：`1` 销售业绩、`2` 发货业绩、`3` 归属业绩
- 时间口径：`1` 订单支付时间、`2` 业绩更新时间
- 是否限定某个门店 `vid`

`merchant-id` 使用当前 BasicInfo 的 `bosId`，不要在回复或 skill 中固化它。

## 标准路径

1. 首批从 `last-id=0` 开始：

   ```text
   woscli team store-performance-list \
     --merchant-id <CURRENT_BOS_ID> \
     --performance-type <1|2|3> \
     --time-type <1|2> \
     --start-time "<START>" \
     --end-time "<END>" \
     --last-id 0 \
     --page-size 100 \
     -f json
   ```

   用户明确指定门店时追加 `--vid <STORE_VID>`。

2. 读取 `data.pageList[]`。需要继续滚动时，把本批最后一条的 `id` 原样作为下一次 `--last-id`；不要改成页码。
3. 返回空批次时停止。按 `id` 去重，避免重试或游标处理错误造成重复统计。
4. 用户要汇总时，在完整拉取所需范围后按门店聚合 `performanceAmount`；同时标注时间范围、`performance-type` 与 `time-type`。排名前先说明这是业绩口径，不是包含流量和成本的综合经营排名。

## 已验证参数模式与结果字段

- `id`：滚动游标与去重键
- `performanceAmount`：当前业绩口径金额
- `performanceType`：业绩类型
- `storeName`：门店名称
- `tradeCreateTime`、`tradeNo`：交易明细辅助字段

不要把 `paymentAmount`、`performanceAmount`、`storePerformance` 混为一个指标；只聚合用户确认的业绩口径。

## 失败时

- `merchant-id` 缺失：检查当前 BasicInfo 是否完整，不要猜商户 ID。
- 时间范围无数据：先核对时间口径和时区，再决定是否扩大范围。
- 下一批与上一批重复：确认使用的是上一批最后一条 `id`，并保留按 `id` 去重。
- 用户要店铺待办或完整经营看板：定向 search；本命令只能回答门店业绩明细与基于它的汇总。
