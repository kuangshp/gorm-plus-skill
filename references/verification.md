# 来源与验证场景

## 核对基线

- 上游：https://github.com/kuangshp/gorm-plus
- 核对日期：2026-09-28
- 核对提交：`8735474c4ff11b1c061d1e8373de7a3b49755d97`
- 制作时读取了本机该提交的干净源码工作区；不要求使用者拥有相同的本地路径。
- 此提交是资料基线，不是最低支持版本，也不表示项目应升级。

优先在当前项目实际解析的模块源码中检查以下文件；无法获取时再查询对应 tag/commit 的上游源码。缺少源码和可用签名时说明未验证，不把最新 README 当作已安装版本的证据。

| 内容 | 源码 |
| --- | --- |
| 覆盖与跳过策略 | `generator/generator.go` |
| 生成配置 | `generator/config.go` |
| Repository 签名、事务方法、缓存时序 | `generator/template/repository_gen_template.txt` |
| 自定义扩展结构 | `generator/template/repository_template.txt` |
| 条件和 LIKE 的准确语义 | `query/gen_wrapper.go` |
| 页码转换 | `gormplus_page.go` |
| 事务封装 | `gormplus_transaction.go` |
| 缓存参数 | `query/query_option.go`、`query/query_executor.go` |
| 插件与 Context | `plugin/tenant.go`、`plugin/autoOperator.go`、`plugin/ctx.go` |

可使用固定提交链接，例如：
https://github.com/kuangshp/gorm-plus/blob/8735474c4ff11b1c061d1e8373de7a3b49755d97/query/gen_wrapper.go

## 维护与行为验证

修改 Skill 后检查 Markdown 相对链接和 YAML 元数据；存在 skill-creator 时运行其 `scripts/quick_validate.py`。结构校验通过不等于以下行为已经通过。

在独立测试项目中按实际需要执行以下任务，记录输入、改动、运行结果和未验证项：

