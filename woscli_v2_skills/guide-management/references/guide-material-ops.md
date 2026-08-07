# 查询与创建导购素材

## 何时用

用户要查询内容素材、查看素材详情、创建自定义图文素材，或查看素材分享/收藏统计时使用。

## 标准路径

### 查询素材

`biz-vid` 与 `biz-vid-type` 使用当前 BasicInfo：

```text
woscli team guide-material-list \
  --biz-vid <CURRENT_VID> \
  --biz-vid-type <CURRENT_VID_TYPE> \
  --page 1 \
  --page-size 100 \
  -f json
```

可按 `publish-status`、`classify-id`、`start-time`、`end-time` 筛选。列表 ID 字段是 `contentMaterialId`；查询详情时映射到 `material-id`：

```text
woscli team guide-material-detail \
  --material-id <CONTENT_MATERIAL_ID> \
  --biz-vid <CURRENT_VID> \
  --biz-vid-type <CURRENT_VID_TYPE> \
  -f json
```

### 创建已验证的自定义图文素材

当前闭环只验证 `material-type=1`、全部门店、单张图片关联：

```text
--biz-vid-type <CURRENT_VID_TYPE>
--biz-vid <CURRENT_VID>
--title "<最多16字标题>"
--material-type 1
--content "<最多200字文案>"
--scope-info-scope-type 0
--link-list "[]"
--relate-list '[{"relateType":1,"imageUrl":"<IMAGE_URL>"}]'
```

标准步骤：

1. 确认标题、文案、图片、适用范围；图片必须来自用户或当前可信素材，不复制 eval URL。
2. dry-run：

   ```text
   woscli team guide-material-create <上方参数> --dry-run -f json
   ```

3. 展示摘要并取得明确确认，去掉 `--dry-run` 正式创建。
4. 保存 `data.contentMaterialId`，立即执行 `guide-material-detail`，核对标题、内容、图片、素材类型和适用范围。
5. 列表有传播延迟或默认排序差异时，使用创建时间范围筛选回查；详情回查仍是主验收。

### 素材行为统计

最多一次查询 20 个素材：

```text
woscli team material-behavior-statistic-list \
  --material-id-list '[<MATERIAL_ID>]' \
  --biz-vid <CURRENT_VID> \
  --biz-vid-type <CURRENT_VID_TYPE> \
  --page 1 \
  --page-size 20 \
  -f json
```

当前环境对新素材和抽样已有素材均可能返回空列表；空结果只能表示接口当前未返回统计，不能推断分享/收藏为 0。

## 已验证参数模式

- 素材列表和详情都必须使用当前 `biz-vid`、`biz-vid-type`；列表 ID `contentMaterialId` 映射为详情参数 `material-id`。
- 当前创建闭环只覆盖 `material-type=1`、`scope-type=0`、空链接数组和单张图片的 `relate-list`。
- 行为统计的 `material-id-list` 单次最多 20 个；空列表不补成 0。

## 失败时

- 创建成功但列表暂未出现：按创建时间筛选后只读重查一次；仍以详情是否一致为准。
- `relate-list` JSON 错误：保持外层数组和内部双引号，不改成单对象。
- 用户要视频、多关联对象或部分门店：当前 reference 未覆盖，先 search/help，不套用单图模板。
