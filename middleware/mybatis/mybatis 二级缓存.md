# 什么是MyBatis缓存

我们知道，频繁的数据库操作是非常耗费性能的（主要是因为对于DB而言，数据是持久化在磁盘中的，因此查询操作需要通过IO，IO操作速度相比内存操作速度慢了好几个量级），尤其是对于一些相同的查询语句，完全可以把查询结果存储起来，下次查询同样的内容的时候直接从内存中获取数据即可，这样在某些场景下可以大大提升查询效率。

MyBatis的缓存分为两种：

1. 一级缓存，一级缓存是**SqlSession级别**的缓存，对于相同的查询，会从缓存中返回结果而不是查询数据库
2. 二级缓存，二级缓存是**Mapper级别**的缓存，定义在Mapper文件的<cache>标签中并需要开启此缓存，多个Mapper文件可以共用一个缓存，依赖<cache-ref>标签配置

下面来详细看一下MyBatis的一二级缓存。

# 一级缓存

## **MyBatis一级缓存工作流程**

接着看一下MyBatis一级缓存工作流程。前面说了，MyBatis的一级缓存是SqlSession级别的缓存，当openSession()的方法运行完毕或者主动调用了SqlSession的close方法，SqlSession就被回收了，一级缓存与此同时也一起被回收掉了。前面的文章有说过，在MyBatis中，无论selectOne还是selectList方法，最终都被转换为了selectList方法来执行，那么看一下SqlSession的selectList方法的实现：

DefaultSqlSession

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

BaseExecutor

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

第3行构建缓存条件CacheKey，这里涉及到怎么样条件算是和上一次查询是同一个条件的一个问题，因为同一个条件就可以返回上一次的结果回去，这部分代码留在下一部分分析。

