---
description: '当用户要查询、搜索、筛选或排序商品，查看商品详情、SKU、编码或条码，查询类目、分组或标签， 修改商品价格或上下架状态，查询、盘点或修改库存，创建或上新商品，以图或无图发品，商品创建时使用。

  '
name: product-ops
---



# product-ops（商城 · 商品）

## Category 绑定

- 主 category：`goods`
- search 默认：`woscli search "..." --category goods -f json`
- 跨 category：仅当 `references/` 步骤写明；商品缺图时可调用工具型 `image` category，默认不把 `shoppingmall` 当商品运营 category；活动创建归 `promotion-ops`

## 决策协议

1. 命中下方场景索引 → Read 对应 `references/...`，按步骤执行
2. 未命中 → `woscli search "<对象 动作 近义词>" --category goods -f json`
3. 参数不明或失败 → `woscli goods <command> --help -f json`
4. 禁止无目标 `woscli goods --help` 翻页

## 场景索引

| 意图关键词 | 文件 |
| --- | --- |
| 查商品 / 名称或编码搜索 / 筛选排序 / 商品详情 / SKU 条码 / 类目分组标签 | `references/goods-search-and-detail.md` |
| 改价 / 调价 / 修改 SKU 售价 / 门店独立价 | `references/goods-price-update.md` |
| 上架 / 下架 / 修改商品上下架状态 | `references/goods-online-status.md` |
| 查库存 / SKU 库存 / 库存盘点 / 补货分析 | `references/goods-inventory-query.md` |
| 增加库存 / 减少库存 / 补库存 / 设置库存 / 覆盖库存 / 批量改库存 | `references/goods-inventory-update.md` |
| 创建商品 / 以图或无图发品 / 自动补参数 / 单 SKU 上新 / 完整测试创建接口 / 分配门店 | `references/single-sku-product-create.md` |

### 当前已知边界

- 库存修改按商品类型分别使用 `stock-update` 或 `virtual-update`；操作人 `wid` 应由 Provider 从会话身份注入，不作为业务参数询问或手工传入。当前部署仍可能返回 `operateId` 为空，按库存修改 reference 的发布卡点处理。
- 商品创建闭环只验证普通实物、现货、单 SKU；上下架状态按用户明确要求设置，未要求时默认下架。只有用户明确要求分配到指定门店时才执行门店分配；当前门店上下文或任务中出现门店信息不构成分配授权。多规格、预售、虚拟商品、限购、周期购等走定向 search/help，不套用单 SKU 模板。
- 创建入参中的 `skuStockNum` 不保证成为门店真实库存；未给库存时参考商品表单约定填 `100`。普通创建不额外查询门店库存，也不宣称已验证真实库存；只有完整测试或用户明确要求查询、设置库存时才调用 `stock-list`。
- 老版商品 skill 的“可售状态过滤”在当前 `shop-goods-list` 没有等价入参；可读取返回的 `isCanSell`，不要编造筛选参数。

## 横切约定

- 需要稳定解析时使用 `--output-format json`（或 `-f json`）
- 分页默认 `page=1`、`page-size=20`，除非场景另有说明
- 只读查询直接使用当前上下文、BasicInfo 和已写明默认值；不要让用户提供商品、SKU、门店等内部 ID
- 写操作：只读定位 → 一次汇总确认 → 正式变更 → 独立只读复核；不逐字段或逐 SKU 重复确认
- 用户已明确说“完整测试”“直接创建”或同义授权时，将其视为正式写入确认，不重复询问
- 商品创建默认走普通创建，正式创建后用商品详情复核一次即返回；只有用户明确要求“完整测试”“测试接口”或“完整跑一遍”时才增加编码唯一性与门店库存等完整测试验收。完整测试不自动授权门店分配或上架。
- 单个商品创建是一个业务目标，禁止使用 `todo_write` 或维护内部 todo；只有批量创建多个商品或同时包含多个独立写操作时才使用 todo。
- 系统自动生成的外部商品编码必须包含毫秒时间戳和任务 ID 或随机后缀，普通创建不做创建前编码查重；只有用户明确提供外部商品编码时才在写入前精确查重。完整测试的创建后唯一性回查和超时恢复查询仍保留。
- 用户已明确说“增加 / 减少 / 设置库存”并给出可唯一定位的商品和数值时，将其视为本次库存变更授权；只读取得内部 ID 和当前库存后直接执行一次，不再追加摘要确认
- 只有目标仍有多个候选，或价格等业务值确实无法推导时才合并成一个问题
- 创建商品必须先读 `references/single-sku-product-create.md` 的必填参数表，按其中来源自动补齐；命令报参数错误或契约变化时才定向查看 `shop-goods-create --help`
- JSON 对象/数组参数必须保留完整 JSON；单元素数组也不能去掉外层 `[...]`
- 商品、SKU、门店、类目、模板和配送 ID 必须来自当前上下文的只读结果，不复制其它任务或示例 ID
- 场景未覆盖的能力走 search，不要猜 command 名
- 订单发货证据变厚后优先拆独立 skill；营销活动创建使用 `promotion-ops`，不要写进本 skill
