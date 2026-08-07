# 修改商品 SKU 价格

## 标准路径

1. 按名称/商品编码/SKU 编码定位商品，读取
   `references/goods-search-and-detail.md`。必须获得准确的 `goodsId`、`skuId` 和当前价格。
2. 从用户描述和只读结果确定目标 SKU、新价格及作用范围。只有门店独立价才传 `store-id` 或 `store-code`；缺少新价格，或品牌价/门店价确实无法判断时，把缺失项合并成一个问题。
3. `sku-list` 只包含用户要求修改的价格字段。下面示例只修改销售价：

   ```text
   --goods-id <GOODS_ID>
   --sku-list '[{"skuId":<SKU_ID>,"salePrice":<NEW_PRICE>}]'
   --wid <CURRENT_WID>
   ```

   `wid` 取当前 BasicInfo/会话身份，不复制其它任务的值，也不在回复中展示。

4. 展示商品、SKU、旧价、新价和作用门店，做唯一一次汇总确认。
5. 用完整参数正式执行：

   ```text
   woscli goods price-update <上方参数> -f json
   ```

6. 普通商品价用详情复核：

   ```text
   woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
   ```

   门店独立价用门店视角复核：

   ```text
   woscli goods shop-goods-list-by-id \
     --goods-id-list '[<GOODS_ID>]' \
     --store-id <STORE_ID> \
     -f json
   ```

7. 只有目标 `skuId` 的目标价格字段与新值一致，才报告改价完成。

## 已验证参数模式与风险

- `sku-list` 是 JSON 数组；多 SKU 可一次提交，把所有 SKU 的旧价/新价放在同一摘要中确认，不逐个追问。
- 用户只要求销售价时，不要顺带覆盖市场价、成本价或建议售价。
- 价格是高影响写操作。名称模糊命中、SKU 不唯一或用户只说“便宜一点”时，先澄清具体对象和数值。

## 失败时

- `skuId` 不属于 `goodsId`：重新执行详情/SKU 查询，不要交换或猜测 ID。
- 正式成功但回查未变化：报告不一致并停止，不要重复写入掩盖问题。
- 门店独立价回查为空：确认商品与目标门店存在分配关系。
