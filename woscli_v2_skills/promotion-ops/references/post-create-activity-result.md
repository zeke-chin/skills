# 创建后返回活动标识与状态

## 何时用

任何营销活动创建命令成功后都读取本文件。目标是返回以下四个字段：

- `activityId`
- `belongVidType`
- `promotionStatus`
- `belongVid`

创建响应负责判断写入是否成功并提供活动 ID；四字段最终值以创建后的 `activity-detail` 只读响应为准。

## 活动类型映射

| 活动类型 | `activityType` | 当前创建响应 |
| --- | ---: | --- |
| 满减满折 | `1` | 通常返回 `data.activityId`、`data.success` |
| 满减邮 | `2` | 通常返回 `data.activityId` |
| 限时折扣 | `3` | 通常返回 `data.activityId` |
| N 元 N 件 | `12` | 通常返回 `data.activityId`、`data.success` |
| 满赠 | `32` | 通常返回 `data.activityId` |
| 单品换购 | `14` | 若按独立的单品换购创建路径执行，通常返回 `data.activityId` |

不要从相近活动猜 `activityType`；表外类型走定向 search/help。

## 标准路径

1. 先按活动 reference 判断创建响应：
   - 有顶层 `error`、`success=false` 或非空失败明细时，按创建失败处理，不回查。
   - HTTP 5xx、超时、连接中断或没有取得响应时，写入状态不确定，不回查、不重复创建。
   - 没有明确业务失败时，创建成功。
2. 从创建响应读取 `data.activityId`：
   - 有值：使用上表对应的 `activityType`，立即执行一次统一只读回查。
   - 无值：报告创建成功，但当前创建接口没有返回活动 ID，因此本次无法安全取得四字段；不要猜 ID、按标题搜索或重复创建。
3. 统一回查命令：

   ```text
   woscli marketing activity-detail --activity-id <ACTIVITY_ID> --activity-type <ACTIVITY_TYPE> --basic-info-vid-type <BASIC_INFO_VID_TYPE> --query-basic false --store-id <BASIC_INFO_VID> -f json
   ```

4. 验收详情响应的 `data.activityId` 与创建响应中的 ID 一致，再读取实际值：

   | 最终字段 | 唯一取值来源 | 说明 |
   | --- | --- | --- |
   | `activityId` | `activity-detail` 的 `data.activityId` | 创建响应中的 ID 用于发起查询和一致性校验 |
   | `belongVidType` | `activity-detail` 的 `data.belongVidType` | 活动实际所属组织节点类型 |
   | `promotionStatus` | `activity-detail` 的 `data.promotionStatus` | 活动当前状态 |
   | `belongVid` | `activity-detail` 的 `data.belongVid` | 活动实际所属组织节点 ID |

5. 四字段都存在时组成精简结果返回；任一字段缺失时保留已取得的值，并明确列出缺失字段，不用创建入参或其它详情命令拼凑。

`promotionStatus` 当前已知：`2` 已结束、`4` 预热、`5` 进行中、`6` 未开始。返回原始数值时可附带中文含义，不要把未知状态值强行映射。

## 已验证边界

- 满减满折、限时折扣、满减邮、满赠、N 元 N 件的创建响应可提供 `activityId`，随后 `activity-detail` 能返回四字段。
- 满减满折使用 `activityType=1`；当前 `fulldiscount-create` 成功响应还会返回 `data.success=true`。若出现 `success=false`，不要进入回查。
- `activity-pair-list` 返回状态和活动 ID，但当前输出契约不含 `belongVid`、`belongVidType`，不能替代本路径。

## 失败时

- `activity-detail` 返回空数据或业务错误：创建结果仍按原创建响应判断；报告“创建成功，但四字段回查失败”，附带已有 `activityId` 和原始错误摘要，不重复创建。
- 创建响应没有 ID：不要调用 `activity-goods-recommend-list` 猜测。它按商品和活动类型返回候选，无法证明候选就是本次创建。
- 活动类型映射不确定：先对准确 command 做 help/search；无法确认时停止回查，避免用错误类型得到空结果。

## 不要

- 不要为了等活动开始而 `sleep`；未开始状态本身就是有效结果。
- 不要把创建请求里的 `store-id` / `basic-info-vid-type` 当成已回读的 `belongVid` / `belongVidType`。
- 不要因回查失败而换 UUID 或再次创建。
- 不要使用无 ID 的标题模糊搜索结果冒充本次活动。
