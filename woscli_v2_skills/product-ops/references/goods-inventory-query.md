# 查询 SKU 库存与补货分析

## 标准路径

### 已知 SKU

从商品详情或 SKU 编码查询获得 `skuId` 后：

```text
woscli goods stock-list \
  --store-id <STORE_ID> \
  --sku-id-list '[<SKU_ID>]' \
  -f json
```

一次最多查询 100 个 SKU。读取
`data.skuStoreStockVoList[]`：

- `availableStockNum`：可用库存
- `frozenStockNum`：冻结库存
- `realStockNum`：实际库存
- `goodsId`、`skuId`、`vid`：商品、规格与门店关联键

不要把可用库存、冻结库存和实际库存混为一个值。

### 门店库存列表

```text
woscli goods item-sku-list \
  --basic-info-bos-id <CURRENT_BOS_ID> \
  --search "" \
  --search-type 1 \
  --page-num 1 \
  --page-size 100 \
  --store-id <STORE_ID> \
  -f json
```

用户给商品名时填入 `search`；给规格条码时使用 `search-type=3`。需要按后台分组筛选时先查询 `classify-list`，再加 `--classify-id`。

### 补货分析

1. 明确门店、时间范围和“低库存”阈值。
2. 用 `item-sku-list` 获取目标 SKU，再按批次用 `stock-list` 读取真实库存。
3. 若需要销量或商品状态，补充 `shop-goods-list`；标注其 `realSaleNum` 口径。
4. 用户给了低库存阈值时输出候选补货清单；未给阈值时直接按可用库存升序返回原始盘点结果，不为阈值先追问。没有供应周期、在途量或安全库存时，不编造建议采购量。

## 写入边界

用户要求增减、覆盖或批量修改库存时，读取 `goods-inventory-update.md`。库存查询中的
`availableStockNum` 是计算增量和复核结果的依据，不要直接把它拼成未经确认的写请求。

## 失败时

- `stock-list` 空：确认 SKU 与目标门店存在分配关系，并检查 `store-id`。
- `item-sku-list` 名称无结果：用商品列表/详情确认商品与 SKU，再改用条码精确查询。
- 用户把 0 可售库存等同于商品下架：分别回查 `isOnline` 与库存，不混淆状态。
