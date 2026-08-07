# 查询导购业绩明细

## 何时用

用户要查询一段时间内的导购业绩订单明细，或按导购 wid、工号、手机号、组织编码筛选时使用。

## 标准路径

1. 把用户日期转换为明确的 `yyyy-MM-dd HH:mm:ss` 起止时间，并说明所用时区。
2. 基础查询：

   ```text
   woscli team guide-performance-list \
     --create-time "<START>" \
     --end-time "<END>" \
     --page 1 \
     --page-size 100 \
     -f json
   ```

3. 用户指定导购时，优先从
   `references/guider-and-customer-lookup.md` 获得 `guiderWid`，追加
   `--guider-wid <GUIDER_WID>`。也可使用用户明确提供的
   `--job-number`、`--guider-phone` 或 `--vid-code`。
4. 根据 `data.totalCount` 继续页码分页；以业务组合键去重后再汇总，至少包含交易与导购标识，不能只按金额去重。
5. 回复时明确统计字段：

   - `paymentAmount`：支付金额
   - `commissionAmount`：佣金金额
   - `pointsAmount`、`balanceDiscountAmount` 等：对应抵扣金额
   - `tradeId`、`tradeTime`、`tradeType`、`orderType`：交易明细

   当前响应没有老版统一的 `performanceAmount` 字段。用户只说“业绩”时，先展示明细口径或请其确认要汇总支付金额、佣金还是其它字段，不能自行选一个冒充。

## 已验证参数模式与注意点

- `page-size` 最大 100；页码 1、2 实测无重复。
- `guiderWid` 过滤实测能稳定返回该导购数据。
- 当前返回手机号再反向作为 `guider-phone` 过滤实测为 0 条；优先用 wid。

## 失败时

- 时间范围无结果：核对完整时间和时区，不默认扩大范围。
- 手机号过滤无结果：通过导购列表解析 wid，再按 wid 查询。
- 用户要销售/专属/发货业绩类型筛选：当前命令没有老版对应参数；不要编造枚举。
- 用户要排名：先确定汇总指标，完整分页并去重后再排序。
