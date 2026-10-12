# 默认接口开发流程

本流程用于使用 `github.com/kuangshp/gorm-plus` 的项目中新建或修改业务接口，包括已有表上的接口。先检查依赖、配置和已有生成产物，再按以下顺序执行；用户当前指令优先。纯只读审查、故障定位或无关工具函数修改不执行数据库写入与重生成。

## 1. 准备根目录 generator.go

- 根目录指当前服务所属 Go module 的根目录；多模块仓库先确认目标服务。文件已存在时读取并复用，缺失时创建。
- 调用实际版本支持的 `generator.Generate` 与 `generator.Config`。包名取自 `go.mod`，输出路径遵循项目布局；项目无约定时可使用 `query/dao`、`query/model`、`query/repository`、`query/mapper`、`query/apis`。
- 直接复用项目配置类型、配置加载方式和数据库配置映射。若受 Go `internal` 包可见性限制，使用同一配置源的最小映射类型，不复制第二套数据库配置。
- 提供明确配置文件选择方式（沿用项目参数/环境变量），未指定且无项目默认配置路径时明确报错。配置加载失败或缺少必要连接字段时停止，禁止回退到硬编码主机、账号、密码或数据库。
- 生成入口与 SQL 执行工具使用相同配置文件、环境及数据库覆盖规则；库名或环境覆盖必须显式指定，避免 SQL 同步和生成连接到不同库。
- 根目录已有服务 `main` 或其他包时，按项目方式隔离生成入口；可在文件顶部添加 `//go:build ignore` 及空行，使用显式文件命令 `go run generator.go`，避免进入常规服务构建。
- 生成失败返回非零退出码。只生成目标表，查清当前版本表选择和交互机制后传入；不能假定命令行表名会自动生效。

用户提供的示例体现如下配置映射方式；这是入口内部片段，`databaseConfig` 必须先由项目配置加载，字段和签名按实际依赖核对：

```go
err := generator.Generate(&generator.Config{
    DBType:        "mysql",
    Host:          databaseConfig.MysqlConfig.Host,
    Port:          databaseConfig.MysqlConfig.Port,
    Username:      databaseConfig.MysqlConfig.Username,
    Password:      databaseConfig.MysqlConfig.Password,
    Database:      databaseConfig.MysqlConfig.Database,
    OutPath:       "./query/dao",
    ModelPkgPath:  "./query/model",
    RepoPath:      "./query/repository",
    APIMapperPath: "./query/mapper",
    ApiPath:       "./query/apis",
    Package:       modulePath,
})
```

示例中的 go-zero `conf.Load` 只在 go-zero 项目适用；其他项目沿用自身配置加载器。不要照抄示例的 `kg-api`、排除表、默认密码、业务 Mapper 修补、排序规则、DAO 注册表锚点或中间目录删除。goctl 路径兼容与单表注册保护仅在当前版本确有相同问题时采用，并确保失败时恢复原文件和环境。

## 2. 先创建日期 SQL 文件

- 在编写 Handler、Logic 或业务服务代码前，先创建本次 SQL 文件。路径为 `sql/YYYYMMDD描述.sql`，SQL 文件直接放在 `sql` 目录下，不创建年月日子目录。日期使用用户时区的当天日期；描述根据当前表的业务性质及本次操作确定，例如 `sql/20261013创建用户表.sql`、`sql/20261013更新用户表.sql`。同日同表有多次变更时，用更具体的描述区分，例如 `sql/20261013更新用户表增加手机号字段.sql`；有依赖的 SQL 按依赖顺序执行，不依赖描述的字典序。
- 新表写建表 SQL，字段或索引变化写增量 SQL，必要初始化数据一并记录。按当前业务需求确定表名、主键、字段、索引和注释。
- 已有表且本次无结构变化，也创建 SQL 文件，说明复用的表和无 DDL 变更，并加入项目可执行的结构核对 SQL（如 `SHOW CREATE TABLE`，依数据库类型适配）。执行核对而非重复建表；不能用 `CREATE TABLE IF NOT EXISTS` 当作字段一致性的证明。
- SQL 是本次结构变更的依据。不能用临时手改数据库、`AutoMigrate` 或先手写 Model/Repository 代替 SQL 文件。

## 3. 使用项目配置同步数据库

- 优先使用项目已有 SQL 执行脚本或迁移工具，使其从与生成器相同的项目配置读取连接信息。没有工具时补充最小执行入口或使用当前可用数据库客户端，不能另写硬编码连接。
- 先确认配置指向本次开发目标，按文件顺序应用本次尚未执行的 SQL；沿用项目迁移记录，禁止默认执行整个历史 SQL 目录。密码不写入命令行参数、日志或交付说明。
- 同步后查询实际表结构，核对字段、主键、索引、默认值与 SQL。无 DDL 变更时，执行文件中的结构核对并确认已有表可供生成。
- 同步失败先定位原因；DDL 可能部分成功，检查真实数据库状态再修复，不盲目重跑或删除重建。没有配置、连接不可用或权限不足时保留 SQL 与入口，说明具体阻碍，不能宣称已同步。
- 当前接口开发请求按这套流程授权必要的开发数据库同步；生产环境或超出任务范围的破坏性操作不能仅凭 Skill 自动执行。

## 4. 同步成功后运行 generator.go

- 从模块根目录执行 `go run generator.go`，配置参数、环境变量及表选择方式采用入口真实实现。不要把业务服务启动命令当作生成命令。
- 必须连接已同步并核对的同一目标数据库。执行前阅读 [生成器](generator.md) 的覆盖规则，保护自定义扩展和已有 DAO 注册。
- 检查目标表对应的 Model、DAO、Repository 及按项目需要输出的 API/Mapper，核对实际文件差异；命令退出成功不代表每个产物都成功。
- 生成失败不能手写等价生成文件绕过；修正配置、输入或模板后重生成。已有 API 被跳过时，按项目策略更新契约并生成下游文件。

## 5. 注入 Repository 再实现接口

- 在应用启动或依赖装配入口（如 go-zero `ServiceContext`、Gin 应用容器、Wire/Fx provider 或服务构造函数）创建 Repository，通过接口类型注入业务层。
- Repository 构造函数和参数以实际生成文件为准；复用启动时初始化的数据库及项目要求的缓存、事务和插件上下文。不要编造统一构造函数签名。
- Logic/Service 从注入字段或构造参数取得 Repository，传递请求 Context；禁止在每个请求中临时创建 Repository、连接数据库，或用全局变量代替注入。
- 数据库查询尽量写在 Repository 中，不要直接在 Handler、Logic、Service 等业务代码中编写 SQL 或 DAO/GORM 查询链。标准查询复用生成 Repository 方法；复杂筛选、联表、聚合及原生 SQL 查询封装在项目规定的非生成 Repository 扩展文件中，通过 Repository 接口暴露并由业务层注入调用。业务层负责业务校验与编排，不直接绕过 Repository 访问数据库。
- 按业务契约实现校验、映射和编排，验证注入链、生成产物编译及本次相关行为。

## 交付证据

列出根目录入口、日期 SQL 文件、配置来源（不含密钥）、实际同步与结构核对结果、生成命令与目标产物、Repository 注入位置及相关验证结果。未连接数据库或未运行生成器时，明确标为未完成，保留可继续执行的文件和步骤。
