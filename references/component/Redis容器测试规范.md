# Redis 容器测试规范

本文档定义使用 Testcontainers Redis 容器进行 DT 测试的编写规范，适用于需要真实 Redis 交互的测试场景。

---

## 一、依赖配置

### 【强制】Maven 依赖

Testcontainers 无专用 Redis 模块，使用 `GenericContainer` + 官方 Redis 镜像。

```xml
<!-- Testcontainers 核心（已包含在父依赖中） -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.16.2</version>
    <scope>test</scope>
</dependency>
```

---

## 二、Redis 容器配置

### 2.1 基础配置

```java
static final GenericContainer<?> REDIS = new GenericContainer<>(
    DockerImageName.parse("redis:7-alpine"))
.withExposedPorts(6379);
```

### 2.2 常用配置项

| 方法 | 用途 | 示例 |
|------|------|------|
| `withExposedPorts()` | 暴露端口 | `.withExposedPorts(6379)` |
| `withCommand()` | 覆盖启动命令 | `.withCommand("redis-server", "--requirepass", "testpass")` |
| `withStartupTimeout()` | 启动超时 | `.withStartupTimeout(Duration.ofSeconds(30))` |

### 2.3 带密码配置

```java
static final GenericContainer<?> REDIS = new GenericContainer<>(
    DockerImageName.parse("redis:7-alpine"))
.withExposedPorts(6379)
.withCommand("redis-server", "--requirepass", "testpass");
```

---

## 三、RedisTemplate 注入

### 3.1 @DynamicPropertySource 注入

```java
@DynamicPropertySource
static void redisProperties(DynamicPropertyRegistry registry) {
    // Redis 容器已在基类 static 块中启动，此处仅注入属性
    registry.add("spring.data.redis.host", REDIS::getHost);
    registry.add("spring.data.redis.port", REDIS::getFirstMappedPort);
    // 如果设置了密码
    // registry.add("spring.data.redis.password", () -> "testpass");
}
```

### 3.2 application-testcontainers.yml 配置

```yaml
spring:
  data:
    redis:
      # host 和 port 由 @DynamicPropertySource 动态注入
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
```

---

## 四、数据隔离与清理

### 【强制】每个测试方法前后清理 Redis 数据

根据测试环境选择合适的清理策略：

**策略1：flushAll() — 独立 Redis 环境【推荐】**

适用于每个测试类使用独立 Redis 容器的场景。清空所有数据，确保测试隔离。

```java
@Autowired
protected RedisTemplate<String, Object> redisTemplate;

@BeforeEach
void setUp() {
    // 清空所有数据，确保测试隔离
    redisTemplate.getConnectionFactory().getConnection().flushAll();
}

@AfterEach
void tearDown() {
    // 清空所有数据
    redisTemplate.getConnectionFactory().getConnection().flushAll();
}
```

**策略2：按前缀批量删除 — 共享 Redis 环境**

适用于多个测试类共享同一 Redis 容器的场景。按统一前缀清理，避免影响其他测试。

```java
@AfterEach
void tearDown() {
    Set<String> keys = redisTemplate.keys("DT_TEST:*");
    if (keys != null && !keys.isEmpty()) {
        redisTemplate.delete(keys);
    }
}
```

### 【强制】测试 Key 使用统一前缀

```java
private static final String KEY_PREFIX = "DT_TEST:";

String cacheKey = KEY_PREFIX + "user:" + userId;  // DT_TEST:user:001
```

---

## 五、完整示例

### 5.1 Redis 容器基类

```java
package com.example.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

/**
 * Redis 容器级 DT 基类
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractRedisTestcontainers {

    protected static final GenericContainer<?> REDIS;

    static {
        REDIS = new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);
        REDIS.start();
    }

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", REDIS::getFirstMappedPort);
    }
}
```

### 5.2 缓存测试

