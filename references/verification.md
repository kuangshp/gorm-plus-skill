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
| SQL 同步失败或配置缺失 | 保留入口与 SQL，定位阻碍，不连接硬编码示例库，不手写生成产物，不宣称流程完成 |
| SQL 工具与生成器选择不同配置环境 | 执行前统一目标配置，核对同步和生成的数据库一致 |
| 给列表增加名称包含筛选 | 使用包含匹配，空值处理符合接口约定 |
| 时间范围只提供起点 | 使用 BetweenIfNotZero 前校验成对输入，不静默丢失范围筛选 |
| 查询金额从 0 到 100 | 不把合法的 0 当作未提供导致漏筛选 |
| 分页传负页码或超大 pageSize | 检查输入校验及真实实现，分页参数经 DbPage，查询使用 ScanByPage/FindByPage |
| 发现手算 offset/limit | 改为 offset, limit := gormplus.DbPage(req.PageNumber, req.PageSize)，生成代码从模板修正 |
| 联表查询使用 order/supplier 局部实体别名 | 改为完整 dao 实体和字段名称，保留必要 SQL 输出列别名 |
| Req 时间字段为 int64 秒级时间戳 | 改为 string，更新源契约和生成 Types，dateTime 校验对合法、非法及可选空值生效 |
| 主表写入后明细失败 | 主表回滚，没有调用事务外写入方法 |
| 新字段需要更新已有 API | 识别“已有文件跳过”，从生成输入/模板及项目策略处理 |
| 模板没有所需方法 | 核对版本、已有替代能力，不编造方法或擅自升级 |
| 两个租户同样筛选且启用缓存 | 核对缓存键的租户与权限维度，而不只检查 SQL |
| 只修改普通 Go 字符串工具 | 不引入数据库生成、迁移或插件配置 |

不用每次加载 Skill 都执行全套测试。普通任务只验证被修改的行为；维护 Skill 时用这些场景检查它是否会选对方法和保持任务范围。
