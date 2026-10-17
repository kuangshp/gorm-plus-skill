---
name: gorm-plus-skill
description: 在使用 github.com/kuangshp/gorm-plus 的 Go 项目中，默认按根目录 generator.go、按 YYYYMMDD描述.sql 命名 SQL、项目配置同步数据库、生成代码、注入 Repository 的流程开发接口，并支持查询、分页、事务和上下文问题的审查排查。用户提及 gorm-plus，或项目依赖及待修改代码已使用该库时使用；普通 Go 问题及未使用该库的 GORM 项目无需启用。
---

# gorm-plus 开发指南

帮助使用者复用库与项目现有能力，按实际依赖版本完成实现和验证。项目的业务需求、生成流程和分层约定以当前用户指令及适用的 AGENTS.md 为准。

## 先确认环境

1. 阅读适用的项目说明，检查工作区已有改动，确定本次范围。
2. 检查 `go.mod`、`go.work`、`replace`、vendor 和项目构建参数，确认实际使用的 gorm-plus 源码。必要时运行 `go list -m -json github.com/kuangshp/gorm-plus`，同时检查返回的 Replace；不要为了查版本执行 `go get -u`。
3. 阅读当前实体的 Repository 接口、实现及相关调用。生成文件可能来自旧版本或本地模板，不能只看依赖版本推断其能力。
4. 只读取当前任务需要的下列参考资料。资料中的基线用于定位源码，不代表项目必须升级到该版本。

## 按任务选择资料

| 当前任务 | 阅读 |
| --- | --- |
| 新建或修改业务接口（包括使用已有表） | [默认接口开发流程](references/interface-workflow.md)，再读生成器及本次查询所需资料 |
| 新表、字段变化、生成配置、生成结果异常 | [生成器](references/generator.md) |
| 单条/列表/分页、可选筛选、联表、缓存参数 | [Repository 与查询](references/repository-query.md) |
| 多表写入、回滚、事务内读取 | [事务](references/transactions.md) |
| 租户条件、操作人字段、Context 丢失 | [插件与上下文](references/plugins-context.md) |
| 维护本 Skill、复核版本或验证行为 | [来源与验证场景](references/verification.md) |

## 执行原则

- 在 gorm-plus 项目开发业务接口时，默认严格按顺序执行：准备根目录 `generator.go` → 创建 `sql/YYYYMMDD描述.sql` 文件（描述根据当前表及操作确定，如创建用户表、更新用户表） → 使用项目配置同步目标数据库并核对结构 → 运行 `generator.go` 生成对应文件 → 注入 Repository 后实现接口。详细规则见 [默认接口开发流程](references/interface-workflow.md)。SQL 文件直接放在 `sql` 目录下，不创建年月日子目录。
- 已有根目录生成入口先复用和调整，不重复创建；数据库连接必须来自项目配置，不硬编码连接信息或回退到示例数据库。接口开发即使使用已有表，也要先留下 SQL 文件并核对数据库状态；无结构变化时不要重复执行建表或破坏性 DDL。

