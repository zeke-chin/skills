# 自动创建限时折扣活动

## 自动补全规则

仅覆盖固定时间、全部组织、全部人群、线上订单、不预热、不限购和单个有效商品/SKU；其它变体按主 skill 的决策协议定向 search。

| 字段 | 用户未指定时的规则 |
| --- | --- |
| 商品 | 未指定商品时，从当前商户已上架商品中按列表顺序选择第一个存在正库存 SKU 的商品 |
| 指定商品 | 先按商品名查询；优先标题完全匹配，否则使用返回顺序中第一个已上架且存在正库存 SKU 的商品 |
| SKU | 选择库存最大的正库存 SKU；库存相同则选择 `skuId` 较小者 |
| 活动名称 | 取商品标题前 18 个字符，去掉首尾空白，再拼接“限时八折”；总长度不得超过 30 |
| 开始、结束时间 | 当前时区 15 分钟后开始，开始后 3 天结束；格式 `yyyy-MM-dd HH:mm:ss` |
| 折扣 | `discount=8`，即八折 |
| 活动库存 | `min(floor(skuStockNum), 100)`；结果必须大于 0 |
| 活动标签 | `限时折扣` |
| 组织 | `store-id` 使用当前 BasicInfo 的 `vid`，`basic-info-vid-type` 使用 `vidType` |
| 其它玩法 | 全部门店、全部人群、线上订单、不预热、不限购、不打人群标签 |
| 订单自动关闭 | 固定传 `5`；当前 help 未声明单位，不向用户扩展解释或让用户选择未知单位 |

旧表单 schema 使用 `limitType=1` 表示不限购，并给出订单自动关闭时间 `3～360` 分钟的前端范围；当前 CLI 已实跑模板分别使用 `limit-type=0` 和 `order-auto-closed-time=5`。这里继续以当前 CLI 实跑值为准，不用旧表单编码覆盖。

### 图片取值

限时折扣使用固定的默认活动图和分享图：

1. 用户明确给出活动图片或分享图片时，分别使用用户给出的 URL。
2. 用户未给活动图片时，`image-url` 固定使用 <https://image-c-dev.weimobwmc.com/qa-Oofb/01e293b06ceb490ead5ba56790138c61.jpg>。
3. 用户未给分享图片时，`promotion-share-image` 固定使用 <https://image-c-dev.weimobwmc.com/qa-Oofb/2c9b8a1b38ab4142bba9057fe87604da.jpg>。

其余场景直接使用上述默认 URL，无需向用户询问；只有用户明确要求定制图时才走图片生成能力。

## 自动选品

用户没有指定商品时：

```text
woscli goods shop-goods-list --goods-status 0 --sort-by 2 --sort-order 1 --page 1 --page-size 20 -f json
```

`goods-status=0` 明确表示上架。当前商户上下文由 Gateway 注入；这条商品列表命令不接受 `--store-id` 或 `--basic-info-vid-type`，不要把后续 `marketing discount-create` 的组织参数提前带入。

按 `data.pageList` 顺序检查候选。对 `isOnline=true` 且商品总库存大于 0 的候选执行：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

从 `data.skuList` 按“正库存最大、`skuId` 最小”选择 SKU。前 20 个候选都不可用时继续下一页，最多检查 5 页；仍无可用 SKU 时停止并报告“当前商户没有可自动用于限时折扣的已上架正库存商品”，不要让用户抄 `goodsId` 或 `skuId`。

用户指定商品名时：

```text
woscli goods shop-goods-list --keyword '<商品名>' --search-type 1 --search-match 2 --goods-status 0 --page 1 --page-size 20 -f json
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

指定商品没有可用 SKU 时停止并报告，不得静默换成完全不同的商品。

## 正式创建必要参数

固定时间、八折、单个 SKU 使用以下参数。尖括号值均由前述规则自动生成：

```text
--title '<活动名称>'
--time-type 1
--start-date '<YYYY-MM-DD HH:mm:ss>'
--end-date '<YYYY-MM-DD HH:mm:ss>'
--sync-tag 1
--limit-type 0
--order-auto-closed-time 5
--select-store-type 201
--select-people-type 301
--use-of-scene '1'
--warm-up-type 0
--tag-name '限时折扣'
--activity-countdown 0
--image-url 'https://image-c-dev.weimobwmc.com/qa-Oofb/01e293b06ceb490ead5ba56790138c61.jpg'
--store-id <BASIC_INFO_VID>
--basic-info-vid-type <BASIC_INFO_VID_TYPE>
--repeat-days '[1]'
--promotion-share-image 'https://image-c-dev.weimobwmc.com/qa-Oofb/2c9b8a1b38ab4142bba9057fe87604da.jpg'
--discount-goods <DISCOUNT_GOODS_JSON_ARRAY>
--available-asset-rule-vos '[]'
```

`discount-goods` 的逻辑值必须是 JSON 数组：

```json
[{"goodsId":123,"sort":1,"availableStockNum":0,"discountType":1001,"ignoreChangeType":1,"skuList":[{"skuId":456,"changeCanSaleNum":100,"discount":8}]}]
```

即使只有一个商品和一个 SKU，也必须保留两层数组。`changeCanSaleNum` 是本模板使用的活动库存字段；不要把它表述为所有限时折扣变体都必须使用的规则。即使 `time-type=1`，当前契约仍把 `repeat-days` 标为必填，因此保留 `[1]`。用户明确提供某个图片 URL 时，只替换对应的固定默认值。

## 标准路径

1. 从当前 BasicInfo 读取 `vid` 和 `vidType`，只供后续 `marketing discount-create` 使用；不要把它们传入“自动选品”中的 `shop-goods-list` / `shop-goods-get`，也不要输出完整 BasicInfo 或让用户提供内部 ID。
2. 按“自动选品”定位商品/SKU，确认 `isOnline=true`、SKU 存在且 `skuStockNum>0`，并计算活动库存。
3. 自动生成活动名称、时间和固定玩法参数；图片参数按“图片取值”使用两个固定默认 URL，除非用户明确覆盖。
4. 若用户要求预览，在此返回摘要并停止；否则使用上面的完整必要参数正式创建一次：

   ```text
   woscli marketing discount-create <完整参数> -f json
   ```

5. 检查创建响应：

   - `data.activityId` 存在时按 `post-create-activity-result.md` 做一次统一只读回查；没有返回时也不视为创建失败
   - `data.resultVOS` 返回时必须是数组
   - `resultVOS` 非空时逐项报告 `goodsId` 和 `errorMsg`，不得把部分失败说成完整成功
   - 命令成功且 `resultVOS` 没有业务失败时，报告创建成功、时间、商品/SKU、折扣和活动库存；有活动 ID 时补充四字段回查结果，没有就说明接口未返回活动 ID、无法安全取得四字段

## 失败时

- `discount-goods must be a JSON array`：仅当错误明确表示写入未开始时，把商品规则重建为一个完整 JSON 数组参数后重试；不要改成单对象或二次 JSON 字符串。
- `skuId非法`：重新读取当前商品详情；不得猜 SKU 或换成用户未指定的其它商品。
- 创建返回 `resultVOS` 非空：按部分失败处理，保留活动 ID 和失败商品详情。
- 创建命令成功但没有 `activityId`：按创建成功报告，并说明本次无法安全取得四字段；不要等待、猜测或重复创建。
- HTTP 5xx、工具超时、连接中断或没有取得创建响应：写入状态不确定，报告卡点并停止；不要查询后重试，也不要盲目重复创建。
