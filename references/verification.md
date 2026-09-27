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
| 给列表增加名称包含筛选 | 使用包含匹配，空值处理符合接口约定 |
| 时间范围只提供起点 | 发现范围会整体跳过，按契约处理单边条件或校验 |
| 查询金额从 0 到 100 | 不把合法的 0 当作未提供导致漏筛选 |
| 分页传负页码或超大 pageSize | 检查真实入口，不假定 Repository 自动调用 DbPage |
| 主表写入后明细失败 | 主表回滚，没有调用事务外写入方法 |
| 新字段需要更新已有 API | 识别“已有文件跳过”，从生成输入/模板及项目策略处理 |
| 模板没有所需方法 | 核对版本、已有替代能力，不编造方法或擅自升级 |
| 两个租户同样筛选且启用缓存 | 核对缓存键的租户与权限维度，而不只检查 SQL |
| 只修改普通 Go 字符串工具 | 不引入数据库生成、迁移或插件配置 |

不用每次加载 Skill 都执行全套测试。普通任务只验证被修改的行为；维护 Skill 时用这些场景检查它是否会选对方法和保持任务范围。