接着看第4行的query方法的实现

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
  throws SQLException {
  Cache cache = ms.getCache();
  if (cache != null) {
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  // 具体的查询逻辑，交由“代表”去实现，这里的 delegate 就是 BaseExecutor
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

BaseExecutor

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
    // 先尝试从 localCache 缓存中获取数据
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 如果缓存中没有数据，则从 database 中查询数据
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}
```

MyBatis一级缓存存储流程看完了，接着我们从这段代码中可以得到三个结论：

1. **MyBatis的一级缓存是 SqlSession 级别的，但是它并不定义在 SqlSessio 接口的实现类 DefaultSqlSession 中，而是定义在 DefaultSqlSession 的成员变量 Executor 中，Executor 是在 openSession 的时候被实例化出来的，它的默认实现为 SimpleExecutor**
2. MyBatis 中的一级缓存，与有没有配置无关，**只要 SqlSession 存在，MyBastis 一级缓存就存在**，localCache的类型是 PerpetualCache，它其实很简单，**一个id属性+一个HashMap属性**，id是一个名为"localCache"的字符串，HashMap 用于存储数据，Key 为 CacheKey，Value 为查询结果
3. MyBatis的一级缓存查询的时候默认都是会先尝试从一级缓存中获取数据的，但是我们看代码中做了一个判断，ms.isFlushCacheRequired()，即**想每次查询都走DB也行，将<select>标签中的flushCache属性设置为true即可**，这意味着每次查询的时候都会清理一遍PerpetualCache，PerpetualCache中没数据，自然只能走DB

从MyBatis一级缓存来看，它以单纯的HashMap做缓存，没有容量控制，而一次SqlSession中通常来说并不会有大量的查询操作，因此只适用于一次SqlSession，如果用到二级缓存的Mapper级别的场景，有可能缓存数据不断碰到而导致内存溢出。

还有一点，差点忘了写了，<insert>、<delete>、<update>最终都会转换为update方法，看一下BaseExecutor的update方法：

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  // 更新操作清空本地缓存 
  clearLocalCache();
  return doUpdate(ms, parameter);
}
```

**所有的增、删、改都会清空本地缓存**，这和是否配置了flushCache=true是无关的。

这很好理解，因为增、删、改这三种操作都可能会导致查询出来的结果并不是原来的结果，如果增、删、改不清理缓存，那么可能导致读取出来的数据是脏数据。

## 一级缓存的CacheKey

接着我们看下一个问题：怎么样的查询条件算和上一次查询是一样的查询，从而返回同样的结果回去？这个问题，得从CacheKey说起。

我们先看一下CacheKey的数据结构：

```java
public class CacheKey implements Cloneable, Serializable {

  private static final long serialVersionUID = 1146682552656046210L;

  public static final CacheKey NULL_CACHE_KEY = new NullCacheKey();

  private static final int DEFAULT_MULTIPLYER = 37;
  private static final int DEFAULT_HASHCODE = 17;

  private final int multiplier;
  private int hashcode;
  private long checksum;
  private int count;
  // 8/21/2017 - Sonarlint flags this as needing to be marked transient.  While true if content is not serializable, this is not always true and thus should not be marked transient.
  private List<Object> updateList;
```

其中最重要的是 updateList 这个两个属性，为什么这么说，因为HashMap的Key是CacheKey，而HashMap的get方法是先判断hashCode，在hashCode冲突的情况下再进行equals判断，因此最终无论如何都会进行一次equals的判断，看下equals方法的实现：

```java
@Override
public boolean equals(Object object) {
  if (this == object) {
    return true;
  }
  if (!(object instanceof CacheKey)) {
    return false;
  }

  final CacheKey cacheKey = (CacheKey) object;

  if (hashcode != cacheKey.hashcode) {
    return false;
  }
  if (checksum != cacheKey.checksum) {
    return false;
  }
  if (count != cacheKey.count) {
    return false;
  }

  for (int i = 0; i < updateList.size(); i++) {
    Object thisObject = updateList.get(i);
    Object thatObject = cacheKey.updateList.get(i);
    if (!ArrayUtil.equals(thisObject, thatObject)) {
      return false;
    }
  }
  return true;
}
```

看到整个方法的流程都是围绕着updateList中的每个属性进行逐一比较，因此再进一步的，我们要看一下updateList中到底存储了什么。

关于updateList里面存储的数据我们可以看下哪里使用了updateList的add方法，然后一步一步反推回去即可。updateList中数据的添加是在update方法中：

```java
public void updateAll(Object[] objects) {
  for (Object o : objects) {
    update(o);
  }
}

public void update(Object object) {
  int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

  count++;
  checksum += baseHashCode;
  baseHashCode *= count;

  hashcode = multiplier * hashcode + baseHashCode;

  updateList.add(object);
}
```

其实update方法的调用方有挺多处，但是这里我们要看的是Executor中的，看一下BaseExecutor中的createCacheKey方法实现：

BaseExecutor

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  CacheKey cacheKey = new CacheKey();
  cacheKey.update(ms.getId());
  cacheKey.update(rowBounds.getOffset());
  cacheKey.update(rowBounds.getLimit());
  cacheKey.update(boundSql.getSql());
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
  // mimic DefaultParameterHandler logic
  for (ParameterMapping parameterMapping : parameterMappings) {
    if (parameterMapping.getMode() != ParameterMode.OUT) {
      Object value;
      String propertyName = parameterMapping.getProperty();
      if (boundSql.hasAdditionalParameter(propertyName)) {
        value = boundSql.getAdditionalParameter(propertyName);
      } else if (parameterObject == null) {
        value = null;
      } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
        value = parameterObject;
      } else {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        value = metaObject.getValue(propertyName);
      }
      cacheKey.update(value);
    }
  }
  if (configuration.getEnvironment() != null) {
    // issue #176
    cacheKey.update(configuration.getEnvironment().getId());
  }
  return cacheKey;
}
```

到了这里应当一目了然了，MyBastis从四组共五个条件判断两次查询是相同的：

1. **<select>标签所在的Mapper的Namespace+<select>标签的id属性**
2. **RowBounds的offset和limit属性，RowBounds是MyBatis用于处理分页的一个类，offset默认为0，limit默认为Integer.MAX_VALUE**
3. **<select>标签中定义的sql语句**
4. **输入参数的具体参数值，一个int值就update一个int，一个String值就update一个String，一个List就轮询里面的每个元素进行update**

即只要两次查询满足以上三个条件且没有定义flushCache="true"，那么第二次查询会直接从MyBatis一级缓存PerpetualCache中返回数据，而不会走DB。

# 二级缓存

上面说完了MyBatis，接着看一下MyBatis二级缓存，还是从二级缓存工作流程开始。还是从DefaultSqlSession的selectList方法进去：

跟一级缓存一样，看一下SqlSession的selectList方法的实现：

DefaultSqlSession

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

BaseExecutor

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

同样继续接着看第4行的query方法的实现

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
  throws SQLException {
  // 二级缓存实现
  Cache cache = ms.getCache();
  if (cache != null) {
    //根据flushCache=true或者flushCache=false判断是否要清理二级缓存
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      // 保证MyBatis二级缓存不会存储存储过程的结果
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      /*先尝试从tcm中获取查询结果，这个tcm解释一下，这又是一个装饰器模
      式（数数MyBatis用到了多少装饰器模式了），创建一个事物缓存TranactionalCache，
      持有Cache接口，Cache接口的实现类就是根据我们在Mapper文件中配置的
      <cache>创建的Cache实例*/
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        /*如果没有从MyBatis二级缓存中拿到数据，那么就会查一次数据库，然后放到MyBatis二级缓存中去*/
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  // 一级缓存实现
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

从这里看到，执行BaseExecutor的query方法之前，会先拿Mybatis二级缓存，而BaseExecutor的query方法会优先读取MyBatis一级缓存，由此可以得出一个重要结论：**假如定义了MyBatis二级缓存，那么MyBatis二级缓存读取优先级高于MyBatis一级缓存**。

至于如何判定上次查询和这次查询是一次查询？由于这里的CacheKey和MyBatis一级缓存使用的是同一个CacheKey，因此它的判定条件和前文写过的MyBatis一级缓存三个维度的判定条件是一致的。

最后再来谈一点，"**Cache cache = ms.getCache()**"这句代码十分重要，这意味着Cache是从MappedStatement中获取到的，而MappedStatement又和每一个<insert>、<delete>、<update>、<select>绑定并在MyBatis启动的时候存入Configuration中：

```
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
```

因此**MyBatis二级缓存的生命周期即整个应用的生命周期**，应用不结束，定义的二级缓存都会存在在内存中。

从这个角度考虑，为了避免MyBatis二级缓存中数据量过大导致内存溢出，MyBatis在配置文件中给我们增加了很多配置例如size（缓存大小）、flushInterval（缓存清理时间间隔）、eviction（数据淘汰算法）来保证缓存中存储的数据不至于太过庞大。

# MyBatis二级缓存实例化过程

接着看一下MyBatis二级缓存<cache>实例化的过程，代码位于XmlMapperBuilder的cacheElement方法中：

```java
private void cacheElement(XNode context) {
  if (context != null) {
    String type = context.getStringAttribute("type", "PERPETUAL");
    Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
    String eviction = context.getStringAttribute("eviction", "LRU");
    Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
    Long flushInterval = context.getLongAttribute("flushInterval");
    Integer size = context.getIntAttribute("size");
    boolean readWrite = !context.getBooleanAttribute("readOnly", false);
    boolean blocking = context.getBooleanAttribute("blocking", false);
    Properties props = context.getChildrenAsProperties();
    builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
  }
}
```

这里分别取<cache>中配置的各个属性，关注一下两个默认值：

1. type表示缓存实现，默认是PERPETUAL，根据typeAliasRegistry中注册的，PERPETUAL实际对应PerpetualCache，这和MyBatis一级缓存一致
2. eviction表示淘汰算法，默认是LRU算法

拿到了所有属性，那么调用useNewCache方法创建缓存：

```java
public Cache useNewCache(Class<? extends Cache> typeClass,
    Class<? extends Cache> evictionClass,
    Long flushInterval,
    Integer size,
    boolean readWrite,
    boolean blocking,
    Properties props) {
  Cache cache = new CacheBuilder(currentNamespace)
      .implementation(valueOrDefault(typeClass, PerpetualCache.class))
      .addDecorator(valueOrDefault(evictionClass, LruCache.class))
      .clearInterval(flushInterval)
      .size(size)
      .readWrite(readWrite)
      .blocking(blocking)
      .properties(props)
      .build();
  configuration.addCache(cache);
  currentCache = cache;
  return cache;
}
```

这里又使用了建造者模式，跟一下第16行的build()方法，在此之前该传入的参数都已经传入了CacheBuilder：

```java
public Cache build() {
  setDefaultImplementations();
  /*构建基础的缓存，implementation指的是type配置的值，这里是默认的PerpetualCache。*/
  Cache cache = newBaseCacheInstance(implementation, id);
  setCacheProperties(cache);
  // issue #352, do not apply decorators to custom caches
  /*如果是PerpetualCache，那么继续装饰，这里的装饰是根据eviction进行装饰，到这一步，
  给PerpetualCache加上了LRU的功能。*/
  if (PerpetualCache.class.equals(cache.getClass())) {
    for (Class<? extends Cache> decorator : decorators) {
      cache = newCacheDecoratorInstance(decorator, cache);
      setCacheProperties(cache);
    }
    /*继续装饰，这次MyBatis将它命名为标准装饰，setStandardDecorators方法实现见下面分析*/
    cache = setStandardDecorators(cache);
  } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
    cache = new LoggingCache(cache);
  }
  return cache;
}
```

```java
private Cache setStandardDecorators(Cache cache) {
  try {
    MetaObject metaCache = SystemMetaObject.forObject(cache);
    if (size != null && metaCache.hasSetter("size")) {
      metaCache.setValue("size", size);
    }
    if (clearInterval != null) {
      /*如果配置了flushInterval，那么继续装饰为ScheduledCache，这意味着在调
      用Cache的getSize、putObject、getObject、removeObject四个方法的时候
      都会进行一次时间判断，如果到了指定的清理缓存时间间隔，那么就会将当前缓存清空*/
      cache = new ScheduledCache(cache);
      ((ScheduledCache) cache).setClearInterval(clearInterval);
    }
    if (readWrite) {
      /*如果readWrite=true，那么继续装饰为SerializedCache，这意味着缓存中所
      有存储的内存都必须实现Serializable接口*/
      cache = new SerializedCache(cache);
    }
    /*跟配置无关，将之前装饰好的Cache继续装饰为LoggingCache与SynchronizedCache，
    前者在getObject的时候会打印缓存命中率，后者将Cache接口中所有的方法都加了Synchronized
    关键字进行了同步处理*/
    cache = new LoggingCache(cache);
    cache = new SynchronizedCache(cache);
    if (blocking) {
      /*如果blocking=true，那么继续装饰为BlockingCache，这意味着针对同一个CacheKey，
      拿数据与放数据、删数据是互斥的，即拿数据的时候必须没有在放数据、删数据*/
      cache = new BlockingCache(cache);
    }
    return cache;
  } catch (Exception e) {
    throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
  }
}
```

Cache全部装饰完毕，返回，至此MyBatis二级缓存生成完毕。

最后说一下，MyBatis支持三种类型的二级缓存：

- **MyBatis默认的缓存，type为空，Cache为PerpetualCache**
- **自定义缓存**
- **第三方缓存**

从build()方法来看，后两种场景的Cache，MyBatis只会将其装饰为LoggingCache，理由很简单，这些缓存的定期清除功能、淘汰过期数据功能开发者自己或者第三方缓存都已经实现好了，根本不需要依赖MyBatis本身的装饰。