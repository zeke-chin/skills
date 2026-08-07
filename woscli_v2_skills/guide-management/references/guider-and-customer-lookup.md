# 查询导购及其绑定客户

## 何时用

用户要查启用导购、按名称/门店/手机号找导购，或查看某导购名下、归属、绑定的客户时使用。

“线索、未跟进、已成交”等状态不是本路径的字段，不能把绑定客户回答成导购线索。

## 标准路径

1. 查询启用导购：

   ```text
   woscli team guider-list --is-used 1 --page 1 --page-size 100 -f json
   ```

   可按需增加：

   - `--guider-name "<姓名>"`
   - `--guider-vid <STORE_VID>`
   - `--guider-wid-list '[<GUIDER_WID>]'`
   - `--guider-phone-list '["<完整手机号>"]'`

   导购 wid、手机号或导购 ID 三类精确列表条件选一种；单次最多 50 个。

2. 名称命中多个导购时，用门店、工号或用户确认来消歧。保存
   `data.pageList[].guiderWid` 作为后续稳定关联键。
3. 查询绑定客户：

   ```text
   woscli team guider-customer-list \
     --guider-wid <GUIDER_WID> \
     --biz-vid <GUIDER_BIZ_VID> \
     --page 1 \
     --page-size 100 \
     -f json
   ```

4. 读取 `data.pageList[]` 的 `wid`、`nickName`、`customerPhone`、`bindSceneTypeName` 和 `bindTime`。需要客户完整资料时，把客户 `wid` 交给 `crm-ops` 的客户/会员检索路径。

## 已验证参数模式与注意点

- 当前返回的导购手机号再作为 `guider-phone-list` 回传，实测可能得到 0 条；完整用户输入手机号可以尝试一次，但 0 条不能单独证明导购不存在。
- `guiderWid` 精确过滤已实测稳定。能从导购列表取得 wid 时，后续优先使用 wid。
- `guider-customer-list` 返回的是关系链与绑定场景，不含线索状态、加好友来源或时间区间聚合。

## 失败时

- 手机号无结果：不要模糊猜测；可按姓名/门店缩小范围并让用户确认。
- 绑定客户为空：核对 `biz-vid` 是否是导购业务节点，不能换成其它门店 ID 试探。
- 用户要未跟进线索或新增好友统计：定向 search；当前无等价 command 就说明能力缺口。
