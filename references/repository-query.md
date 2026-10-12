# Repository 与查询

数据库查询尽量封装在 Repository 中，业务代码通过注入的 Repository 调用，不直接编写 SQL 或 DAO/GORM 查询链。优先复用生成方法；不足时在项目规定的非生成 Repository 扩展文件中实现复杂筛选、联表、聚合或原生 SQL，并在 Repository 接口中暴露所需方法。

## 选择已有方法

以下名称来自核对基线，先检查当前实体的实际接口：

| 需求 | 优先查找 |
| --- | --- |
| 主键详情 | `FindById` |
| 条件单条 | `FindOne`、`FindOneWrapper` |
| 列表 | `FindList`、`FindListByWrapper` |
| 分页 | 核对 Repository 内部使用 `ScanByPage`/`FindByPage`，并通过 `DbPage` 计算参数 |
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
- 范围筛选统一使用 `BetweenIfNotZero`。只传起点或终点时先按契约校验成对输入，不能误以为该方法支持单边筛选。数值 0 是合法边界时先核对实际版本的零值语义与项目表示方式；无法正确表达时说明冲突并解决，不能静默丢失筛选，也不能擅自改成手写范围条件。
- 时间字符串先按项目统一方法和时区解析为正确类型，再构造条件；不要把未经校验的字符串当作时间边界。
- AND/OR 混合条件要确认分组；必要时通过 SQL 与结果验证，避免扩大查询范围。

## 分页

基线 `FindPageByWrapper` 接收 `(ctx, pageNumber, pageSize, wrapper回调, query选项...)`，返回 `(列表, total, err)`。回调类型为对应实体的 `gormplus.IGenWrapper[...]`，不要猜测实体 Do 接口名称。

标准分页优先走生成 Repository。特殊联表先检查生成产物是否已有 `FindXxxJoinPage` 等辅助方法；确实不足再在当前表的非生成 Repository 中扩展。

分页查询严格使用 `ScanByPage` 或 `FindByPage`：实体列表通常使用 `FindByPage`，自定义字段投影或联表结果通常使用 `ScanByPage`，按实际查询对象的方法签名调用。不用 `Count` 加 `Offset`/`Limit`/`Find`/`Scan` 拼出另一套分页逻辑。

分页参数统一写为：

```go
offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)
```

遇到 `int((req.PageNumber-1)*req.PageSize), int(req.PageSize)` 或其他手算 offset/limit 的写法时替换为上述调用。基线 `DbPage` 将页码 0 变为 1，将非正 pageSize 变为 10，大于 500 变为 500；它没有完整拒绝负页码和溢出输入，仍需按接口契约验证输入。

基线生成模板的普通 `FindPage`/`FindPageByWrapper` 直接计算 offset/limit，并未调用 `DbPage`。使用这些 Repository 方法前核对内部实现；不符合本规则时修正实际生成模板并重新生成，或在非生成 Repository 扩展方法中实现合规分页。不要直接修改 `_gen.go` 后留下无法复现的产物，也不能把 Repository 的方法名当作内部已经合规的证据。

## DAO 完整名称

不要定义 `order := dao.FinishedGoodsPurchaseOrderEntity`、`supplier := dao.SupplierEntity` 等局部实体别名。查询链中完整书写实体与字段名称，使读者直接看到字段所属的表。`query`、`offset`、`limit`、`rows` 等变量及 SQL 输出列别名 `.As(...)` 不受此限制。

下面是 Repository 内的查询构造片段；最终分页调用按实际签名选择 `ScanByPage` 或 `FindByPage`，时间边界先完成校验及必要解析：

```go
query := gormplus.GenWrap(
    dao.FinishedGoodsPurchaseOrderEntity.WithContext(queryCtx).
        LeftJoin(
            dao.SupplierEntity,
            dao.SupplierEntity.ID.EqCol(dao.FinishedGoodsPurchaseOrderEntity.SupplierID),
        ),
)
entityDo := query.
    Where(dao.FinishedGoodsPurchaseOrderEntity.DeletedAt.IsNull()).
    Where(dao.SupplierEntity.DeletedAt.IsNull()).
    BetweenIfNotZero(dao.FinishedGoodsPurchaseOrderEntity.ArrivalDate, startDate, endDate).
    Apply().
    Select(
        dao.FinishedGoodsPurchaseOrderEntity.ID,
        dao.FinishedGoodsPurchaseOrderEntity.SourceType,
        dao.FinishedGoodsPurchaseOrderEntity.SupplierID,
        dao.SupplierEntity.Name.As("SupplierName"),
        dao.SupplierEntity.FullAddress.As("FullReceiverAddress"),
        dao.FinishedGoodsPurchaseOrderEntity.ArrivalDate,
    )
```

## 请求时间参数与校验

所有请求（Req）中的时间参数均使用 `string`，不使用 `int64` 秒级时间戳等数值类型；包括 `StartTime`、`EndTime`、`StartDate`、`EndDate` 及其他请求时间字段。为每个时间字段添加 `dateTime` 校验，并沿用项目现有标签、校验器注册及请求校验入口，确保运行时真正执行。

例如，采用 `validate` 标签的 go-zero API 项目可按项目约定写为：

```text
PageSysAccessGrantReq {
    PageBaseReq
    GrantObject string `json:"grantObject,optional"` // 授权对象，精准IP地址
    StartTime string `json:"startTime,optional" validate:"omitempty,dateTime"` // 操作开始时间，格式遵循项目 dateTime 校验器
    EndTime string `json:"endTime,optional" validate:"omitempty,dateTime"` // 操作结束时间，格式遵循项目 dateTime 校验器
    Operator string `json:"operator,optional"` // 操作人账号
    Status int64 `json:"status,optional"` // 状态：1正常，2禁用
}
```

`validate:"omitempty,dateTime"` 是标签示例，不假定框架自带名为 `dateTime` 的规则。项目使用其他校验标签时沿用其语法，但规则仍为 `dateTime`；缺少规则时补充注册并接入校验。可选字段允许未传或空字符串，非空值必须验证；必填字段按项目语法组合必填与 `dateTime` 校验。格式和时区以项目统一时间规范为准。

从 API/DTO 源定义修正类型与标签，再更新生成 Types、字段注释和解析逻辑；不要只改生成文件。将旧数值时间改成字符串时检查调用方契约，不保留“秒级时间戳”的旧注释。构造查询前校验、按实际数据库字段类型解析时间，并处理开始时间晚于结束时间及只传单端点的情况。

## 只在涉及缓存时检查

Wrapper 回调中的筛选值不应假定自动进入缓存键。开启缓存或 SingleFlight 时，检查 ExecuteQuery/ExecutePage 和 QueryOption，确认筛选、排序、分页、租户及权限维度均得到区分。

`WithCacheArgsFromStruct` 可以展开请求字段，但不自动证明 Context 中的租户或权限已被隔离。不要因使用了租户 SQL 插件就认定缓存也安全隔离。

普通查询任务不主动新增缓存。需要新增时，同时明确写后失效和事务提交时序。

来源和核对基线见 [来源与验证场景](verification.md)。
