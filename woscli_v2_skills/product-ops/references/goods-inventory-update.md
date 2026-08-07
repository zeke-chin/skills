# 修改商品库存

## 发布状态保护

- `stock-update`、`virtual-update` 和新版 `batch-update-stock` 的操作人 `wid` 应由 Provider 从当前会话注入。命令不需要 `--wid`；遇到 `operateId` 为空时说明当前 Provider 尚未发布对应修复，停止并报告，不向用户索要 `wid`，也不原样重试。
- 只有 `woscli goods batch-update-stock --help -f json` 已显示 `--goods-list` 时，才使用本文的批量覆盖命令。若仍显示旧的 `--input`，改用 `stock-update` 分批覆盖普通商品；不要把新版参数套到旧契约，也不要继续使用已废弃的 `--input` 配方。

## 标准路径

1. 定位商品和 SKU：

   ```text
   woscli goods shop-goods-get \
     --goods-id <GOODS_ID> \
     --include-sku true \
     -f json
   ```

   从当前响应读取 `goodsId`、`goodsType`、`subGoodsType` 和真实 `skuList[].skuId`。用户只给名称或编码时，先按 `goods-search-and-detail.md` 定位；写操作目标仍不唯一时才让用户选择一次。

2. 按商品类型拆分，不能混批：

   - 普通实物商品：使用 `stock-list` 读取变更前的 `availableStockNum`。
   - `goodsType=2` 的虚拟/服务商品：使用商品详情中的 `skuStockNum` 作为变更前库存；不要混入普通商品的 `stock-list` 批次。

3. 根据用户语义选择编辑类型：

   - “增加 N”使用 `quantity-edit-type=1`，`stockNum=N`。
   - “减少 N”使用 `quantity-edit-type=1`，`stockNum=-N`；计算后库存不得小于 0。
   - “设置为 / 覆盖为 N”使用 `quantity-edit-type=0`，`stockNum=N`。
   - 用户同时给出增量和目标库存但二者与当前库存不一致时，合并成一个问题澄清；不要静默选择。

4. 组装完整 JSON 数组：

   ```json
   [{"goodsId":123,"skuList":[{"skuId":456,"stockNum":100}]}]
   ```

   即使只有一个商品和一个 SKU，也必须保留两层数组。商品 ID 与 SKU ID 不可互换。

5. 普通商品使用：

   ```text
   woscli goods stock-update \
     --store-id <CURRENT_VID> \
     --quantity-edit-type <0_OR_1> \
     --goods-list '<GOODS_LIST_JSON_ARRAY>' \
     -f json
   ```

   单批 SKU 不超过 50 个，超过时分批。增量请求的成功项不能重发，否则会重复加减库存。

6. 虚拟或服务商品先取仓库：

   ```text
   woscli goods warehouse-list-by-vid \
     --store-id <CURRENT_VID> \
     -f json
   ```

   从当前响应读取可用的 `warehouseId` 和 `warehouseCode`，再执行：

   ```text
   woscli goods virtual-update \
     --store-id <CURRENT_VID> \
     --warehouse-id <WAREHOUSE_ID> \
     --warehouse-code <WAREHOUSE_CODE> \
     --quantity-edit-type <0_OR_1> \
     --goods-list '<GOODS_LIST_JSON_ARRAY>' \
     -f json
   ```

7. 每批都检查 `data.failList`，空数组才表示该批全部成功。非空时按
   `goodsId`、`skuIdSet` 和 `message` 汇报；`message` 为空且响应包含
   `skuFailList` 时，继续读取 `skuFailList[].skuId` 与 `failMsg`，不能只依据进程退出码。

8. 只读复核：

   - 普通商品再次调用 `stock-list`，核对 `availableStockNum`。
   - 虚拟/服务商品再次调用 `shop-goods-get --include-sku true`，核对 `skuStockNum`。
   - 超时、连接中断或“请求频繁”时先复核，无法确认当前库存前不要重发增量请求。

## 批量覆盖

用户明确要求覆盖多商品库存，且当前 help 已显示 `--goods-list` 时，可以使用：

```text
woscli goods batch-update-stock \
  --goods-list '<GOODS_LIST_JSON_ARRAY>' \
  -f json
```

该命令只表达覆盖，不用于“增加 / 减少”。同样检查 `data.failList`；如果当前 help 仍要求 `--input`，就按标准 `stock-update` 路径分批处理。

## 失败时

- 返回 `operateId` 为空：这是 Provider 发布状态问题，不是缺少用户参数；停止并汇报，不添加 `--wid`。
- 普通命令拒绝虚拟商品：读取商品详情确认 `goodsType=2`，改走 `virtual-update`，不要原样重试。
- 返回 `sku不存在`：重新读取当前商品详情确认 SKU；仍一致时报告该商品不支持此库存路径，不猜替代接口。
- 部分 SKU 失败：已成功 SKU 不回滚也不重发，只对 `failList` 明确列出的失败项处理。
- 复核值与目标不一致：报告命令响应和实际库存，不为“修正结果”连续重复写入。

## 不要

- 不要手工拼接或展示 BasicInfo 中的 `wid`。
- 不要把虚拟/服务商品和普通商品放入同一写入批次。
- 不要在未读取当前库存时把“增加 100”解释成“设置为 100”。
