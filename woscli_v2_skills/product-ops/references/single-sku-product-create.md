# 创建单 SKU 普通实物商品

## 创建契约：必填参数与获取方式

当前 `shop-goods-create` 契约有 14 个顶层必填参数。命中本场景后直接按下表补齐，不要逐项询问用户，也不要让用户提供内部 ID。只有命令报参数错误或返回契约已经变化时，才执行 `woscli goods shop-goods-create --help -f json`，以其中的 `required` 列表更新本表后重试。

| 必填参数 | 本场景取值 | 如何获取 |
| --- | --- | --- |
| `title` | 商品名称，最多 60 字 | 用户描述 → 已有图片的理解结果 → 明确接口测试时生成“AI接口测试-<商品对象>-<时间戳>” |
| `category-id` | 叶子类目 ID | 优先取语义匹配的同类商品详情 `categoryList` 最末级 ID；没有可信同类商品时，用 `category-list` 先查一级，再带 `parent-id` 查二级 |
| `goods-type` | `1` | 本 playbook 固定为普通实物商品 |
| `image-url` | 主图 URL | 用户给出的 URL → 当前图片处理结果 → 无图时调用 `woscli image generateImage --prompt ...` 并取首个 `urls[]` |
| `deduct-stock-type` | 同类商品值；无法取得时用 `1` | 优先读取同类普通实物现货商品详情的 `deductStockType`；本场景的已验证兜底值 `1` 表示下单扣库存 |
| `goods-image-urls` | 至少含主图的 JSON 数组 | 把 `image-url` 放入数组，例如 `["<IMAGE_URL>"]`；不能去掉外层方括号 |
| `goods-template-id` | 商品模板 ID | 从语义匹配的同类普通实物现货商品详情 `goodsTemplateId` 获取 |
| `init-sales` | `0` | 新商品固定从 0 开始，不询问用户 |
| `is-multi-sku` | `false` | 本 playbook 固定创建单 SKU 商品 |
| `is-online` | 用户明确要求上架时为 `true`，否则为 `false` | 上下架状态直接按用户要求设置；只说创建商品时默认下架 |
| `limit-switch` | `false` | 本 playbook 不创建限购规则 |
| `sub-goods-type` | `101` | 本 playbook 固定为普通实物现货子类型 |
| `operator-wid` | 当前操作人 `wid` | 从当前 BasicInfo 读取，不询问用户 |
| `delivery-list` | 至少一个配送对象的 JSON 数组 | 从与类目、商品类型一致的同一次 `shop-goods-get` 返回的 `performanceWay.deliveryList` 获取；不要拼接不同商品的模板和配送信息 |

`delivery-list` 中每个对象还必须包含 4 个字段：

| 嵌套必填字段 | 如何获取 |
| --- | --- |
| `deliveryId` | 同类商品详情 `performanceWay.deliveryList[].deliveryId` |
| `deliveryNodeShipId` | 同一配送对象的 `deliveryNodeShipId`；接口返回 `0` 时原样保留 |
| `deliveryType` | 同一配送对象的 `deliveryType` |
| `templateId` | 同一配送对象的 `templateId` |

四个配送字段必须来自同一个配送对象。缺少任一字段时，继续查询当前组织内语义匹配的同类商品；真实上新仍无法取得时，把“缺少配送配置”作为一个合并问题告知用户，不能猜 ID。明确的接口测试可以使用当前组织只读查询得到且已验证可用的通用配置，但不能复制其它任务文档中的静态 ID。

### 单 SKU 闭环额外参数

以下参数不在当前顶层 `required` 列表中，但为了创建出可核验的单 SKU 商品，本 playbook 每次都应传入：

| 参数 | 本场景取值与获取方式 |
| --- | --- |
| `sku-list` | 构造一个 SKU。价格取用户值 → 同类商品 SKU → 明确接口测试的安全测试值；`skuStockNum` 取用户值或 `100`，`skuPreStockNum=0`；SKU 编码和条码生成任务内唯一值；图片复用主图 |
| `outer-code` | 用户给出时原样使用并在创建前精确查重；未给出时用毫秒时间戳加任务 ID 或随机后缀生成高熵唯一编码，直接用于创建 |
| `sold-type` | 普通现货固定为 `1` |
| `goods-delivery-mode` | 本闭环固定为 `0` |

