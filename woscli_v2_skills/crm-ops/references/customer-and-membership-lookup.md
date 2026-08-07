# 查询客户、会员资料与会员卡

## 何时用

用户提供手机号、`wid`、姓名或昵称，要定位客户并查看 CRM 基础资料、会员卡、会员方案或会员权益时使用。

## 标准路径

1. 用户提供手机号或 `wid` 时，先做精确解析：

   ```text
   woscli customer user-info-get --phone "<手机号>" --zone 0086 -f json
   woscli customer user-info-get --wid <WID> -f json
   ```

   二选一即可；同时传入时命令按 `wid` 查询。记录返回的 `wid`，后续查询以它为主。

2. 只有姓名或昵称时，用 B 端搜索先找候选人：

   ```text
   woscli customer user-search \
     --fields '["wid","phone","name","nickname"]' \
     --query '[{"field":"name","value":"<姓名>"}]' \
     --is-return-page-result 1 \
     --page 1 --page-size 20 -f json
   ```

   按昵称查询时把 `field` 改为 `nickname`。候选不唯一时先让用户确认；不要把第一条当成目标。

3. 查询完整 CRM 资料前，若上下文没有可确认的 `vidCode`，先列出启用的组织节点：

   ```text
   woscli auth org-list --vid-status 1 --page 1 --page-size 20 -f json
   ```

   从当前 BasicInfo 上下文的返回中选择正确节点，再查询资料：

   ```text
   woscli customer crm-customer-list \
     --wid-list '[<WID>]' \
     --result-types '[1,2,3,4]' \
     --vid-code "<VID_CODE>" \
     -f json
   ```

   `result-types` 依次覆盖基础、身份、归属和拓展信息；单次最多 20 个 `wid`。

4. 查询客户拥有的会员卡：

   ```text
   woscli customer membercard-list \
     --wid <WID> \
     --query-type '[1,2,3,4,5,6]' \
     -f json
   ```

   读取卡状态、等级、方案、有效期和开卡来源。按自定义卡号反查时还必须同时提供
   `--membership-plan-id`，卡号数组最多 10 个。

5. 用户问会员方案或权益时再执行：

   ```text
   woscli customer membercard-plan-list --query-types '[1,2,3,6]' -f json

   woscli customer membercard-privilege-list \
     --wid <WID> \
     --privilege-types '[1,2,3,4,5,6,7,8,9,10,11,12,14,15,16,17,18,19]' \
     -f json
   ```

   方案列表是商户配置，不代表目标客户已持有；客户权益以权益查询和有效会员卡状态为准。

## 已验证参数模式

- `user-search` 的过滤白名单是归属节点、会员状态/等级/方案、姓名、昵称、成为客户/会员时间和应用渠道；不支持 `phone`、`wid` 或未列出的字段。
- `membercard-list` 的查询类型 `4` 依赖类型 `2`；使用完整 `[1,2,3,4,5,6]` 可避免缺少等级名。
- `membercard-privilege-list` 必须同时传 `wid` 和至少一个权益类型。

## 结果与隐私

- 搜索用于找候选，`user-info-get` 用于手机号与 `wid` 的精确互换，二者不可互相替代。
- 默认只返回完成任务所需的客户字段；手机号、证件号、邮箱和生日等信息应脱敏。
- `hasValidMemberCard=false` 与“从未领过卡”不是同一结论；结合会员卡列表和卡状态解释。

## 失败时

- 手机号解析无结果：核对区号和号码，不要转用 `user-search` 的手机号过滤。
- 姓名/昵称未命中：分别尝试另一字段并继续分页；仍无结果就报告当前上下文未找到。
- CRM 资料为空或报节点错误：重新确认 `vidCode` 来自当前 BasicInfo 上下文，不能复制历史 Case 的节点编号。
- 企业微信好友、客户跟进或更复杂人群筛选：执行定向
  `woscli search "<对象 动作 近义词>" --category customer -f json`；无等价 command 时明确边界。

