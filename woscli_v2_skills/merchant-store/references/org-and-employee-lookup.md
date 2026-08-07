# 查询组织、门店和员工

## 何时用

用户要查当前商户下的组织/门店、按名称找门店、查看门店地址和营业时间，或查询员工列表时使用。

## 标准路径

1. 先确认查询基于当前 BasicInfo 上下文；不要从历史 Case、示例或其它商户复制 `bosId`、`vid`。
2. 查询启用的组织。用户给了名称时使用服务端模糊过滤；明确只查门店时再加 `--vid-type 10`：

   ```text
   woscli auth org-list --vid-name "<组织名称关键词>" --vid-status 1 --page 1 --page-size 20 -f json
   ```

   用户要查全部组织层级时省略 `--vid-type`。读取
   `data.data[]` 中的 `vid`、`vidCode`、`vidName`、`vidType`、`vidStatus` 和 `parentVid`。

3. 需要地址、电话、营业时间、网店状态等详情时，把上一步确认的节点组成数组：

   ```text
   woscli auth org-detail-list \
     --vid-list '[<VID_1>,<VID_2>]' \
     --fields '["provinceName","cityName","countyName","detailAddress","contactTels","onlineStatus","businessHours"]' \
     -f json
   ```

   核对返回节点与请求节点一致。单次最多查询 50 个节点；更多节点分批执行。

4. 查询员工时分页执行：

   ```text
   woscli auth employee-list --page 1 --page-size 100 -f json
   ```

   命令没有姓名或手机号服务端过滤参数。用户要找指定员工时，在每页
   `data.data[]` 中做精确匹配；未命中就继续下一页，不能只查第一页便断言不存在。

## 已验证参数模式

- `org-list`：`vid-name` 为模糊名称过滤，`vid-status=1` 表示启用，`vid-type=10` 表示门店；`page-size` 最大 50。
- `org-detail-list`：`vid-list` 与 `vid-code-list` 选其一，数组最多 50 项；`fields` 只传本次需要的详情字段。
- `employee-list`：`page-size` 最大 100；`status=0` 表示启用、`1` 表示禁用。

## 结果与隐私

- 先用名称、节点类型、状态帮助用户确认目标，再按需展示地址、营业时间等详情。
- 员工手机号属于个人信息。查找时可以用于精确匹配，回复时默认脱敏；只有用户明确需要且有业务必要时才展示完整号码。
- 组织列表不是“可访问商户列表”，员工列表也不是导购列表；不要跨概念回答。

## 失败时

- 名称无结果：去掉 `vid-type` 重试一次，排除用户把品牌/区域叫作“门店”的情况；仍无结果就报告当前上下文未找到。
- 详情为空：检查 `vid-list` 是否来自同一 BasicInfo 上下文，不要换成示例 ID。
- 用户问商户列表或店铺待办：定向
  `woscli search "<对象 动作 近义词>" --category auth -f json`；没有等价 command 就说明当前不支持。

