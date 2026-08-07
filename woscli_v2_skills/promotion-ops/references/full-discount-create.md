# 自动创建满减满折活动

## 自动补全

仅覆盖固定时间、线上订单、全部组织、全部人群、指定一个商品和满 100 元减 10 元的单层规则；满件、打折、循环优惠、多层阶梯和全部商品不套用。用户指定门槛或减免金额时替换对应规则值。

- 商品：从当前商户选择第一个已上架且有正库存的商品，只需要 `goodsId`。
- 名称：`<商品简称>满100减10`，最多 30 字。
- 时间：未指定时取当前时区 15 分钟后开始、开始后 3 天结束，并转换成毫秒时间戳。
- 范围：全部组织 `201`、全部人群 `301`、线上订单 `1`、指定商品 `102`。
- 其它：固定时间 `1`、不打标签 `1`、系统默认专题页 `1`，每次创建生成 UUID。
- 规则：满元 `102`、阶梯计算 `1`、减钱 `1001`、满 100 减 10。

## 自动选品

用户没有指定商品时，执行：

```text
woscli goods shop-goods-list --goods-status 0 --sort-by 2 --sort-order 1 --page 1 --page-size 20 -f json
```

`goods-status=0` 明确表示上架。当前商户上下文由 Gateway 注入；`shop-goods-list` 不接受 `--store-id` 或 `--basic-info-vid-type`。

按 `data.pageList` 顺序检查 `isOnline=true` 且 `goodsStock.goodsStockNum>0` 的候选，并执行：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

确认详情仍为 `isOnline=true` 且至少一个 `skuList[].skuStockNum>0` 后使用该 `goodsId`。前 20 个候选不可用时继续下一页，最多检查 5 页。

用户指定商品名时，执行：

```text
woscli goods shop-goods-list --keyword '<商品名>' --search-type 1 --search-match 2 --goods-status 0 --page 1 --page-size 20 -f json
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

没有准确命中或命中商品无正库存 SKU 时停止并报告，不静默换成其它商品。

## 正式创建必要参数

`rule-biz-vo` 的逻辑值：

```json
{"ruleGroupDataList":[{"promotionContentVOS":[{"resultValue":10,"conditionOperator":1,"conditionValue":"100","levelPriority":1,"maxAmount":10,"resultType":1001}],"groupPriority":1,"calculateType":1,"effectRange":1,"conditionType":102}]}
```

创建参数：

```text
--store-id <BASIC_INFO_VID>
--basic-info-vid-type <BASIC_INFO_VID_TYPE>
--start-date <START_MILLIS>
--end-date <END_MILLIS>
--uuid <UUID>
--fit-people 301
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

`rule-biz-vo` 必须作为一个完整 JSON 对象参数传入；不要把它改成数组或二次 JSON 字符串。

## 标准路径

1. 从 BasicInfo 读取 `vid` 和 `vidType`，只供后续 `marketing` 创建命令使用。创建 command 由 Gateway 注入操作人，不传 `--wid`。不要把这些身份字段传给“自动选品”命令，也不要输出完整 BasicInfo。
2. 严格按本文件“自动选品”中的完整命令选择已上架正库存商品；不要再跳转到其它 reference 推断参数。
3. 生成名称、毫秒时间、UUID 和规则 JSON。
4. 用完整必要参数正式创建一次：

   ```text
   woscli marketing fulldiscount-create <完整参数> -f json
   ```

   当前成功响应包含 `data.activityId`、`data.success=true` 和提示消息。`data.success=false` 或明确业务错误表示失败；成功时读取活动 ID。
5. 按 `post-create-activity-result.md` 使用 `activityType=1` 调用一次统一 `activity-detail`，回读并返回 `activityId`、`belongVidType`、`promotionStatus`、`belongVid`。
6. 报告活动名称、时间、商品范围、满 100 减 10 规则和四字段回查结果。禁止等待活动开始、按标题模糊搜索或重复创建。

## 失败时

- `data.success=false` 或明确业务错误：按失败报告，不执行详情回查，不重复创建。
- 成功响应意外没有 `data.activityId`：报告创建成功但无法安全取得四字段，并指出当前 Provider 响应与已验证契约不一致；不要猜测、等待或重复创建。
- HTTP 5xx、工具超时、连接中断或没有取得创建响应：写入状态不确定，报告卡点并停止，不要再次提交相同或新 UUID。
- `rule-biz-vo must be a JSON object`：仅当错误明确表示写入未开始时，把规则重建为一个完整 JSON 对象参数后重试；不要改成数组或二次 JSON 字符串。

## 不要

- 不要把旧表单字段名直接传给 CLI。
- 不要把 `activity-create` 当满减满折创建。
- 不要为了获得活动 ID 安排两分钟后开始或阻塞等待；活动 ID 由创建响应直接返回，默认开始时间仍使用当前时区 15 分钟后。
- 不要把创建请求中的门槛和金额说成“已从详情回读验证”。
