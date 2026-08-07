# 自动创建满赠活动

## 自动补全

仅覆盖固定时间、线上订单、全部组织、全部人群、指定一个适用商品、满 100 元赠 1 件指定商品和每人参与 1 次。

- 适用商品：按商品列表顺序选择第一个已上架正库存商品。
- 赠品：选择下一个不同商品的正库存 SKU。
- 名称：`<适用商品简称>满100赠礼`，最多 30 字。
- 时间：当前时区 15 分钟后开始，开始后 3 天结束。
- 规则：满元 `102`、阶梯赠送一次 `1`、满 100、赠品 1 件、每人参与 1 次。
- 范围：全部组织 `201`、全部人群 `301`、线上订单 `1`、指定适用商品 `102`。
- 页面：系统默认样式 `1`，分享图使用默认 URL。

## 自动选品

用户没有指定商品时，执行：

```text
woscli goods shop-goods-list --goods-status 0 --sort-by 2 --sort-order 1 --page 1 --page-size 20 -f json
```

`goods-status=0` 明确表示上架。当前商户上下文由 Gateway 注入；`shop-goods-list` 不接受 `--store-id` 或 `--basic-info-vid-type`。

按 `data.pageList` 顺序对 `isOnline=true` 且商品总库存大于 0 的候选逐个执行：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

详情仍为上架且存在正库存 SKU 才是可用商品。选择前两个不同的可用 `goodsId`：第一个作为适用商品，第二个作为赠品；赠品 SKU 按正库存最大、`skuId` 最小选择。前 20 个候选不足两个时继续下一页，最多检查 5 页。

用户指定适用商品或赠品名称时，对每个名称分别执行：

```text
woscli goods shop-goods-list --keyword '<商品名>' --search-type 1 --search-match 2 --goods-status 0 --page 1 --page-size 20 -f json
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

指定商品无准确命中、两个角色命中同一商品或赠品无正库存 SKU 时停止并报告，不静默换商品。

## 正式创建必要参数

`gift-market-rule-level-vos` 的逻辑值：

```json
[{"giftMarketGoodsVOS":[{"singleLimit":1,"goodsId":123,"skuIds":[456],"skuId":456}],"levelPriority":1,"fullGiftCondition":100,"fullGiftLimit":1}]
```

将 `123`、`456` 替换为自动选择的赠品 `goodsId`、`skuId`。创建参数：

```text
--title '<活动名称>'
--start-date '<YYYY-MM-DD HH:mm:ss>'
--end-date '<YYYY-MM-DD HH:mm:ss>'
--sync-tag 1
--select-store-type 201
--select-people-type 301
--select-goods-type 102
--select-goods-ids '[<QUALIFYING_GOODS_ID>]'
--use-of-scene '1'
--project-page-style 1
--full-gift-factor 102
--full-gift-type 1
--limit-num 1
--store-id <BASIC_INFO_VID>
--basic-info-vid-type <BASIC_INFO_VID_TYPE>
--promotion-share-image 'https://image-c-dev.weimobwmc.com/qa-Oofb/2c9b8a1b38ab4142bba9057fe87604da.jpg'
--gift-market-rule-level-vos <RULE_JSON_ARRAY>
```

## 标准路径

1. 读取 BasicInfo 的组织信息，只供后续 `marketing gift-create` 使用；不要把 `vid`、`vidType` 传入“自动选品”命令。
2. 严格按本文件“自动选品”中的完整命令选择两个不同的已上架正库存商品，并取得赠品 SKU。
3. 生成名称、时间和规则 JSON。
4. 使用上面的完整必要参数正式创建一次：

   ```text
   woscli marketing gift-create <完整参数> -f json
   ```

5. 命令成功且没有明确业务失败时报告创建成功、适用商品、赠品商品/SKU、时间和规则；`data.activityId` 存在时按 `post-create-activity-result.md` 回读四字段，不存在时说明接口未返回活动 ID、无法安全取得四字段。

## 失败时

- 找不到两个不同的可用商品：停止并说明当前商品条件不足，不让用户抄内部 ID。
- 赠品 SKU 无正库存：重新读取商品详情；不得猜 SKU。
- 创建命令成功但没有 `activityId`：按创建成功报告，并说明本次无法安全取得四字段；不要等待、猜测或重复提交。
- HTTP 5xx、工具超时、连接中断或没有取得创建响应：写入状态不确定，报告卡点并停止，不重复提交。

## 不要

- 不要提交空赠品数组；旧表单中的空数组只是页面初始态。
- 不要把适用商品和赠品商品静默混为同一个商品。
- 统一 `activity-detail` 只用于回读四字段；不要用详情响应替代已提交的门槛和赠送组数。
