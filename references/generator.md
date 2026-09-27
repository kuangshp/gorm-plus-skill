# 代码生成

## 先检查，后执行

读取项目的生成入口（例如 generator.go）、Makefile/脚本、配置和本地模板，确认：

- 使用哪个数据库与 schema；不要把密码或连接串抄入日志和交付资料。
- 当前表名、主键和所需业务接口；数据库结构是否已按项目流程同步。
- DAO、Model、Repository、API、Mapper 等实际输出位置。
- 模板来自项目本地还是库内嵌；goctl 的模板来源也需要单独核对。
- 所有交互输入的含义及空输入的行为；只生成已确认的目标表。

库的入口可从 `gormplus_generator.go` 追到 `generator/config.go` 与 `generator/generator.go`。项目已经有生成入口时优先使用项目入口，不额外搭建另一套。

## 覆盖行为必须核对源码

基线实现中：

| 产物 | 已有文件的处理 |
| --- | --- |
| Repository `_gen.go` | `writeFileAlways`，覆盖 |
| Repository 自定义扩展 `.go` | `writeFileIfNotExist`，保留已有文件 |
| 表对应 API `.api` | 已存在时跳过 |

DAO/Model 由 gorm-gen 路径生成，其他产物按各自分支处理，必须检查实际输出和差异。不要根据 README 中概括的“Repository 不覆盖”推断 `_gen.go` 也会保留。

字段变化后，重新运行生成器不代表已存在的 API、扩展文件、Handler 和 Logic 都已更新。goctl 对已有文件的行为也应按当前版本核实。

## 生成任务流程

1. 按当前项目要求确认业务操作、筛选字段、响应字段和最终接口清单。
2. 先完成项目规定的 SQL/迁移与结构同步，再运行项目生成入口。若步骤失败，先定位失败原因；不要跳过或手写替代产物。
3. 检查日志及实际文件，防止某个文件生成失败但整体命令仍结束。
4. 基于生成产物按业务裁剪 API，并接入项目聚合入口；go-zero 项目执行 API validate 后重新生成下游 Types、路由、Handler 和 Logic。
5. 模板缺少标准业务实现或格式不符合项目要求时，修正实际使用的模板并重生成。不能在生成文件上手工补完后宣称可复现。
6. 仅在项目允许的扩展位置补充生成器无法表达的业务校验或编排。
7. 按项目要求在独立临时目标用相同版本、输入、配置和模板重生成，比较目标产物。保护已有手写实现，不为验证直接删除工作区文件。

## 配置定位

常见键包括 `out_path`、`model_pkg_path`、`repo_path`、`api_path`、`api_mapper_path`、`proto_mapper_path` 和 `goctl_template_home`，具体支持情况以当前 `generator/config.go` 为准。

不要假定所有模板都能通过同一个配置项覆盖；追踪当前生成入口如何定位模板。更新依赖库内嵌模板涉及库发布/依赖变更，先确认是否属于任务范围。

来源和核对基线见 [来源与验证场景](verification.md)。
