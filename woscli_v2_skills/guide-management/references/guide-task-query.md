# 查询导购任务与执行进度

## 何时用

用户要查导购任务模板、某个导购的任务实例、任务执行进度，或任务关联客户时使用。

## 标准路径

### 任务模板

```text
woscli team quest-template-list --page 1 --page-size 100 -f json
woscli team quest-template-detail --template-no "<TEMPLATE_NO>" -f json
```

先从列表确认模板编号、标题、类型、起止时间和状态，再查详情；不要猜模板编号。

### 导购任务实例

先通过导购列表取得 `guiderWid`：

```text
woscli team quest-instance-list \
  --guider-wid <GUIDER_WID> \
  --start-time "<START>" \
  --end-time "<END>" \
  --page 1 \
  --page-size 100 \
  -f json
```

时间可选；也可以用导购工号或手机号筛选。列表确认 `assignmentNo` 后：

```text
woscli team quest-instance-detail \
  --guider-wid <GUIDER_WID> \
  --assignment-no "<ASSIGNMENT_NO>" \
  -f json
```

核对任务标题、状态、完成状态、起止时间和模板编号。

### 任务客户

客户运营任务可继续查询：

```text
woscli team quest-customer-list \
  --guider-wid <GUIDER_WID> \
  --assignment-no "<ASSIGNMENT_NO>" \
  --page 1 \
  --page-size 100 \
  -f json
```

当前响应只返回 `customerWid`。需要姓名、手机号、会员信息或消费资料时，把这些 wid 交给 `crm-ops`；不要从任务列表补造客户资料。

## 已验证参数模式与边界

- 模板详情使用列表返回的 `templateNo`；实例详情和客户列表使用实例返回的 `assignmentNo`。
- 实例与客户查询都以已确认的 `guiderWid` 为稳定关联键；分页上限使用当前已验证的 100。
- 任务实例的 `completeStatus` 是任务执行状态，不等同于老版导购线索的未查看/未跟进/已成交状态。
- 本 reference 是查询路径，不包含创建任务目标、派发任务或自动生成客户跟进话术。

## 失败时

- 模板或实例无结果：核对当前组织、导购 wid 和时间范围。
- 详情为空：确认编号来自同一列表结果，不拿模板编号替代任务实例编号。
- 任务客户只有 wid：按 CRM 路径补查；不要调用绑定关系命令当作客户资料。
