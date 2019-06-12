@GlobalTransaction 注解 的扫描与 AOP 拦截
GlobalTransactionScanner 扫描有 GlobalTransaction 注解的类，如果有实现注解，则在一个新的 TransactionTemplate 中执行当前方法

TransactionTemplate 类实现分布式事务客户端的主要逻辑
在事务开始时，通过 TmRpcClient 提交到 Server 一个注册请求，Server 将该 GlobalTransaction 保存在内存，并写入事务 FileBasedSessionManager。
Client 端通过 ThreadLocal 的方式，保存当前事务信息

比较重要的类
AbstractSessionManager  ： server 端的 GlobalSession 管理工具，提供了事务状态变更的底层操作方法

分布式调用时，事务 ID 如何传播到另外一个进程
目前只支持 dubbo 事务传播，在 dubbo 协议中携带一个 attachment，服务方在接收到请求之后，将 ID 

客户端 FeScar 通过设置 ConnectionProxy 来对所有的数据库 Connection 操作做拦截，如果当前 Transaction 操作，在一个全局事务内部
1. 拦截 execute 方法，设置 autoCommit = false，在 sql 执行前后，分别记录 beforeImage 和 afterImage，记录 UndoLog 和 UndoItem
2. 拦截 commit 方法，调用 Server 的 doRegister 方法，注册 BranchTransaction 事务 （DefaultCore），注册完毕之后，Branch 相关记录与数据会写入到 Server 的本地文件中。

分支事务注册的时候，在 Server 端，会尝试获取分支锁，如果当前分支锁尚未被释放，则创建分支事务失败，失败原因为
1. DefaultCore#branchRegister() 方法实现分支注册，在代码内部，获取 lock，如果失败，则分布式事务失败