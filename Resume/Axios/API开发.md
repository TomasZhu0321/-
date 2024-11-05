# JPA
## 最常用的是：
* 直接使用JpaRepository提供的基础CRUD方法
* 通过方法名定义简单查询
* 使用@Query写SQL处理复杂查询
## 记住这些就够用了：
* 实体类上要加@Entity注解
* 主键字段加@Id注解
* Repository接口继承JpaRepository
* 复杂查询用@Query写SQL

# Query 学校列表api
* 输入: cityId
* 输出: List<school>

## 优化
* 通过redis缓存进行了优化
要点说明：
* key设计: "school:list:city:${cityId}"
    * 业务前缀:school
    * 数据类型:list
    * 查询条件:city
    * 具体值:cityId
* value设计:
    * 使用JSON字符串存储
    * List<SchoolDTO>序列化后的结果
    * DTO中只包含必要字段（id,name）

* 过期时间:
    * 通过配置文件设置
    * 使用TimeUnit.SECONDS指定单位
    * 可以根据不同条件设置不同过期时间
```java
@Service
public class SchoolService {
    // 注入Redis模板
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 从配置文件中读取过期时间
    @Value("${redis.cache.school.ttl:86400}")
    private long cacheTTL; // 默认24小时
    
    public List<SchoolDTO> getSchoolsByCity(Long cityId) {
        // 1. 构造缓存key
        String cacheKey = "school:list:city:" + cityId;
        
        // 2. 尝试从Redis获取
        String cachedData = redisTemplate.opsForValue().get(cacheKey);
        if (cachedData != null) {
            // 反序列化缓存数据
            return JSON.parseArray(cachedData, SchoolDTO.class);
        }
        
        // 3. 缓存未命中，从数据库查询
        List<SchoolDTO> schools = schoolRepository.findByCity(cityId);
        
        // 4. 存入Redis，设置过期时间
        redisTemplate.opsForValue().set(
            cacheKey,
            JSON.toJSONString(schools),
            cacheTTL,
            TimeUnit.SECONDS
        );
        
        return schools;
    }
}
```