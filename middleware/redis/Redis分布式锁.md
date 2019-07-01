以下是基于单redis实例，通过setnx 和 getSet 实现的 redis 分布式锁。

```java
/**
 * 获取一个锁 必须保证分布式环境的多个主机的时钟是一致的
 *
 * @param lockKey 锁Key
 * @param expired 锁的失效时间（毫秒）
 */
public boolean acquireLock(String lockKey, long expired) {

    ShardedJedis jedis = null;

    try {

        jedis = pool.getResource();// ShardedJedisPool
        String value = String.valueOf(System.currentTimeMillis() + expired + 1);
        int tryTimes = 0;

        while (tryTimes++ < 3) {

            /*
             *  1. 尝试锁
             *  setnx : set if not exist
             */
            if (jedis.setnx(lockKey, value).equals(1L)) {
                return true;
            }

            /*
             * 2. 已经被别的线程锁住，判断是否失效
             */
            String oldValue = jedis.get(lockKey);
            if (StringUtils.isBlank(oldValue)) {
                /*
                 * 2.1 value存的是超时时间，如果为空有2种情况
                 *      1. 异常数据，没有value 或者 value为空字符
                 *      2. 锁恰好被别的线程释放了
                 * 此时需要尝试重新尝试，为了避免出现情况1时导致死循环，只重试3次
                 */
                continue;
            }

            Long oldValueL = Long.valueOf(oldValue);
            if (oldValueL < System.currentTimeMillis()) {
                /*
                 * 已超时，重新尝试锁
                 *
                 * Redis:getSet 操作步骤：
                 *      1.获取 Key 对应的 Value 作为返回值，不存在时返回null
                 *      2.设置 Key 对应的 Value 为传入的值
                 * 这里如果返回的 getValue != oldValue 表示已经被其它线程重新修改了
                 */
                String getValue = jedis.getSet(lockKey, value);
                return oldValue.equals(getValue);
            } else {
                // 未超时，则直接返回失败
                return false;
            }
        }

        return false;

    } catch (Throwable e) {
        logger.error("acquireLock error", e);
        return false;

    } finally {
        returnResource(jedis);
    }
}
```