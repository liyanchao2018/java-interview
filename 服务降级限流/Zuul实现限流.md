# [Zuul：构建高可用网关之多维度限流](https://my.oschina.net/giegie/blog/1583705)



- 对请求的目标URL进行限流（例如：某个URL每分钟只允许调用多少次）
- 对客户端的访问IP进行限流（例如：某个IP每分钟只允许请求多少次）
- 对某些特定用户或者用户组进行限流（例如：非VIP用户限制每分钟只允许调用100次某个API等）
- 多维度混合的限流。此时，就需要实现一些限流规则的编排机制。与、或、非等关系。

## 介绍

[spring-cloud-zuul-ratelimit](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fgithub.com%2Fmarcosbarbero%2Fspring-cloud-zuul-ratelimit)是和zuul整合提供分布式限流策略的扩展，**只需在yaml中配置几行配置，就可使应用支持限流**

```xml
<dependency>
    <groupId>com.marcosbarbero.cloud</groupId>
    <artifactId>spring-cloud-zuul-ratelimit</artifactId>
    <version>1.3.4.RELEASE</version>
</dependency>
```

## 支持的限流粒度

- 服务粒度 (默认配置，当前服务模块的限流控制)
- 用户粒度 （详细说明，见文末总结）
- ORIGIN粒度 (用户请求的origin作为粒度控制)
- 接口粒度 (请求接口的地址作为粒度控制)
- 以上粒度自由组合，又可以支持多种情况。
- 如果还不够，自定义RateLimitKeyGenerator实现。

```java
//默认实现
public String key(final HttpServletRequest request, final Route route, final RateLimitProperties.Policy policy) {
    final List<Type> types = policy.getType();
    final StringJoiner joiner = new StringJoiner(":");
    joiner.add(properties.getKeyPrefix());
    if (route != null) {
        joiner.add(route.getId());
    }
    if (!types.isEmpty()) {
        if (types.contains(Type.URL) && route != null) {
            joiner.add(route.getPath());
        }
        if (types.contains(Type.ORIGIN)) {
            joiner.add(getRemoteAddr(request));
        }
        // 这个结合文末总结。
        if (types.contains(Type.USER)) {
            joiner.add(request.getUserPrincipal() != null ? request.getUserPrincipal().getName() : ANONYMOUS_USER);
        }
    }
    return joiner.toString();
}
```

## 支持的存储方式

![image](http://a.pigx.top/ratelimiter.png)

- InMemoryRateLimiter - 使用 ConcurrentHashMap作为数据存储
- ConsulRateLimiter - 使用 Consul 作为数据存储
- RedisRateLimiter - 使用 Redis 作为数据存储
- SpringDataRateLimiter - 使用 数据库 作为数据存储

## 限流配置

- limit 单位时间内允许访问的个数
- quota 单位时间内允许访问的总时间（统计每次请求的时间综合）
- refresh-interval 单位时间设置

```
zuul:
  ratelimit:
    key-prefix: your-prefix 
    enabled: true 
    repository: REDIS 
    behind-proxy: true
    policies:
      myServiceId:
        limit: 10
        quota: 20
        refresh-interval: 30
        type:
          - user
        
```

以上配置意思是：30秒内允许10个访问，或者要求总请求时间小于20秒

## 效果展示

yaml配置：

```
zuul:
  ratelimit:
    key-prefix: pig-ratelimite 
    enabled: true 
    repository: REDIS 
    behind-proxy: true
    policies:
      pig-admin-service:
        limit: 2
        quota: 1
        refresh-interval: 3
```

**动态图 ↓↓↓↓↓**
![image](.\image\limit01.gif)

**Redis 中数据结构 注意红色字体** ![image](.\image\redislimiter.png)

## 总结

- 可以使用Spring Boot Actuator 提供的服务状态，动态设置限流开关
- 源码可以参考：https://gitee.com/log4j/pig        
- 源代码已下载下来，同级目录下：log4j-pig-master.zip
- 用户限流的实现：如果你的项目整合 Shiro 或者 Spring Security 安全框架，那么会自动维护request域UserPrincipal，如果是自己的框架，请登录成功后维护request域**UserPrincipal**，才能使用用户粒度的限流，未登录默认是：**anonymous**。具体代码实现可以看 DefaultRateLimitKeyGenerator,type为USER的实现