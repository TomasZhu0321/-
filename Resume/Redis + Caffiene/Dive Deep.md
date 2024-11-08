# Cache Break
* where a frequently accessed key in the cache expires, causing a sudden surge of requests to hit the backend database.

# Cache Avalanche
*  a large number of cache entries expire simultaneously or the cache service fails entirely, causing a massive number of requests to flood the backend system.

# Solution
* 提前预热(Advance Cache Warming)（推荐）：针对热点数据提前预热，将其存入缓存中并设置合理的过期时间比如秒杀场景下的数据在秒杀结束之前不过期。
    * Pre-warm hot data in advance by storing it in the cache and setting appropriate expiration times. For example, in a flash sale scenario, the data should not expire before the sale ends."
* 使用定时任务，比如 xxl-job，来定时触发缓存预热的逻辑，将数据库中的热点数据查询出来并存入缓存中。
    * Use scheduled tasks, such as XXL-Job, to periodically trigger cache warming logic that queries hot data from the database and stores it in the cache.

