# 查询客户储值余额与流水

## 何时用

用户要看某个客户的当前余额、冻结余额、本金、赠金，或查询储值/余额变动记录时使用。

## 标准路径

1. 先取得准确 `wid`。只有手机号时：

   ```text
   woscli customer user-info-get --phone "<手机号>" --zone 0086 -f json
   ```

2. 查询当前余额账户：

   ```text
   woscli customer balance-detail --wid <WID> -f json
   ```

   分别读取总余额、可用余额、冻结余额，以及本金和赠金的各自字段。

3. 查询余额流水：

   ```text
   woscli customer balance-transfer-list \
     --wid <WID> \
     --page 1 --page-size 20 \
     -f json
   ```

   用户给了时间区间时增加毫秒时间戳：

   ```text
   --trans-begin-date <开始毫秒时间戳> --trans-end-date <结束毫秒时间戳>
   ```

   根据 `totalCount` 继续翻页；`page-size` 最大 100。

## 已验证参数模式

- `balance-detail` 必须传 `wid`。
- `balance-transfer-list` 的 schema 允许不传 `wid`，但客户场景必须显式传入，避免误查商户级流水。
- 流水重点读取 `balanceChange`、`balance`、`principalChange`、`principal`、`bonusChange`、`bonus`、`changeTypeDesc` 和 `transDate`。

## 结果解释

- 总余额、本金和赠金是不同口径；不要合并后声称都是“可提现本金”。
- `balanceChange` 是本次余额变化，`balance` 是变动后余额；正负方向结合 `isPositive`。
- 当前余额与历史流水独立。流水为空不等于余额为 0。

## 失败时

- 只有手机号：先用 `user-info-get` 换成 `wid`。实测同一客户按 `wid` 有 10 条流水，而手机号身份组合返回 0 条，因此不沿用旧版 skill 的“手机号优先查流水”经验。
- 用户问“储值方案列表”：当前 category 搜索只稳定命中余额规则详情而非已验证的方案列表；定向
  `woscli search "balance stored value plan rule" --category customer -f json`。
- 时间过滤异常：确认传入毫秒时间戳；再去掉时间条件验证基础查询。

