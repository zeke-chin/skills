# 自动创建满减邮活动

## 自动补全

仅覆盖固定时间、线上订单、全部组织、全部人群、指定一个商品和满 99 元全地区包邮；满件、指定地区或减部分运费不套用。

- 商品：自动选择第一个已上架且有正库存的商品。
- 名称：`<商品简称>满99包邮`，最多 30 字。
- 时间：当前时区 15 分钟后开始，开始后 3 天结束，格式 `yyyy-MM-dd HH:mm:ss`。
- 范围：全部组织 `201`、全部人群 `301`、线上订单 `1`、指定商品 `102`。
- 规则：满元 `102`、满 99、包邮 `2001`、不限制地区 `0`。
- 页面：系统默认样式 `1`，分享图使用默认 URL。
- 其它：不打标签 `1`；每次正式创建生成一个 UUID，结果不确定时保留该 UUID，禁止换 UUID 重复提交。

## 自动选品

用户没有指定商品时，执行：

```text
woscli goods shop-goods-list --goods-status 0 --sort-by 2 --sort-order 1 --page 1 --page-size 20 -f json
```

`goods-status=0` 明确表示上架。当前商户上下文由 Gateway 注入；`shop-goods-list` 不接受 `--store-id` 或 `--basic-info-vid-type`。

按返回顺序检查 `isOnline=true` 且商品总库存大于 0 的候选，再执行：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

详情仍为上架且存在正库存 SKU 时使用该 `goodsId`。前 20 个候选不可用时继续下一页，最多检查 5 页。

用户指定商品名时，执行：

```text
woscli goods shop-goods-list --keyword '<商品名>' --search-type 1 --search-match 2 --goods-status 0 --page 1 --page-size 20 -f json
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

没有准确命中或商品无正库存 SKU 时停止并报告，不静默替换商品。

## 正式创建必要参数

`free-freight-rule-level-vos` 的逻辑值：

```json
[{"postageCondition":99,"areaJson":"[]","postageType":2001,"levelPriority":1,"postageReduceAmt":0,"limitDistrictType":0}]
```

创建参数：

```text
--title '<活动名称>'
--start-date '<YYYY-MM-DD HH:mm:ss>'
--end-date '<YYYY-MM-DD HH:mm:ss>'
--sync-tag 1
--postage-setting-factor 102
--select-store-type 201
--select-goods-type 102
--select-goods-ids '[<GOODS_ID>]'
--select-people-type 301
--project-page-style 1
--uuid <UUID>
--store-id <BASIC_INFO_VID>
--basic-info-vid-type <BASIC_INFO_VID_TYPE>
--use-of-scene '1'
--time-type 1
--repeat-type 1
--repeat-start-interval '00:00:00'
--repeat-end-interval '23:59:59'
--repeat-week-days '[1]'
--repeat-days '[1]'
--promotion-share-image 'https://image-c-dev.weimobwmc.com/qa-Oofb/2c9b8a1b38ab4142bba9057fe87604da.jpg'
--free-freight-rule-level-vos <RULE_JSON_ARRAY>
```

当前 contract 即使 `time-type=1` 也把四个重复时间字段标为 required，因此保留上述占位值；不要据此把活动描述成周期活动。

## 标准路径

1. 读取 BasicInfo 的组织信息，只供后续 `marketing freefreight-create` 使用；不要把 `vid`、`vidType` 传入“自动选品”命令。
2. 严格按本文件“自动选品”中的完整命令选择一个已上架正库存商品。
3. 生成名称、固定时间、UUID 和包邮规则。
4. 使用上面的完整必要参数正式创建一次：

   ```text
   woscli marketing freefreight-create <完整参数> -f json
   ```

5. 命令成功且没有明确业务失败时报告创建成功、标题、时间、适用商品和满 99 包邮规则；`data.activityId` 存在时按 `post-create-activity-result.md` 回读四字段，不存在时说明接口未返回活动 ID、无法安全取得四字段。

## 失败时

- 创建命令成功但没有 `activityId`：按创建成功报告，并说明本次无法安全取得四字段；不要等待、猜测或重复创建。
- HTTP 5xx、工具超时、连接中断或没有取得创建响应：写入状态不确定，报告卡点并停止，不重复创建。
- `free-freight-rule-level-vos must be a JSON array`：仅当错误明确表示写入未开始时，把规则重建为一个完整 JSON 数组参数后重试，并保留数组外壳。

## 不要

- 纯包邮 `2001` 不要追问减免邮费金额。
- 不要省略 fixed-time 模式下 contract 仍要求的 repeat 字段。
- 统一 `activity-detail` 只用于回读四字段；不要用详情响应替代已提交的包邮规则。