| 测试请求 | 观察结果 |
| --- | --- |
| gorm-plus 项目新增订单接口，尚无生成入口 | 先准备根目录 generator.go，连接取自项目配置；创建日期 SQL，同步并核对后生成，最后注入 Repository |
| 已有表增加只读接口，无字段变化 | 仍先创建日期 SQL，执行结构核对，不重复建表，再运行目标表生成并注入 Repository |
| 只写建表 SQL、数据库里尚无目标表就请求生成 | 先执行 SQL 创建真实表并核对结构，成功后运行生成器生成 Model/DAO/Repository；不得提前生成或手写替代 |
| SQL 同步失败或配置缺失 | 保留入口与 SQL，定位阻碍，不连接硬编码示例库，不手写生成产物，不宣称流程完成 |
| SQL 工具与生成器选择不同配置环境 | 执行前统一目标配置，核对同步和生成的数据库一致 |
| 已有 Repository 需要复杂查询或事务内数据读写 | 先复用模板能力，仅缺失的纯数据操作在对应的已有非生成文件中新增方法；业务判断和跨 Repository 事务编排在 Logic/Service，不新建 Repository 文件或修改 `_gen.go` |
| 模板已提供 CRUD，HTTP 接口叫 CreateAttribute、DeleteAttribute、RenameAttribute | 核对并直接使用已有创建、按 ID 删除和字段更新方法，不为对齐 HTTP 名称另定义一套 Repository 方法 |
| AttributeRepository 已嵌入默认接口，实现已嵌入默认 Repository | 识别继承的方法集，保留嵌入及构造函数，不重新声明默认方法或创建第二套实现 |
| 只为接收 Req 而新增一行转调 Create 的方法 | 将参数适配留在现有 Mapper/Logic，直接调用模板方法；确需新增数据访问操作时说明实际能力缺口 |
| query/model 缺少新表实体，或实体缺少字段 | 从日期 SQL/真实表结构与生成配置定位并重生成，不手写实体、不手改现有文件 |
| DAO 缺少查询文件、Do 接口或 gen.go 注册 | 核对真实 DAO 输出目录，从数据库结构、生成输入或模板修复并重生成，不手写、手改或新增辅助文件补齐 |
| 需要给生成 Model 加标签、方法或 extra.go | 不直接编辑或新建文件；标签等生成需求调整配置/模板，业务方法放业务层，不在 Model 目录建扩展文件 |
| ModelPkgPath 指向其他目录，或文件没有 .gen.go 后缀 | 按真实 Model 输出目录应用相同禁写规则，不以路径名称或文件后缀绕过 |
| 对应的非生成 Repository 文件缺失 | 修复配置或模板并运行目标表生成；生成失败时不手写空壳或另建文件绕过 |
| go-zero 新增 ListOrdersLogic | 声明 purchaseOrderRepository repository.PurchaseOrderRepository，在 NewListOrdersLogic 的结构体字面量中用完整构造函数初始化；保留上下文、日志、请求与响应字段，不改成从 ServiceContext 取 Repository |
| Gin 控制器或 Logic 使用 Repository | 使用相同的完整接口字段和构造函数内初始化方式，保持 Gin 控制器接口与目录布局，不引入 go-zero 专有上下文类型 |
| Repository 字段写为 repo、poRepo 或 purchaseOrderRepo | 改为 purchaseOrderRepository，调用直接使用完整字段，不另设局部或包别名 |
| 单数据库构造 Repository | 实际签名支持默认连接时允许无参，也允许显式传连接；不编造构造函数签名 |
| 多数据库尚未填写 Repository 连接参数 | 保留清晰 TODO，由用户填写，不猜选库、不填 nil、不擅自添加回退逻辑；未完成编译验证则如实说明 |
| 给列表增加名称包含筛选 | 使用包含匹配，空值处理符合接口约定 |
| 时间范围只提供起点 | 使用 BetweenIfNotZero 前校验成对输入，不静默丢失范围筛选 |
| 查询金额从 0 到 100 | 不把合法的 0 当作未提供导致漏筛选 |
| 分页传负页码或超大 pageSize | 检查输入校验及真实实现，分页参数经 DbPage，查询使用 ScanByPage/FindByPage |
| 发现手算 offset/limit | 改为 offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)，生成代码从模板修正 |
| 联表查询使用 order/supplier 局部实体别名 | 改为完整 dao 实体和字段名称，保留必要 SQL 输出列别名 |
| Req 时间字段为 int64 秒级时间戳 | 改为 string，更新源契约和生成 Types，dateTime 校验对合法、非法及可选空值生效 |
| Repository 重写 Create 校验末级分类、必选属性与存量商品限制 | 将查询后的业务判断、业务错误和流程迁到 Logic/Service；保留模板 Create，Repository 只提供所需数据 |
| 将业务校验提取成 Repository 私有函数或塞入模板 | 仍判定为越层，迁到已有业务层，不能通过辅助函数或模板隐藏业务规则 |
| 业务校验与多表写入需要原子性 | Logic/Service 在同一事务中通过事务绑定的 Repository 读取、判断、写入；不将校验移到事务外，不在业务层写 DAO/GORM 查询链 |
| 主表写入后明细失败 | Logic/Service 编排事务并返回错误，主表回滚，没有调用事务外写入方法 |
| 新字段需要更新已有 API | 识别“已有文件跳过”，从生成输入/模板及项目策略处理 |
| go-zero 新增用户接口 | 优先生成并复制目标 .api 到服务 desc 后裁剪，接入聚合入口并校验、生成下游代码；不未经检查直接手写 |
| 生成器没有产出 .api 或执行失败 | 先核对输入、配置、模板与失败原因，合理修复可解决时重生成；确认无法取得可用产物时允许手写 .api，说明原因，不伪造生成来源 |
| 生成器不支持所需业务契约 | 能复用的部分先复制，确认能力缺口后允许手写缺失 .api，说明范围并执行同样的 API 校验及下游生成 |
| desc 已有裁剪后的契约，本次增加字段 | 将最新生成源复制到独立暂存位置，基于副本裁剪、比较并合并本次变化，不覆盖已有业务契约 |
| 模板没有所需方法 | 核对版本、已有替代能力，不编造方法或擅自升级 |
| 两个租户同样筛选且启用缓存 | 核对缓存键的租户与权限维度，而不只检查 SQL |
| 已依赖 go-utils，需要映射、筛选、分组或去重 | 先查实际版本并复用可满足的工具方法，不重复定义方法或写等价循环 |
| 已检查 go-utils 但确实无法满足需求 | 说明候选及具体缺口后，才按职责新增方法或使用循环，不擅自升级依赖 |
| 项目未依赖 go-utils | 不为技能约束新增依赖，使用项目现有能力或标准库 |
| 新字段、接口方法或方法实现缺少右侧备注 | 在字段/签名/定义右侧补充准确含义或用途，多行定义在最后一行标注 |
| 生成的实体字段或方法没有说明 | 从输入或实际模板补全再生成，不手改 Model/DAO/_gen.go，也不另建文件绕过 |
| 只修改普通 Go 字符串工具 | 不引入数据库生成、迁移或插件配置 |

不用每次加载 Skill 都执行全套测试。普通任务只验证被修改的行为；维护 Skill 时用这些场景检查它是否会选对方法和保持任务范围。
