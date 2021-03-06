# 业务场景

一般情况下，对于一份数据，如果查询需求和更新需求同时存在，我们会在 Redis 中保存一份业务数据副本，当查询时，先从 Redis 缓存中查询，如果缓存有数据，则直接返回缓存数据，如果缓存中没有数据，再从 DB 数据库中查询，从而降低数据库的查询次数，降低 DB 的压力，进而提升应用的 QPS 。具体代码如下：

```java
@Override
public V queryById(Long id) {
  
   // 1. 先查询 redis 缓存数据
   String redisKey = PREFIX + id;
   V v = redis.get(redisKey, V.class);
  
   // 2. 如果 redis 缓存中有数据，则直接返回缓存数据
   if (v != null) {
      return v;
   }
  
   // 3. 缓存没有数据，再从 DB 查询
   v = dao.queryFromDB(id);
  
   // 4. 如果数据库有数据，则将数据放入缓存
   if (v != null ) {
      redis.set(redisKey, v);
   }
   return v;
}
```

以上时业务数据查询时的场景，基本都是上面的流程不变，但是，在数据需要更新时，面临一个问题：是先更新缓存，还是先更新 DB ？

## 先删除 Redis，再更新 DB
```java
@Override
public int updateById(Long id) {
  
   // 1. 先删除 redis 缓存数据
   String redisKey = PREFIX + id;
   redis.del(redisKey);
  
   // 2. 更新 DB 数据
   return dao.updateDB(id);
}
```

现象：很容易产生 Redis 脏数据

#### 并发异常场景

两个并发操作，一个是更新操作，另一个是查询操作，更新操作删除缓存后，查询操作没有命中缓存，先把老数据读出来后放到缓存中，然后更新操作更新了数据库。于是，在缓存中的数据还是老的数据，导致缓存中的数据是脏的，而且还一直这样脏下去了

| 线程A <br/>更新1->2 | 线程B<br/> 查询                                     | Redis<br/> 1 | DB<br/> 1 |
| ------------------- | --------------------------------------------------- | :----------: | :-------: |
| 删除缓存            |                                                     |     Nil      |     1     |
|                     | 查询缓存无数据<br>查询DB = 1<br>将查询结果1放入缓存 |      1       |     1     |
| 更新DB = 2          |                                                     |      1       |     2     |

#### 一致性异常场景

先删除 Redis，程序出现异常，然后 DB 未做处理，此时如果删 redis 操作是删除成功的，只是相应超时，那么最坏的影响也只是增加一次 Cache miss，不会出现数据不一致。

##先更新 DB，再删除 Redis

**该模式使用场景较多**

```java
@Override
public int updateById(Long id) {
   // 1. 更新 DB 数据
   int n = dao.updateDB(id);
   String redisKey = PREFIX + id;
   
   // 2. 删除 redis 缓存数据
   redis.del(redisKey);
   return n;
}
```

结论：产生脏数据的概率较小，但是会出现一致性的问题；若更新操作的时候，同时进行查询操作，若hit，则查询得到的数据是旧的数据。但是不会影响后面的查询。（代价较小）

一个是查询操作，一个是更新操作的并发，首先，没有了删除cache数据的操作了，而是先更新了数据库中的数据，此时，缓存依然有效，所以，并发的查询操作拿的是没有更新的数据，但是，更新操作马上让缓存的失效了，后续的查询操作再把数据从数据库中拉出来。而不会像文章开头的那个逻辑产生的问题，后续的查询操作一直都在取老的数据。

#### 并发异常场景
<u>一个是读操作，但是没有命中缓存，然后就到数据库中取数据，此时来了一个写操作，写完数据库后，让缓存失效，然后，之前的那个读操作再把老的数据放进去，所以，会造成脏数据。</u>

| 线程A <br/>更新1->2              | 线程B<br/> 查询                 | Redis<br> Nil | DB<br/> 1 |
| -------------------------------- | ------------------------------- | ------------- | --------- |
|                                  | 查询缓存无数据 <br/>查询DB得到1 | Nil           | 1         |
| 更新DB = 2 <br/>删除缓存(无数据) |                                 | Nil           | 2         |
|                                  | 将得到的1保存到缓存             | 1             | 2         |

**该情况出现的概率可能非常低**，因为这个条件需要发生在读缓存时缓存失效，而且并发着有一个写操作。而实际上数据库的写操作会比读操作慢得多，而且还要锁表，而读操作必需在写操作前进入数据库操作，而又要晚于写操作更新缓存，所有的这些条件都具备的概率基本并不大。 

#### 一致性异常场景

先更新数据库成功，跟新 redis 失败了，此时会出现 redis 与数据库不一致的现象，此时无法恢复。



# 总结

根据上面的分析，我们可以看到，在实际使用中，需要根据真实的业务场景与生产环境的情况，来觉定采用哪一种方案

1. **如果是并发非常高（尤其是查询操作），但是网络很好，也就是出现异常的情况非常低，建议使用先更新DB，再更新缓存的策略**
2. **如果是并发并不高，但是网络条件很差，操作经常失败的情况，建议使用先更新 Redis，从而降低数据一致性出现的概率**