# 查询客户积分与积分流水

## 何时用

用户要看某个客户的当前积分、冻结/可用积分，或分页查询积分变动记录时使用。

## 标准路径

1. 先取得准确 `wid`。只有手机号时：

   ```text
   woscli customer user-info-get --phone "<手机号>" --zone 0086 -f json
   ```

2. 查询当前积分账户：

   ```text
   woscli customer point-detail --wid <WID> -f json
   ```

   分开读取 `totalPoint`、`frozenPoint`、`availablePoint`，并检查账户和客户状态。

3. 查询积分流水：

   ```text
   woscli customer point-transfer-list \
     --wid <WID> \
     --page 1 --page-size 20 \
     -f json
   ```

   用户给了时间区间时再增加毫秒时间戳：

   ```text
   --trans-begin-date "<开始毫秒时间戳>" --trans-end-date "<结束毫秒时间戳>"
   ```

   根据 `totalCount` 继续翻页；`page-size` 最大 100。

## 已验证参数模式

- `point-detail` 和 `point-transfer-list` 都要求 `wid`。
- 时间区间参数在当前 command 中是字符串形式的毫秒时间戳；开始和结束成对传入。
- 流水结果重点读取 `transNo`、`transDate`、`changeTypeDesc`、`variationRange`、`amount`、`ruleName` 和发生门店。

## 结果解释

- `variationRange` 是本次变动值，`amount` 是变动后的积分；不要互换。
- 当前积分与历史流水是两次独立查询。流水为空不能直接说明当前积分为 0。
- 返回的客户名称、操作人、外部业务号按需展示，避免泄露无关个人或交易信息。

## 失败时

- 只有手机号、会员卡号或昵称：优先先解析并确认 `wid`。实测已确认有流水的客户用手机号身份组合筛选仍返回 0 条，因此标准路径不使用旧版 skill 的身份筛选捷径。
- 用户问“积分方案列表”：当前 category 搜索只稳定命中规则详情而非已验证的方案列表；定向
  `woscli search "point plan rule" --category customer -f json`，不要伪造列表能力。
- 时间过滤异常：检查是否传了毫秒而非秒，并用不带时间的第一页确认客户本身有流水。

