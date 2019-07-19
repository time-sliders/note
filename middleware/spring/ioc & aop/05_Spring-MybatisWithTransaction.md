在继承 ibatis 的数据访问层，往往都会继承 `SqlSessionDaoSupport`，该对象往往是Spring Bean，在构建对象的时候，自动注入已经定义好的  `SqlSessionFactory`，代码如下

**SqlSessionDaoSupport**

```java
@Autowired(required = false)
public final void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
  if (!this.externalSqlSession) {
    this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);  // 每一次数据库操作都是用 SqlSessionTemplate 来进行操作的
  }
}
```

如果要实现事物机制，那么这个 sqlSession 必须使用 Spring-AOP 中被 Spring 管理的 `Connection` 来进行数据库操作，我们继续看下 `SqlSessionTemplate` 的构造器。

**SqlSessionTemplate**

```java
/**
 * Constructs a Spring managed {@code SqlSession} with the given
 * {@code SqlSessionFactory} and {@code ExecutorType}.
 * A custom {@code SQLExceptionTranslator} can be provided as an
 * argument so any {@code PersistenceException} thrown by MyBatis
 * can be custom translated to a {@code RuntimeException}
 * The {@code SQLExceptionTranslator} can also be null and thus no
 * exception translation will be done and MyBatis exceptions will be
 * thrown
 *
 * @param sqlSessionFactory
 * @param executorType
 * @param exceptionTranslator
 */
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
  notNull(executorType, "Property 'executorType' is required");

  this.sqlSessionFactory = sqlSessionFactory;
  this.executorType = executorType;
  this.exceptionTranslator = exceptionTranslator;
  this.sqlSessionProxy = (SqlSession) newProxyInstance(
      SqlSessionFactory.class.getClassLoader(),
      new Class[] { SqlSession.class },
      new SqlSessionInterceptor());
}

public int update(String statement, Object parameter) {
    return this.sqlSessionProxy.update(statement, parameter);
}
```

这里可以看到一个 `sqlSessionProxy` 对象，后续，所有的数据库操作，都是调用的该对象上的方法，很明显，这是一个代理类。最终执行的是 `SqlSessionInterceptor#invoke` 方法

```java
/**
 * Proxy needed to route MyBatis method calls to the proper SqlSession got
 * from Spring's Transaction Manager
 * It also unwraps exceptions thrown by {@code Method#invoke(Object, Object...)} to
 * pass a {@code PersistenceException} to the {@code PersistenceExceptionTranslator}.
 */
private class SqlSessionInterceptor implements InvocationHandler {
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 获取 sqlSession，这里返回的是被 Spring TransactionManager 管理的 sqlSession, 具体实现后面分析
    final SqlSession sqlSession = getSqlSession(
        SqlSessionTemplate.this.sqlSessionFactory,
        SqlSessionTemplate.this.executorType,
        SqlSessionTemplate.this.exceptionTranslator);
    try {
      Object result = method.invoke(sqlSession, args); // 调用真实的 sqlSession 目标方法
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        sqlSession.commit(true);
      }
      return result;
    } catch (Throwable t) {
      Throwable unwrapped = unwrapThrowable(t);
      if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
        Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
        if (translated != null) {
          unwrapped = translated;
        }
      }
      throw unwrapped;
    } finally {
      closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
    }
  }
}
```

**SqlSessionUtils**

```java
/**
 * Gets an SqlSession from Spring Transaction Manager or creates a new one if needed.
 * Tries to get a SqlSession out of current transaction. If there is not any, it creates a new one.
 * Then, it synchronizes the SqlSession with the transaction if Spring TX is active and
 * <code>SpringManagedTransactionFactory</code> is configured as a transaction manager.
 *
 * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
 * @param executorType The executor type of the SqlSession to create
 * @param exceptionTranslator Optional. Translates SqlSession.commit() exceptions to Spring exceptions.
 * @throws TransientDataAccessResourceException if a transaction is active and the
 *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
 * @see SpringManagedTransactionFactory
 */
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sessionFactory, "No SqlSessionFactory specified");
  notNull(executorType, "No ExecutorType specified");

  // 从 Spring 的 TransactionSynchronizationManager 中获取 ThreadLocal holder
  SqlSessionHolder holder = (SqlSessionHolder) getResource(sessionFactory);

  if (holder != null && holder.isSynchronizedWithTransaction()) {
    if (holder.getExecutorType() != executorType) {
      throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
    }

    holder.requested();

    if (logger.isDebugEnabled()) {
      logger.debug("Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction");
    }

    return holder.getSqlSession();// 返回对应的会话
  }

  if (logger.isDebugEnabled()) {
    logger.debug("Creating a new SqlSession");
  }

  SqlSession session = sessionFactory.openSession(executorType);

  // Register session holder if synchronization is active (i.e. a Spring TX is active)
  //
  // Note: The DataSource used by the Environment should be synchronized with the
  // transaction either through DataSourceTxMgr or another tx synchronization.
  // Further assume that if an exception is thrown, whatever started the transaction will
  // handle closing / rolling back the Connection associated with the SqlSession.
  if (isSynchronizationActive()) {
    Environment environment = sessionFactory.getConfiguration().getEnvironment();

    if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
      if (logger.isDebugEnabled()) {
        logger.debug("Registering transaction synchronization for SqlSession [" + session + "]");
      }

      holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
      // 绑定到 TransactionSynchronizationManager
      bindResource(sessionFactory, holder);
      // 注册 Synchronization
      registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
      // 设置状态，方便下一次处理
      holder.setSynchronizedWithTransaction(true);
      holder.requested();
    } else {
      if (getResource(environment.getDataSource()) == null) {
        if (logger.isDebugEnabled()) {
          logger.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
        }
      } else {
        throw new TransientDataAccessResourceException(
            "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
      }
    }
  } else {
    if (logger.isDebugEnabled()) {
      logger.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
    }
  }

  return session;
}
```