- **Model 目录由生成器独占（强制）**：`query/model` 及其子目录中的所有内容禁止手写、手动修改或手动新建文件；该规则同样适用于项目通过 `ModelPkgPath` 等配置指定的实际 Model 输出目录，不仅限于 `.gen.go` 后缀。新增实体、字段、类型、标签、注释及映射调整，必须从 SQL/表结构、生成配置或实际模板修正后运行项目生成器产生，禁止复制实体、追加方法、另建扩展文件或用脚本直接写入来绕过。生成失败时修复原因，不能手写补齐。详见[生成器](references/generator.md)。
- **Repository 文件约束（强制）**：禁止在 `repository` 目录及其子目录中手动新建文件，也不能通过复制、拆分或另建扩展文件承载新增功能。Repository 新文件只能由项目生成器按目标表生成。新增且无法由模板满足的数据访问能力只能在对应表或领域已有的非生成 Repository 文件中增加方法，并同步维护对应的已有接口定义；禁止将自定义方法写入 `_gen.go`。对应文件缺失时，先修复配置或模板并运行生成器，不能手写补齐。具体落点见 [Repository 与查询](references/repository-query.md)。
- **Repository 使用方式（Gin 与 go-zero 统一）**：业务结构体声明完整名称的接口字段，如 `purchaseOrderRepository repository.PurchaseOrderRepository`，在该结构体的 `NewXxx(...)` 构造函数返回值中直接用 `repository.NewPurchaseOrderRepository(...)` 初始化；调用使用 `l.purchaseOrderRepository` 等完整字段名，禁止缩写为 `repo`、`poRepo`、`purchaseOrderRepo` 或再定义局部别名。不要改成统一在 `ServiceContext`/启动容器中创建 Repository 后转取。单数据库且实际构造函数支持默认连接时，可写 `repository.NewPurchaseOrderRepository()`，不强制传 `svcCtx.DB`；保留显式传参也允许。多数据库的连接参数由用户自行填写，不擅自选择数据库。完整示例与参数规则见 [默认接口开发流程](references/interface-workflow.md)。
- **模板能力优先复用（强制）**：先检查对应 `_gen.go`、默认 Repository 接口及其嵌入提供的实际能力。创建、删除、更新、详情、列表、分页、批量操作及事务版本已有可用方法时直接调用，禁止仅为接口命名、接收 Req/DTO 或转调一行生成方法，再定义一套 `CreateXxx`、`DeleteXxx`、`RenameXxx` 等等价方法。保留模板已有的接口嵌入、实现嵌入与构造函数，不重复定义默认接口、实现结构体或 CRUD。只有明确核对已有方法、Wrapper 和选项仍无法表达所需数据访问行为时，才在对应的已有非生成文件中新增方法，并说明缺口；详细判定见 [Repository 与查询](references/repository-query.md)。
- **业务代码禁止放入 Repository（强制，Gin 与 go-zero 一致）**：Repository 只负责数据查询、持久化及其必要的事务绑定、上下文与缓存处理；业务校验、业务状态/权限判断、计算、错误文案和跨 Repository 流程编排放在 Logic/Service 或项目已有业务层。禁止为了增加业务校验而重写模板 `Create`/`Update`/`Delete`，也不能将业务规则转移到 Repository 辅助函数、模板或生成代码中。业务层通过 Repository 获取数据并做决定；事务内校验与写入仍须使用同一事务绑定的 Repository，不能因分层拆开事务。详细边界见 [Repository 与查询](references/repository-query.md)。
- 数据库查询尽量封装在 Repository 中，不要直接在 Handler、Logic、Service 等业务代码中编写数据库查询。业务层通过注入的 Repository 调用；标准查询优先复用生成方法，复杂筛选、联表、聚合及原生 SQL 查询以新增方法的形式写入对应的已有非生成 Repository 文件，禁止为此新建文件。库支持原生 GORM 或 DAL，不代表项目允许业务层绕过 Repository。
- 分页查询严格使用 `ScanByPage` 或 `FindByPage`，分页参数统一使用 `offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)`，不要手算 offset/limit；生成 Repository 方法也要核对内部实现。
- 范围筛选使用 `BetweenIfNotZero`；检查零值和单边输入语义，不能让合法筛选条件静默消失。
- DAO 实体和字段直接使用完整名称（如 `dao.FinishedGoodsPurchaseOrderEntity.ArrivalDate`），不定义 `order := dao.FinishedGoodsPurchaseOrderEntity`、`supplier := dao.SupplierEntity` 等局部实体别名。SQL 列别名 `.As("SupplierName")` 可正常使用。
- 所有请求（Req）中的时间参数使用 `string` 类型，并添加 `dateTime` 校验；沿用项目校验标签和注册机制，更新契约、生成 Types 及解析逻辑。详细示例见 [Repository 与查询](references/repository-query.md)。
- 生成器能够表达的标准代码从输入、配置或模板修正并重新生成。生成失败时排查原因，不手写等价生成文件。
- 根据页面和业务需求决定接口范围，不因生成器支持 CRUD 就自动暴露所有路由。
- Repository 初始化与命名遵循上述统一方式；其他业务扩展遵循当前项目的 Mapper、枚举、时间处理和格式要求，不将某个项目的绝对路径写成库的通用约定。
- 不猜测方法名、返回值、空值语义和覆盖行为；当前源码与资料不一致时，以实际实现和验证结果为准，并说明差异。
- 保留事务、租户和权限上下文；不能通过跳过租户、禁用插件或换成后台 Context 来让失败操作通过。
- 接口开发中的必要 SQL 同步和目标表生成属于上述默认流程，按任务已有授权及项目环境执行。Skill 本身不授权操作生产库、破坏性迁移、升级库、生成全表或安装其他工具。纯审查、排查、普通工具函数修改不为套用流程而创建 SQL 或写数据库。

## 完成检查

- 接口开发具备根目录生成入口、日期 SQL 文件、配置来源、数据库同步/结构核对结果、目标表生成结果及 Repository 注入；受环境阻碍的步骤明确标为未完成，不能跳过后宣称完成。
- 分页使用 `DbPage` 与 `ScanByPage`/`FindByPage`，范围使用 `BetweenIfNotZero`，DAO 无局部实体别名，请求时间参数为字符串且 `dateTime` 校验实际生效。
- Model 输出目录无手写、手动修改或手动新增文件；新增及变更均可追溯到本次生成输入、配置/模板与实际生成结果，未以其他文件后缀或扩展文件绕过。
- Repository 无手动新增文件；所有新增文件均可追溯到本次目标表的生成结果。自定义方法位于对应的已有非生成文件，接口定义同步更新，未手改 `_gen.go`。
- Gin 与 go-zero 的 Repository 均使用完整接口字段名，在业务对象的构造函数中直接初始化；无 Repository 缩写或局部别名。单库省略连接参数时已核对真实签名及默认连接，多库参数未代用户猜填。
- 新增 Repository 方法均有模板方法、Wrapper 和选项无法满足的具体理由；没有改名重复 CRUD、仅转调的包装方法或重复的默认接口/实现，模板嵌入结构得到保留。
- Repository 不含业务规则、业务错误文案或业务流程编排，未为业务校验覆盖模板 CRUD；Logic/Service 承担业务判断，事务内读取与写入仍共享同一事务。
- 实现符合当前实际方法签名；错误分支和返回值已处理。
- 测试与修改相关：分页边界、范围条件、事务回滚、插件作用范围按需验证。
- 生成任务检查输出清单、差异和项目要求的干净目录复现；不能用“命令执行过”替代结果验证。
- 执行项目要求的格式与测试命令，仅检查本次范围。没有修改 Go 时不运行无关全仓格式化。
- 交付说明改了什么、实际验证了什么、哪些内容未验证。
