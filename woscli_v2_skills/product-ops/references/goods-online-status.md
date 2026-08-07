# 修改商品上下架状态

## 标准路径

1. 按 `references/goods-search-and-detail.md` 定位商品，从用户请求和只读结果取得当前状态、目标状态和作用范围；不要让用户提供内部 ID。
2. 一次最多 50 个商品。把全部商品、目标状态和门店范围合成唯一一次摘要确认。
3. 品牌/当前上下文变更：

   ```text
   woscli goods online-status \
     --goods-id-list '[<GOODS_ID>]' \
     --is-online <true|false> \
     -f json
   ```

   门店维度变更追加 `--store-id <STORE_ID>` 或 `--store-code "<STORE_CODE>"`。

4. 检查正式响应 `data.returnResult=true`，再用只读命令独立复核：

   ```text
   woscli goods shop-goods-list-by-id \
     --goods-id-list '[<GOODS_ID>]' \
     --store-id <STORE_ID> \
     -f json
   ```

   门店维度必须保留同一个 `store-id`；品牌/当前上下文可省略它。核对目标商品的 `isOnline`。
5. 只有全部目标商品回查一致时报告完成；批量中有不一致就逐项报告。

## 已验证参数模式、风险与边界

- 上下架会影响销售；目标商品和门店范围必须包含在同一次摘要确认中，不要为每个商品或内部 ID 分别提问。
- `isOnline=true` 不等于 `isCanSell=true`。库存为 0、配送或其它条件不满足时，商品可能已上架但仍不可售。
- 品牌详情里的状态不能替代门店状态；写入时带了 `store-id`，回查也必须带相同门店。

## 失败时

- 正式响应不是 `returnResult=true`：按失败处理，不要只根据退出码宣称成功。
- 回查仍是旧状态：等待短暂传播后只读复核一次；仍不一致就报告，不要循环写入。
- 门店回查找不到商品：先检查门店分配关系，再决定是否需要读取商品创建/分配 playbook。
