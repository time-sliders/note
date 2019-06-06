# Java 程序使用Redis
代码：<https://github.com/xetorthio/jedis>  
下载：<https://github.com/xetorthio/jedis/releases>  
导入 redis 客户端jar包

#  Redis连接池配置
    JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
    jedisPoolConfig.setMaxTotal(1024);
    jedisPoolConfig.setMaxIdle(200);
    jedisPoolConfig.setMaxWaitMillis(1000);
    jedisPoolConfig.setTestOnBorrow(true);
#  Redis 连接信息
    JedisShardInfo jedisShardInfo = new JedisShardInfo("localhost",6379,500);
    List<JedisShardInfo> jedisShardInfoList = new LinkedList<JedisShardInfo>();
    jedisShardInfoList.add(jedisShardInfo);
    // 初始化连接池
    ShardedJedisPool shardedJedisPool = new ShardedJedisPool(jedisPoolConfig,jedisShardInfoList);
    ShardedJedis shardedJedis = null;
    try {
        // 获取连接
        shardedJedis = shardedJedisPool.getResource();
        /**
         * 保存数据
         * <p>相关方法相见api文档以及redis指令</p>
         * */
        shardedJedis.setex(SafeEncoder.encode("zhw"), 5000, new JsonSerializer().serialzation("zhangWei"));
    } finally {
        // 回收连接
        try {
            shardedJedisPool.returnResource(shardedJedis);
        } catch (Exception e) {
            shardedJedisPool.returnBrokenResource(shardedJedis);
        }
        // 销毁连接池
        shardedJedisPool.destroy();
    }
