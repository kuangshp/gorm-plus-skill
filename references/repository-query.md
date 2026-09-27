# Repository 与查询

## 选择已有方法

以下名称来自核对基线，先检查当前实体的实际接口：

| 需求 | 优先查找 |
| --- | --- |
| 主键详情 | `FindById` |
| 条件单条 | `FindOne`、`FindOneWrapper` |
| 列表 | `FindList`、`FindListByWrapper` |
| 分页 | `FindPage`、`FindPageByWrapper` |
| 关联是否存在 | `ExistsByWrapper` |
| 统计 | `CountByWrapper` |
| 创建/批量创建 | `Create`、`CreateBatch` |
| 指定字段更新 | `UpdateById`、`UpdateMapById` |
| 删除/批量删除 | `DeleteById`、`DeleteByIdList` |

`FindOneWrapper` 不叫 `FindOneByWrapper`。`UpdateById` 的基线签名是主键参数加 UpdateOption，不应凭名字传入实体。主键也不一定是单个 int64，应按生成接口和项目规则处理。

单条查询按实际实现区分未找到与数据库错误。批量操作确认空集合语义，不能因过滤条件消失变成全表操作。

## Wrapper 的关键语义

- `WhereIf` 仅在条件为真时加入表达式。合法业务值为 0 时，不能用 `value != 0` 表示是否传参；按请求契约区分未提供和提供零值。
- `Like(col, value)` 为 `%value%`；`LLike` 为 `%value`；`RLike` 为 `value%`。三者在空字符串时跳过条件。需要字面匹配 `%` 或 `_` 时，还应核对转义语义。
- `BetweenIfNotZero(col, min, max)` 在列为空或任一端点为零值时跳过整个范围；有效时使用包含边界的 BETWEEN。
- 只传起点或终点时，按接口契约校验成对输入，或使用明确的单边条件。数值 0 是合法边界时不要盲用 `BetweenIfNotZero`。
- 时间字符串先按项目统一方法和时区解析为正确类型，再构造条件；不要把未经校验的字符串当作时间边界。
- AND/OR 混合条件要确认分组；必要时通过 SQL 与结果验证，避免扩大查询范围。

## 分页

基线 `FindPageByWrapper` 接收 `(ctx, pageNumber, pageSize, wrapper回调, query选项...)`，返回 `(列表, total, err)`。回调类型为对应实体的 `gormplus.IGenWrapper[...]`，不要猜测实体 Do 接口名称。

标准分页优先走生成 Repository。特殊联表先检查生成产物是否已有 `FindXxxJoinPage` 等辅助方法；确实不足再在当前表的非生成 Repository 中扩展。

扩展分页按项目要求使用 `DbPage` 与 `ScanByPage`/`FindByPage`，避免另写分页总数逻辑。基线 `DbPage` 将页码 0 变为 1，将非正 pageSize 变为 10，大于 500 变为 500；它没有完整拒绝负页码和溢出输入。

特别注意：基线生成模板的普通 `FindPage`/`FindPageByWrapper` 直接计算 offset/limit，并未调用 `DbPage`。不能假定所有分页入口都带同样的边界保护；按项目输入校验和模板要求处理。

## 只在涉及缓存时检查

Wrapper 回调中的筛选值不应假定自动进入缓存键。开启缓存或 SingleFlight 时，检查 ExecuteQuery/ExecutePage 和 QueryOption，确认筛选、排序、分页、租户及权限维度均得到区分。

`WithCacheArgsFromStruct` 可以展开请求字段，但不自动证明 Context 中的租户或权限已被隔离。不要因使用了租户 SQL 插件就认定缓存也安全隔离。

普通查询任务不主动新增缓存。需要新增时，同时明确写后失效和事务提交时序。

来源和核对基线见 [来源与验证场景](verification.md)。