`sku-list` 至少补齐 `salePrice`、`marketPrice`、`costPrice`、`skuStockNum`、`skuPreStockNum`、`outerSkuCode`、`skuBarCode`、`imageUrl` 和 `imageUrls`。缺市场价时可取销售价；缺成本价时从同类商品取得不高于销售价的值；明确接口测试且没有可比商品时使用 `19.90 / 29.90 / 10.00`。创建入参中的库存只是期望值；普通创建不额外核验门店真实库存，完整测试或用户明确要求查询、设置库存时才用 `stock-list` 回查。

## 少询问与补值顺序

先从用户描述、当前上下文、BasicInfo 和只读接口生成完整草稿。不要让用户提供内部 ID，也不要逐字段追问；只有无法安全判断商品对象，或真实业务价格仍有多个明显不同的候选时，才把问题合并到最终摘要中。

按以下优先级补值：

| 字段 | 补值来源 |
| --- | --- |
| 商品名称 | 用户描述 → 已有图片理解结果 → 测试场景生成“AI接口测试-<商品对象>-<时间戳>” |
| 主图、轮播图 | 用户 URL → 当前图片理解/编辑结果 → `woscli image generateImage --prompt ...` 的首个 `urls[]` |
| 二级类目 | 语义匹配的同类商品详情 `categoryList` → `category-list` 查询当前一级、二级类目 |
| 模板、配送 | 用商品名称检索同类普通实物现货商品，再从同一次 `shop-goods-get` 复用 `goodsTemplateId` 与 `performanceWay.deliveryList` |
| 销售价、市场价、成本价 | 用户值 → 同类商品 SKU；缺市场价时取销售价，缺成本价时取不高于销售价的同类成本；接口测试且没有可比商品时才用安全测试值 `19.90 / 29.90 / 10.00` |
| 操作人 | 当前 BasicInfo 的 `wid`，不让用户填写 |
| 外部商品编码、SKU 编码、条码 | 用户给出外部商品编码时先精确查重；否则用毫秒时间戳加任务 ID 或随机后缀生成三个不同的高熵唯一值，不做创建前查重 |
| 初始库存 | 用户值 → 当前商品表单约定的默认值 `100`；不能把创建入参当成门店库存结果 |
| 门店 | 仅在用户明确要求分配到门店时，按用户说出的门店名称 → 只读组织查询得到的门店 ID；当前门店上下文或任务中出现门店信息不构成分配授权 |

### 商品表单约定

创建草稿同时遵循当前商品表单约束，并映射到 woscli 参数：

- 商品名称和副标题最多 60 字；副标题没有可靠来源时省略。
- 商品图片是 URL 数组；无图时生成一张，并同时用于主图、轮播图和单 SKU 图片。
- 单 SKU 表单项包含销售价、库存和成本价；未给库存时用表单默认值 `100`，市场价仍按同类商品和上方补值顺序取得。
- `isMultiSku=false`；`isOnline` 在用户明确要求上架时为 `true`，否则为 `false`。未要求定时上下架时不构造 `autoOnlineOfflineDTO`。
- 商品分类列表、标签和商品描述都是可选增强字段。只有只读接口返回可信的分类/标签 ID 时才传；分类项必须同时有 `classifyId`、`name`，标签必须同时有 `tagId`、`name`。商品描述存在时使用 HTML。

这些表单字段不能替代 `shop-goods-create` 的当前命令契约；商品模板、配送、操作人、唯一编码等仍按本 playbook 从当前接口和 BasicInfo 补齐。

### 无图时生成图片

根据商品名称和类目生成包含商品主体、构图、背景、光线以及“无文字无水印”的具体提示词，并显式传入 `--prompt`：

```text
woscli image generateImage \
  --prompt "白色背景电商主图，<商品主体与材质/颜色>，柔和棚拍光线，居中构图，无文字无水印，正方形"
```

读取返回的第一个 `urls[]`，同时作为 `image-url`、`goods-image-urls[0]` 和单 SKU 图片。当前 `generateImage` 即使不加 `-f json` 也返回 JSON；不要把全局 `--output-format` 透传给图片模型。

