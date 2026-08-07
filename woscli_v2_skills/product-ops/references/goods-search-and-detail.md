# 查询、筛选商品与 SKU

## 标准路径

### 商品列表

基础分页：

```text
woscli goods shop-goods-list --page 1 --page-size 20 -f json
```

`page-size` 最大 20。按用户意图组合以下参数，不要为了“更完整”擅自加筛选：

| 意图 | 参数 |
| --- | --- |
| 名称模糊搜索 | `--keyword "<名称>" --search-type 1 --search-match 1` |
| 名称精确搜索 | `--keyword "<名称>" --search-type 1 --search-match 2` |
| 商品编码精确搜索 | `--keyword "<商品编码>" --search-type 2 --search-match 2` |
| 上架 / 下架 / 售罄 | `--goods-status 0 / 1 / 2` |
| 价格区间 | `--min-price <MIN> --max-price <MAX>` |
| 销量 / 上下架时间 / 价格 / 排序值 | `--sort-by 1 / 2 / 3 / 4` |
| 升序 / 降序 | `--sort-order 0 / 1`，与 `sort-by` 同时使用 |
| 二级商品分组 | `--classify-id <CLASSIFY_ID>` |

读取 `data.pageList[]` 的 `goodsId`、`title`、`outerGoodsCode`、价格、库存、销量和 `isOnline`。当前命令没有“可售/不可售”过滤参数；不要照搬老版 skill 的开关。

用户已经给出商品 ID，或需要指定门店视角的状态/价格/库存时，可按 ID 查询：

```text
woscli goods shop-goods-list-by-id \
  --goods-id-list '[<GOODS_ID>]' \
  --store-id <STORE_ID> \
  -f json
```

该命令可返回 `isOnline`、`isCanSell`、`goodsPrice` 和 `goodsStock`；省略 `store-id` 时使用当前上下文。

### 商品与 SKU 详情

从列表确认目标商品后查询完整详情：

```text
woscli goods shop-goods-get --goods-id <GOODS_ID> --include-sku true -f json
```

读取 `skuList[]` 的 `skuId`、售价、成本价、库存、规格值和编码。名称模糊命中多个商品时，普通查询直接返回候选；后续写操作必须锁定唯一商品且无法从编码、门店或上下文消歧时，才请用户选择一次。

用户给的是 SKU 外部编码或规格条码时直接精确查询：

```text
woscli goods sku-search \
  --outer-sku-code "<SKU_CODE>" \
  --query-sku-price-info true \
  --query-sku-cost-price-info true \
  --query-sku-spec-info true \
  -f json
```

条码场景把 `--outer-sku-code` 换成 `--sku-bar-code`；两者至少传一个，不用商品名称模糊搜索代替。

### 类目、分组和标签

商品类目按层级读取：

```text
woscli goods category-list -f json
woscli goods category-list --parent-id <一级类目ID> -f json
```

商品分组同样按层级读取：

```text
woscli goods classify-list --parent-id 0 --page 1 --page-size 20 -f json
woscli goods classify-list --parent-id <一级分组ID> --page 1 --page-size 20 -f json
```

标签按名称查询：

```text
woscli goods shop-goods-tag-list --name "<标签名称>" --page 1 --page-size 20 -f json
```

类目、分组或标签无合适结果时保留为空，不使用“看起来像”的 ID。

## 已验证参数模式

- `shop-goods-list` 每页最多 20 条；`search-type` 与 `search-match` 必须按名称/编码及模糊/精确意图成对设置。
- `shop-goods-list-by-id`、`sku-search` 的数组和布尔参数保持 JSON/布尔原类型；不把单元素数组改成标量。
- SKU 外部编码与条码至少提供一个；类目和分组先读父级 ID，再查询下一层。

## 失败时

- 名称无结果：检查模糊/精确匹配方式；明显是编码时切换到编码查询。
- SKU 编码无结果：再尝试用户确认的条码字段，不要遍历猜测其它 SKU。
- 只查第一页未命中：根据 `totalCount` 继续分页后再判断不存在。
- `shop-goods-list-by-id` 在目标门店为空：先确认商品是否已分配到该门店。
