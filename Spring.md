# Spring

## Spring的事务传播机制

Spring有七种事务传播机制

1. `TransactionDefinition.PROPAGATION_REQUIRED`（默认）

   如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

2. ``2.TransactionDefinition.PROPAGATION_REQUIRES_NEW`

   创建一个新的事务，如果当前存在事务，则把当前事务挂起。

   也就是说不管外部方法是否开启事务，`Propagation.REQUIRES_NEW`修饰的内部方法会新开启自己的事务，且开启的事务相互独立，互不干扰。

3. `TransactionDefinition.PROPAGATION_NESTED`

   如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行。

   如果当前没有事务，则该取值等价于`TransactionDefinition.PROPAGATION_REQUIRED`

4. `TransactionDefinition.PROPAGATION_MANDATORY`

   MANDATORY（强制）

   如果当前存在事务，则加入该事务。

   如果当前没有事务，则抛出异常。

   

   若是错误的配置以下 3 种事务传播行为，事务将不会发生回滚：

5. `TransactionDefinition.PROPAGATION_SUPPORTS`

   如果当前存在事务，则加入该事务。

   如果当前没有事务，则以非事务的方式继续运行。

6. `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`

   以非事务方式运行，如果当前存在事务，则把当前事务挂起。

7. `TransactionDefinition.PROPAGATION_NEVER`

   以非事务方式运行，如果当前存在事务，则抛出异常。