当前 `generateImageV2` 的 help 没有声明运行时必填的 `prompt`，且当前 Provider 会因其 `output_format` 参数失败，因此优先使用已实测可返回 URL 的 `generateImage`。若 `generateImage` 没有返回 `urls[]`，用更短、明确的提示词重试一次；仍失败就停止，不能编造图片 URL。

### 查询同类商品补模板、配送和价格

先用高信息量商品对象检索同类商品：

```text
woscli goods shop-goods-list \
  --keyword "<商品对象>" \
  --search-type 1 \
  --search-match 1 \
  --page 1 \
  --page-size 20 \
  -f json
```

从结果中优先选普通实物、现货、单 SKU 且语义最接近的商品，再读取详情：

```text
woscli goods shop-goods-get --goods-id <SOURCE_GOODS_ID> --include-sku true -f json
```

只复用业务上确实相同的类目、模板、配送和价格口径；不要复制其它任务、示例或无关类目商品的 ID。同一次详情已经返回可信的二级类目时直接使用，不再重复调用 `category-list`。若名称检索没有同类结果或详情缺少类目，才按语义用 `category-list` 定位叶子类目，再检索该类目中的可比商品；仍无结果时，真实上新缺少模板或配送就合并提问，明确的接口测试才可使用当前组织已验证的通用配置。

### 类目与编码

仅在没有可信同类商品类目时，先查一级，再把候选一级 ID 作为 `parent-id` 查二级：

```text
woscli goods category-list -f json
woscli goods category-list --parent-id <PARENT_CATEGORY_ID> -f json
```

系统自动生成外部商品编码时，必须组合毫秒时间戳和当前任务 ID、`logId` 或随机后缀，避免并发任务碰撞；普通创建直接使用该编码，不调用创建前查重接口。

只有用户明确提供外部商品编码时，正式写入前才精确查重：

```text
woscli goods shop-goods-list \
  --keyword "<OUTER_GOODS_CODE>" \
  --search-type 2 \
  --search-match 2 \
  --page 1 \
  --page-size 20 \
  -f json
```

查重结果必须为 `totalCount=0`。如果用户提供的编码已存在，就停止创建并说明该编码已被占用；不能擅自更换用户给出的编码，也不能覆盖或增量更新旧商品。

## 已验证参数模式

以下模板只表示单 SKU 普通实物现货。所有尖括号值优先来自用户描述、当前上下文和只读查询，不要求用户提供内部 ID：

```text
--title "<TITLE>"
--category-id <LEAF_CATEGORY_ID>
--goods-type 1
--sub-goods-type 101
--deduct-stock-type 1
--image-url "<MAIN_IMAGE_URL>"
--goods-image-urls '["<IMAGE_URL>"]'
--goods-template-id <GOODS_TEMPLATE_ID>
--init-sales 0
--is-multi-sku false
--is-online <true|false>
--limit-switch false
--operator-wid <CURRENT_WID>
--delivery-list '[{"deliveryId":<DELIVERY_ID>,"deliveryNodeShipId":<NODE_SHIP_ID>,"deliveryType":<DELIVERY_TYPE>,"templateId":<DELIVERY_TEMPLATE_ID>}]'
--sku-list '[{"salePrice":"<SALE_PRICE>","marketPrice":"<MARKET_PRICE>","costPrice":"<COST_PRICE>","skuStockNum":<INITIAL_STOCK_OR_100>,"skuPreStockNum":0,"outerSkuCode":"<SKU_CODE>","skuBarCode":"<SKU_BARCODE>","imageUrl":"<IMAGE_URL>","imageUrls":["<IMAGE_URL>"]}]'
--outer-code "<OUTER_GOODS_CODE>"
--sold-type 1
--goods-delivery-mode 0
```

`category-type` 不是当前创建契约的顶层必填项；只有当前类目或同类商品详情明确返回该值时才传，不要对普通类目猜 `0`。JSON 数组必须保留外层方括号，编码和条码必须唯一。

## 执行模式

