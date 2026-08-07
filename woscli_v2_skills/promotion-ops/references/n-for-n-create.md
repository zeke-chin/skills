# 自动创建 N 元 N 件活动

## 自动补全

仅覆盖固定时间、线上订单、全部组织、全部人群、指定一个商品、2 件固定总价和每人参与 1 次。

- 商品：自动选择已上架、正库存且 SKU 售价大于 10 元的商品。
- 名称：`<商品简称>2件更省`，最多 30 字。
- 时间：当前时区 15 分钟后开始，开始后 3 天结束，并转换为毫秒时间戳。
- 范围：全部组织 `201`、全部人群 `301`、线上订单 `1`、指定商品 `102`。
- 规则：满件 `103`、循环计算 `2`、条件 2 件、结果类型一口价 `1003`、结果值 20。
- 其它：限购 1 次、不打标签 `1`、系统默认专题页 `1`，每次创建生成 UUID。

## 自动选品

用户没有指定商品时，执行：

```text
woscli goods shop-goods-list --goods-status 0 --sort-by 2 --sort-order 1 --page 1 --page-size 20 -f json
```

`goods-status=0` 明确表示上架。当前商户上下文由 Gateway 注入；`shop-goods-list` 不接受 `--store-id` 或 `--basic-info-vid-type`。

按返回顺序对 `isOnline=true` 且商品总库存大于 0 的候选执行：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

详情仍为上架时，从 `skuList` 中选择 `skuStockNum>0` 且 `salePrice>10` 的 SKU；多个可用 SKU 按库存最大、`skuId` 最小选择。前 20 个候选不可用时继续下一页，最多检查 5 页。

用户指定商品名时，执行：

```text
woscli goods shop-goods-list --keyword '<商品名>' --search-type 1 --search-match 2 --goods-status 0 --page 1 --page-size 20 -f json
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

没有准确命中，或命中商品没有售价大于 10 元的正库存 SKU 时停止并报告，不静默替换商品。

## 正式创建必要参数

以“2 件 20 元”为例，`rule-biz-vo` 的逻辑值：

```json
{"ruleGroupDataList":[{"promotionContentVOS":[{"resultValue":20,"conditionOperator":1,"conditionValue":"2","levelPriority":1,"resultType":1003}],"groupPriority":1,"calculateType":2,"effectRange":1,"conditionType":103}]}
```

创建参数：

```text
--store-id <BASIC_INFO_VID>
--basic-info-vid-type <BASIC_INFO_VID_TYPE>
--start-date <START_MILLIS>
--end-date <END_MILLIS>
--fit-people 301
--uuid <UUID>
--limit-num 1
--marking-type 1
--select-store-type 201
--time-type 1
--title '<活动名称>'
--topic-style-type 1
--use-of-scene '1'
--select-goods-type 102
--select-goods-ids '[<GOODS_ID>]'
--rule-biz-vo <RULE_JSON_OBJECT>
```

## 标准路径

1. 读取 BasicInfo 的组织信息，只供后续 `marketing nynj-create` 使用；不要把 `vid`、`vidType` 传入“自动选品”命令。
2. 严格按本文件“自动选品”中的完整命令选择售价大于 10 元的正库存商品/SKU。
3. 生成名称、毫秒时间、UUID 和规则 JSON。
4. 使用上面的完整必要参数正式创建一次：

   ```text
   woscli marketing nynj-create <完整参数> -f json
   ```

   `data.success=false` 或明确业务错误才表示失败。否则报告创建成功、商品、2 件套餐价和时间；`data.activityId` 存在时按 `post-create-activity-result.md` 回读四字段，不存在时说明接口未返回活动 ID、无法安全取得四字段。

## 失败时

- `/promotion/nynj/create` 未找到：重新读取当前 `nynj-create` help 并报告 Catalog/Provider 不一致；不要回退到旧的 `/promotion/nynj/nynj/create`。
- 创建返回 `success=false` 或明确业务错误：按失败报告，不重复提交。
- 创建命令成功但没有活动 ID：按创建成功报告，并说明本次无法安全取得四字段；不要等待、猜测或重复提交。
- HTTP 5xx、工具超时、连接中断或没有取得创建响应：写入状态不确定，报告卡点并停止，不重复提交。
- `rule-biz-vo must be a JSON object`：仅当错误明确表示写入未开始时，把规则重建为一个完整 JSON 对象参数后重试；不要改成数组或二次 JSON 字符串。

## 不要

- 不要用旧表单的 `calculateType=1` 覆盖已实跑的 `2`。
- 不要把一口价活动与 N 元 N 件互换。
- 统一 `activity-detail` 只用于回读四字段；不要用详情响应替代已提交的件数和金额。