```java
package com.example.cache;

import com.example.dt.AbstractRedisTestcontainers;
import com.example.entity.User;
import com.example.service.UserCacheService;
import com.example.repository.UserRepository;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * 用户缓存容器级 DT 测试
 *
 * 测试策略：真实 Redis 容器，验证序列化、TTL、缓存逻辑
 */
class UserCacheServiceTest extends AbstractRedisTestcontainers {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private UserCacheService userCacheService;

    @Autowired
    private UserRepository userRepository;

    private static final String KEY_PREFIX = "DT_TEST:";

    @BeforeEach
    void setUp() {
        redisTemplate.getConnectionFactory().getConnection().flushAll();
        userRepository.save(createTestUser("user_001", "张三"));
    }

    @AfterEach
    void tearDown() {
        Set<String> keys = redisTemplate.keys(KEY_PREFIX + "*");
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
        userRepository.deleteByUserIdStartingWith("DT_TEST_");
    }

    @Test
    @DisplayName("正常用例-缓存命中直接返回缓存数据")
    void getUser_cacheHit_returnCachedData() {
        // Given：预先写入缓存
        String cacheKey = KEY_PREFIX + "user:DT_TEST_user_001";
        User cachedUser = createTestUser("user_001", "张三");
        redisTemplate.opsForValue().set(cacheKey, cachedUser);

        // When
        User result = userCacheService.getUser("DT_TEST_user_001");

        // Then：验证缓存命中
        assertThat(result).isNotNull();
        assertThat(result.getUserName()).isEqualTo("张三");
        assertThat(redisTemplate.hasKey(cacheKey)).isTrue();
    }

    @Test
    @DisplayName("正常用例-缓存未命中查询数据库并回填缓存")
    void getUser_cacheMiss_queryDbAndFillCache() {
        // Given：缓存为空（setUp 中已 flushAll）
        String cacheKey = KEY_PREFIX + "user:DT_TEST_user_001";

        // When
        User result = userCacheService.getUser("DT_TEST_user_001");

        // Then：验证数据库查询 + 缓存回填
        assertThat(result).isNotNull();
        assertThat(result.getUserName()).isEqualTo("张三");
        // 验证缓存已回填（真实 Redis 操作）
        assertThat(redisTemplate.hasKey(cacheKey)).isTrue();
        Object cachedValue = redisTemplate.opsForValue().get(cacheKey);
        assertThat(cachedValue).isNotNull();
    }

    @Test
    @DisplayName("正常用例-TTL过期后重新查询数据库")
    void getUser_ttlExpired_queryDbAgain() throws InterruptedException {
        // Given：设置短 TTL
        String cacheKey = KEY_PREFIX + "user:DT_TEST_user_001";
        User cachedUser = createTestUser("user_001", "张三");
        redisTemplate.opsForValue().set(cacheKey, cachedUser, Duration.ofSeconds(1));

        // When：等待 TTL 过期
        Thread.sleep(1100);
        User result = userCacheService.getUser("DT_TEST_user_001");

        // Then：验证 TTL 过期后重新查询
        assertThat(result).isNotNull();
    }

    private User createTestUser(String userId, String name) {
        User user = new User();
        user.setUserId("DT_TEST_" + userId);
        user.setUserName(name);
        return user;
    }
}
```

---

## 六、Embedded DT vs Testcontainers DT 选择

| 场景 | Embedded DT (jedis-mock) | Testcontainers DT (Redis 容器) |
|------|--------------------------|-------------------------------|
| 基本缓存读写 | ✅ 足够 | ✅ |
| Redis 特定命令 | ⚠️ 部分不支持 | ✅ 完全支持 |
| TTL 过期机制 | ⚠️ 模拟 | ✅ 真实 |
| Lua 脚本 | ❌ 不支持 | ✅ 支持 |
| Redis Stream | ❌ 不支持 | ✅ 支持 |
| Redis Cluster | ❌ 不支持 | ✅ 可模拟 |
| 发布/订阅 | ⚠️ 部分支持 | ✅ 完全支持 |
| 速度 | 毫秒级 | 秒级（首次启动） |