- 单个商品创建是一个业务目标，禁止调用 `todo_write` 或维护内部 todo。只有批量创建多个商品，或同一请求还包含多个独立写操作时才使用 todo。
- 默认使用普通创建。普通创建执行创建前补值、正式创建和一次详情复核，然后立即返回；只有用户明确提供外部商品编码时才增加创建前查重。
- 只有用户明确要求“完整测试”“测试创建接口”“完整跑一遍”或同义目标时，才使用完整测试。完整测试复用普通创建已经取得的详情，再增加门店库存、外部编码唯一性和完整字段验收。
- 完整测试不等于门店分配或上架授权。只有用户明确要求“分配到某门店”时才执行门店分配；只有用户明确要求上架时才进入上架流程。
- 当前 BasicInfo、当前门店上下文、上层任务描述中附带的门店信息，都只用于定位上下文，不能视为门店分配授权。未获得明确授权时，禁止调用 `shop-goods-store-assign`、`store-relation-list` 或为分配目的搜索门店命令。

## 标准路径

1. 从用户描述提取商品对象；没有名称但有图片时先理解图片，没有图片时按“无图时生成图片”调用 `woscli image generateImage --prompt ...`。
2. 查询同类商品并用一次详情尽量补齐叶子类目、模板、配送与价格；只有详情缺失或语义不匹配时再查类目接口。从 BasicInfo 取 `wid`。用户未提供编码时生成高熵唯一的外部商品编码、SKU 编码和条码并直接进入创建；只有用户明确提供外部商品编码时才先精确查重。
3. 向用户展示商品名称、类目、价格、图片、配送、默认库存和目标上下架状态，做唯一一次汇总确认。用户明确要求上架时目标状态为上架，未要求时默认下架；只有用户明确要求门店分配时，摘要中才加入目标门店。用户已明确说“完整测试”“直接创建”“按默认值创建”或同义授权时，这就是正式写入确认，不再重复询问。
4. 用完整参数正式创建：

   ```text
   woscli goods shop-goods-create <上方参数> -f json
   ```

   保存 `data.goodsId` 和全部 `data.skuList[].skuId`；响应缺少任一值就停止后续写操作。
5. 立即读取详情：

   ```text
   woscli goods shop-goods-get --goods-id <NEW_GOODS_ID> --include-sku true -f json
   ```

   普通创建核对 `goodsId`、标题、外部编码、图片、SKU、价格和用户要求的上下架状态。完整测试复用同一份详情，额外核对类目、模板、配送、销售类型和限购；不要为了完整测试重复读取详情。
6. 普通创建到此结束。如果用户没有明确要求门店分配、完整测试或库存查询/设置，立即返回创建响应和商品详情；禁止追加 `stock-list`、外部编码回查、门店关系查询或其它扩展验证。
7. 仅在用户明确要求分配到指定门店时，使用确认摘要中的目标门店执行。完整测试、当前门店上下文或任务中出现门店信息都不能触发本步骤：

   ```text
   woscli goods shop-goods-store-assign \
     --assign-type 1 \
     --goods-ids '[<NEW_GOODS_ID>]' \
     --vid-list '[<STORE_VID>]' \
     --is-put-away 0 \
     -f json
   ```

   执行后用 `woscli goods store-relation-list --goods-id <NEW_GOODS_ID> -f json`
   确认目标门店出现在 `data.vidList`。
8. 仅在完整测试，或用户明确要求查询、设置库存时，对每个新 `skuId` 查询门店真实库存：

   ```text
   woscli goods stock-list \
     --store-id <TARGET_STORE_OR_CURRENT_VID> \
     --sku-id-list '[<NEW_SKU_ID>]' \
     -f json
   ```

   库存为 `0` 且用户已明确要求设置初始库存时，读取 `goods-inventory-update.md`，按普通商品路径更新并再次回查。若当前 Provider 仍返回操作人缺失等发布卡点，就说明商品已创建但库存未生效；上下架状态仍按用户的明确要求执行，同时报告 `isOnline=true` 不等于 `isCanSell=true` 的不可售风险。
9. 仅在完整测试模式下，用唯一外部商品编码做精确查询，必须恰好命中刚创建的一个商品：

   ```text
   woscli goods shop-goods-list \
     --keyword "<OUTER_GOODS_CODE>" \
     --search-type 2 \
     --search-match 2 \
     --page 1 \
     --page-size 20 \
     -f json
   ```

