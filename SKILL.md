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

- 标准操作优先复用项目已有生成 Repository 方法，先检查签名和行为，再决定是否扩展。
- 数据库查询尽量封装在 Repository 中，不要直接在 Handler、Logic、Service 等业务代码中编写数据库查询。业务层通过注入的 Repository 调用；标准查询优先复用生成方法，复杂筛选、联表、聚合及原生 SQL 查询放在项目规定的非生成 Repository 扩展文件中。库支持原生 GORM 或 DAL，不代表项目允许业务层绕过 Repository。
- 分页查询严格使用 `ScanByPage` 或 `FindByPage`，分页参数统一使用 `offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)`，不要手算 offset/limit；生成 Repository 方法也要核对内部实现。
- 范围筛选使用 `BetweenIfNotZero`；检查零值和单边输入语义，不能让合法筛选条件静默消失。
- DAO 实体和字段直接使用完整名称（如 `dao.FinishedGoodsPurchaseOrderEntity.ArrivalDate`），不定义 `order := dao.FinishedGoodsPurchaseOrderEntity`、`supplier := dao.SupplierEntity` 等局部实体别名。SQL 列别名 `.As("SupplierName")` 可正常使用。
- 所有请求（Req）中的时间参数使用 `string` 类型，并添加 `dateTime` 校验；沿用项目校验标签和注册机制，更新契约、生成 Types 及解析逻辑。详细示例见 [Repository 与查询](references/repository-query.md)。
- 生成器能够表达的标准代码从输入、配置或模板修正并重新生成。生成失败时排查原因，不手写等价生成文件。
- 根据页面和业务需求决定接口范围，不因生成器支持 CRUD 就自动暴露所有路由。
- 业务扩展遵循当前项目的依赖注入、Mapper、枚举、时间处理和格式要求，不将某个项目的绝对路径写成库的通用约定。
- 不猜测方法名、返回值、空值语义和覆盖行为；当前源码与资料不一致时，以实际实现和验证结果为准，并说明差异。
- 保留事务、租户和权限上下文；不能通过跳过租户、禁用插件或换成后台 Context 来让失败操作通过。
- 接口开发中的必要 SQL 同步和目标表生成属于上述默认流程，按任务已有授权及项目环境执行。Skill 本身不授权操作生产库、破坏性迁移、升级库、生成全表或安装其他工具。纯审查、排查、普通工具函数修改不为套用流程而创建 SQL 或写数据库。

## 完成检查

- 接口开发具备根目录生成入口、日期 SQL 文件、配置来源、数据库同步/结构核对结果、目标表生成结果及 Repository 注入；受环境阻碍的步骤明确标为未完成，不能跳过后宣称完成。
- 分页使用 `DbPage` 与 `ScanByPage`/`FindByPage`，范围使用 `BetweenIfNotZero`，DAO 无局部实体别名，请求时间参数为字符串且 `dateTime` 校验实际生效。
- 实现符合当前实际方法签名；错误分支和返回值已处理。
- 测试与修改相关：分页边界、范围条件、事务回滚、插件作用范围按需验证。
- 生成任务检查输出清单、差异和项目要求的干净目录复现；不能用“命令执行过”替代结果验证。
- 执行项目要求的格式与测试命令，仅检查本次范围。没有修改 Go 时不运行无关全仓格式化。
- 交付说明改了什么、实际验证了什么、哪些内容未验证。
