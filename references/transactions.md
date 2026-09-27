# 事务

## 使用方式

基线 `TransactionAs` 接收数据库连接、把连接转为项目查询对象的函数（通常为 `dao.Use`）、事务回调，以及可选的 `*sql.TxOptions`。

下面是调用形状示意，变量由项目现有实现提供；不是可以直接复制编译的完整程序：

```go
// 在同一事务内创建主表和明细，任何一步失败都返回错误。
err := gormplus.TransactionAs(db.WithContext(ctx), dao.Use, func(tx *dao.Query) error {
    if err := orderRepository.CreateTx(ctx, tx, orderEntity); err != nil {
        return err
    }
    return orderDetailRepository.CreateBatchTx(ctx, tx, detailEntities)
})
```

该编排放在哪一层由项目规定。若要求跨表事务放在 Repository，就在对应表的非生成扩展文件实现；不要用这个示意绕过层级规则，也不要在 Logic 方法中临时创建 Repository。

## 必须核对的行为

- 事务回调中的每次写入都使用同一 `tx`。调用普通 `Create` 或全局 `dao.Xxx` 通常会走事务外连接。
- 当前生成接口是否确有 `CreateTx`、`CreateBatchTx` 等方法，参数顺序是否一致；没有时先核对生成模板与项目版本。
- 基线的 `CreateTx` 使用传入查询对象的实体 DAO，再 `WithContext(ctx)`；普通 `Create` 使用全局 DAO。方法体比名称更能证明事务归属。
- 错误必须返回给事务回调；记录日志后返回 nil 会让事务继续提交。
- 事务内读取也要绑定事务连接，不要假设普通 `FindById` 能看见未提交写入。当前模板未必提供所有读取方法的 Tx 版本，必要时在允许的扩展位置实现事务读取。
- Context 必须保留租户、权限与操作人信息；连接必须符合项目数据源路由。不要凭空假定默认连接就是目标数据源。

## 缓存与外部副作用

基线的生成 `CreateTx` 在写入成功后立即调用缓存失效，发生在外层事务提交前。这不等于可靠的提交后失效保证。涉及缓存一致性时，需要检查实际提交、回滚与并发读取时序，不能在 Skill 中承诺事务缓存天然一致。

数据库事务不能回滚已发送的消息或远端 API 操作。涉及这些动作时沿用项目现有提交后处理或 outbox 方案，不为简单数据库事务额外搭建系统。

## 验证

至少按任务需要覆盖：两步都成功；第二步失败时第一步不落库；Context 保持有效。需要事务内读取、缓存或并发保证时，再补相应场景。使用隔离测试数据，不操作已有业务数据。

来源和核对基线见 [来源与验证场景](verification.md)。
