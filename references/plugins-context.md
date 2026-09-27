# 插件与上下文

## 自动填充不是无条件成立

使用库不代表已注册所有插件。检查实际启动代码及配置，核对租户字段、排除表、操作人字段、创建/更新触发条件和 Getter。

`company_id`、`created_id`、`updated_id` 是项目可能采用的字段名，不是所有项目都相同。项目已正确配置自动处理的字段不重复手填；未配置时先定位接入方式，不虚构自动填充行为。

## 租户或操作人异常的排查顺序

1. 确认发起操作的数据库实例确实注册了插件，特别是多数据源和测试连接。
2. 检查中间件写入的 Context key、值类型与插件 Getter/解析器是否匹配。
3. 沿 Handler → Logic → Repository → DAO 跟踪 Context；留意中途换成 `context.Background()` 的位置。
4. 检查表是否在排除列表、字段是否存在、关联表/别名规则是否符合配置。
5. 核对实际 SQL 与返回结果；日志不要泄露凭据和敏感数据。
6. 修复根因后用隔离的不同租户数据回归，验证每个租户只看见允许的数据。

基线的默认 Context 解析器处理标准 context；当框架包装 Context 与请求 Context 不同，需要按真实调用方式检查 `RegisterCtxResolver`。不要仅凭框架名字就重复注册或省略检查。

遇到缺少租户信息时，实际行为以当前插件配置和实现为准；不能统一承诺“必然报错”或“必然过滤”。不要通过 SkipTenant、移除过滤或禁用权限插件来掩盖问题。

租户 SQL 条件和缓存隔离是两件需要分别检查的事；涉及缓存时同时阅读 [Repository 与查询](repository-query.md)。

源码入口：`gormplus_tenant.go`、`gormplus_autofill.go`、`plugin/tenant.go`、`plugin/autoOperator.go`、`plugin/ctx.go`。来源和核对基线见 [来源与验证场景](verification.md)。
