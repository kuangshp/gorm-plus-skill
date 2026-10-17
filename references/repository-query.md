# Repository 与查询

数据库查询尽量封装在 Repository 中，业务代码通过注入的 Repository 调用，不直接编写 SQL 或 DAO/GORM 查询链。优先复用生成方法；不足时在对应的已有非生成 Repository 文件中新增方法，实现复杂筛选、联表、聚合或原生 SQL，并同步维护对应的已有 Repository 接口定义。

## 业务层与 Repository 的职责边界（强制）

- Repository 只负责数据访问：查询条件、联表、聚合、分页、持久化、事务连接绑定，以及模板已有的 Context、租户过滤和缓存失效机制；返回数据、行数或持久化错误，不决定业务是否允许执行。
- Logic/Service 或项目已有业务层负责请求业务校验、业务权限和状态判断、计算、跨 Repository 调用顺序，以及业务错误码/文案与业务异常映射。数据访问所需的条件和参数可传给 Repository，但判断结果的业务含义留在业务层。
- 例如“分类必须为末级”“已有多规格商品时不能新增必选属性”“某状态不能删除”等规则，必须由业务层通过 Repository 读取分类、存在性或数量后判断。Repository 可返回分类、是否存在、数量等数据，不能调用 `leafCategory` 等业务校验函数，或返回上述规则对应的 `domain.Invalid(...)` 等业务错误。
- 禁止为了上述校验重写模板 `Create`、`Update`、`Delete`，也不能把规则藏到同文件私有函数、Repository 装饰器、模板或 `_gen.go` 中。保留模板 CRUD，由业务层完成判断后调用。
- 事务不会改变职责边界：多表业务流程及校验在业务层编排，通过实际支持的 `WithTx`、事务版方法或等效机制，让查询和写入绑定同一事务；Repository 只执行数据操作。依赖原子性的校验不能简单移到事务外，必要的锁与数据库约束仍需保留，见[事务](transactions.md)。
- 调整已有代码时，将业务规则迁回现有 Logic/Service；只将确实缺失的数据访问部分留在对应的已有 Repository 文件。不为本次分层额外创建空 Service 或第二套 CRUD，也不把 DAO/GORM 查询链迁到业务层。

## Repository 文件与方法约束（强制）

- 禁止在 `repository` 目录及其子目录中手动新建任何文件；不能以复杂查询、联表、事务、辅助方法或拆分职责为由另建文件，也不能用复制、重命名或脚本写文件绕过。新增 Repository 文件必须来自项目生成器的目标表生成流程。
- 增加方法前，先定位对应表或领域已有的非生成 Repository 文件、实现类型和接口定义。方法实现追加到该文件中，方法声明补充到对应的已有接口定义中，沿用原有接收者和构造函数；业务层字段命名与初始化遵循[默认接口开发流程](interface-workflow.md)中的统一方式；不要另建 Repository 实现或接口文件。
- 缺失的联表、聚合或事务内数据读写方法放入对应的已有 Repository 文件，复用已有类型和依赖；跨表业务事务由 Logic/Service 编排，不在 Repository 中增加业务流程方法，也不创建专用的 join、query、transaction、helper 或 extra 文件。
- 若对应的非生成文件缺失，检查生成配置与模板，通过项目生成器生成；生成失败先修复原因，不能手写空壳、改写 `_gen.go` 或将查询移到业务层绕过。自动覆盖的接口定义如需扩展，从实际模板修正并重新生成，不直接手改生成文件。
- 例如已有 `user_repository.go` 和 `user_repository_gen.go` 时，新增自定义查询方法应写入 `user_repository.go`，不得另建 `user_repository_query.go` 或 `user_repository_extra.go`；实际文件名与类型以项目生成结果为准。

## 模板复用与扩展判定

- 先读取当前表的非生成文件与 `_gen.go`，沿接口嵌入和结构体嵌入确认完整方法集。例如 `AttributeRepository` 嵌入 `iDefaultAttributeRepository`、实现结构体嵌入 `*defaultAttributeRepository` 时，默认方法已可用，不要把这些方法逐个再声明或实现一遍；名称与嵌入方式以实际模板为准。
- 标准 CRUD、批量、分页和事务方法能满足需求时，业务层直接调用模板方法；筛选、指定字段、缓存等先检查已有 Wrapper 和选项，不因需要传入不同条件就新增方法。
- HTTP 接口叫 `CreateAttribute`、`DeleteAttribute` 或 `RenameAttribute`，不代表 Repository 也必须新增同名方法。若分别只是创建一条记录、按 ID 删除或更新名称字段，应复用实际存在的 `Create`、`DeleteById`、`UpdateById`/`UpdateMapById` 等方法；调用参数和更新选项按真实签名填写。
- 请求 DTO/Req 到实体、更新字段或查询参数的适配，优先放在项目已有 Mapper 或 Logic 中；不为让 Repository 直接接收某个 HTTP 请求类型，重复增加一套 CRUD 方法。不得通过在业务层直接写数据库查询来替代 Repository 复用。
- 禁止仅改名、改参数包装或简单转调模板方法的重复封装；保留模板已生成的公开接口、默认接口、实现嵌入和构造函数，不另建一套接口或实现。
- 需要扩展时，先明确“已检查哪些模板方法/选项，缺少什么数据访问能力”。仅对确实无法表达的联表、聚合或事务内数据读写，在对应的已有非生成文件中扩展；内部仍尽量调用已有方法。业务规则和多表业务流程放在 Logic/Service，不能以“专用业务操作”为由加入 Repository。
- 模板的“可扩展/可覆盖”注释只是扩展能力说明，不是要求新增或覆盖。仅在确有持久化技术需求且不破坏原契约时才覆盖已有方法，禁止借此注入业务规则；不要为了统一命名或格式覆盖默认实现。

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

标准分页优先走生成 Repository。特殊联表先检查生成产物是否已有 `FindXxxJoinPage` 等辅助方法；确实不足再在当前表对应的已有非生成 Repository 文件中增加方法，不能新建分页或联表文件。

分页查询严格使用 `ScanByPage` 或 `FindByPage`：实体列表通常使用 `FindByPage`，自定义字段投影或联表结果通常使用 `ScanByPage`，按实际查询对象的方法签名调用。不用 `Count` 加 `Offset`/`Limit`/`Find`/`Scan` 拼出另一套分页逻辑。

分页参数统一写为：

```go
offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)
```

遇到 `int((req.PageNumber-1)*req.PageSize), int(req.PageSize)` 或其他手算 offset/limit 的写法时替换为上述调用。基线 `DbPage` 将页码 0 变为 1，将非正 pageSize 变为 10，大于 500 变为 500；它没有完整拒绝负页码和溢出输入，仍需按接口契约验证输入。

基线生成模板的普通 `FindPage`/`FindPageByWrapper` 直接计算 offset/limit，并未调用 `DbPage`。使用这些 Repository 方法前核对内部实现；不符合本规则时修正实际生成模板并重新生成，或在对应的已有非生成 Repository 文件中新增方法实现合规分页。不要直接修改 `_gen.go` 后留下无法复现的产物，也不能把 Repository 的方法名当作内部已经合规的证据。

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
    PageBaseReq // 通用分页请求参数。
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