10. 创建请求中已经明确要求上架时，直接通过 `shop-goods-create --is-online true` 完成，不再追加 `online-status`。商品已经以下架状态创建完成、用户之后又明确要求上架时，才按下方“已创建商品上架”执行。

## 已创建商品上架

本节只处理已经创建完成且当前为下架状态的商品。用户在创建请求中已经明确要求上架时，直接把创建参数 `is-online` 设为 `true`，不要再调用本节命令；只说创建商品、普通创建或完整测试都不构成上架授权。

1. 优先复用创建结果中的 `goodsId` 和详情，不重复查询。若当前上下文不能唯一确定商品，再按唯一外部编码定位；不要让用户提供内部 ID。不要仅为了上架额外调用 `stock-list`。
2. 已有查询结果明确显示库存为 `0`、配送缺失或其它条件不满足时，不阻止用户明确要求的上架操作，但必须说明 `isOnline=true` 不等于 `isCanSell=true` 以及可能的不可售风险。
3. 使用商品 ID 上架：

   ```text
   woscli goods online-status \
     --goods-id-list '[<NEW_GOODS_ID>]' \
     --is-online true \
     -f json
   ```

   默认修改品牌/当前上下文状态。用户明确要求门店维度上架时，必须已经取得并确认目标门店，再追加 `--store-id <STORE_ID>` 或 `--store-code "<STORE_CODE>"`；写入和后续回查必须使用同一门店范围。
4. 只有正式响应包含 `data.returnResult=true` 才进入复核。响应不是 `true` 时按失败处理，不要只根据退出码宣称成功。
5. 独立回查最终状态：

   ```text
   woscli goods shop-goods-list-by-id \
     --goods-id-list '[<NEW_GOODS_ID>]' \
     -f json
   ```

   门店维度上架时追加与写入相同的 `--store-id <STORE_ID>`。核对新商品的 `isOnline=true`；`isOnline=true` 不等于 `isCanSell=true`。只有已经执行过 `stock-list` 时才报告门店实际库存，不能把商品详情里的 `skuStockNum` 当成门店库存。
6. 回查仍为下架时，等待短暂传播后只读复核一次；仍不一致就报告上架未生效，不循环执行写命令。最终摘要报告商品创建状态、上下架状态和可售状态；仅在已明确查询库存时报告门店实际库存。

## 完整测试验收

只有用户明确要求测试接口或完整跑一遍时才执行本节。最终报告应逐项给出：

- 生图命令是否返回可用 URL；图片 URL 是否出现在商品详情
- 正式创建是否返回唯一 `goodsId` 和 `skuId`
- 详情中的标题、类目、模板、配送、编码、价格和用户要求的上下架状态是否与草稿一致
- 外部商品编码精确查询是否 `totalCount=1`
- `stock-list` 的真实库存；若与创建入参不同，明确报告差异和可售风险
- 用户明确要求门店分配时，报告分配成功或失败；未要求时只写“未要求，未执行”，并且不能为此调用任何门店接口
- 用户在创建请求中明确要求上架时，报告创建详情中的最终上架状态；创建完成后另行调用 `online-status` 时，分别报告上架响应和独立回查结果
- 总结为“主链路通过”“部分通过”或“失败”，不要只说命令退出码为 `0`

## 失败时

- 创建响应没有 `goodsId` 或 SKU：停止，不猜新商品 ID。
- 类目/模板/配送 ID 无效：重新从当前商品与类目查询获取，不回退到其它任务的示例。
- 图片命令没有返回 URL：最多换短提示词重试一次，仍失败就停止，不用历史示例 URL 冒充新图。
- 用户提供的外部商品编码查重已存在：停止创建并报告编码已被占用；不要擅自换码，也不要覆盖或增量更新旧商品。自动生成的高熵编码若被创建接口明确拒绝为重复编码，重新生成一次后再创建。
- 商品已创建但门店分配失败：报告已创建的商品 ID 与分配失败，不重复创建商品。
- 完整测试或明确库存查询中的门店库存为 0：报告实际库存和不可售风险；不要把创建参数中的初始库存或商品详情里的 `skuStockNum` 当成门店库存结果。
- 超时或结果不确定：先按唯一外部编码精确查询，确认是否已创建，再决定是否重试。
