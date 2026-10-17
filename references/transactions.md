# 事务

## 使用方式

基线 `TransactionAs` 接收数据库连接、把连接转为项目查询对象的函数（通常为 `dao.Use`）、事务回调，以及可选的 `*sql.TxOptions`。

下面是 Logic/Service 内的调用形状示意；`l.orderRepository` 和 `l.orderDetailRepository` 为其构造函数已初始化的完整名称接口字段，其他变量由项目现有实现提供，不是可以直接复制编译的完整程序：

```go
// 在同一事务内创建主表和明细，任何一步失败都返回错误。
err := gormplus.TransactionAs(db.WithContext(ctx), dao.Use, func(tx *dao.Query) error {
    if err := l.orderRepository.CreateTx(ctx, tx, orderEntity); err != nil {
        return err
    }
    return l.orderDetailRepository.CreateBatchTx(ctx, tx, detailEntities)
})
```

跨 Repository 的事务业务流程、业务校验和业务错误处理放在 Logic/Service 或项目已有业务层，禁止放入 Repository，也不能重写模板 CRUD 来容纳这些规则。业务层调用已在构造函数初始化的完整名称 Repository 字段；Repository 只提供数据操作与事务绑定能力，不在业务方法中临时创建 Repository。

## 必须核对的行为

- 依赖同一数据状态的业务校验与写入在业务层的同一事务回调中完成：通过事务绑定的 Repository 获取数据，业务层判断后调用事务绑定的写入方法。不能为保持分层将校验移到事务外；保持原有隔离级别及必要的锁/约束。
- 事务回调中的每次写入都使用同一 `tx`。调用普通 `Create` 或全局 `dao.Xxx` 通常会走事务外连接。
- 当前生成接口是否确有 `CreateTx`、`CreateBatchTx` 等方法，参数顺序是否一致；没有时先核对生成模板与项目版本。
- 基线的 `CreateTx` 使用传入查询对象的实体 DAO，再 `WithContext(ctx)`；普通 `Create` 使用全局 DAO。方法体比名称更能证明事务归属。
- 错误必须返回给事务回调；记录日志后返回 nil 会让事务继续提交。
- 事务内读取也要绑定事务连接，不要假设普通 `FindById` 能看见未提交写入。当前模板未必提供所有读取方法的 Tx 版本，优先复用实际支持的事务绑定机制（如 `WithTx`）与默认方法，确实缺失时才在对应的已有非生成文件中新增纯数据读取方法；不能另建文件，也不能把业务判断带入该方法。
- Context 必须保留租户、权限与操作人信息；连接必须符合项目数据源路由。不要凭空假定默认连接就是目标数据源。

## 缓存与外部副作用

基线的生成 `CreateTx` 在写入成功后立即调用缓存失效，发生在外层事务提交前。这不等于可靠的提交后失效保证。涉及缓存一致性时，需要检查实际提交、回滚与并发读取时序，不能在 Skill 中承诺事务缓存天然一致。

数据库事务不能回滚已发送的消息或远端 API 操作。涉及这些动作时沿用项目现有提交后处理或 outbox 方案，不为简单数据库事务额外搭建系统。

## 验证

至少按任务需要覆盖：两步都成功；第二步失败时第一步不落库；Context 保持有效。需要事务内读取、缓存或并发保证时，再补相应场景。使用隔离测试数据，不操作已有业务数据。

来源和核对基线见 [来源与验证场景](verification.md)。
