# Testcontainers DT 核心规范

本文档定义容器级 DT（Developer Testing）的核心规范，涵盖容器生命周期、基类设计、属性注入、数据隔离和反模式。

---

## 一、核心原则

### 【强制】容器级 DT 的本质

容器级 DT 使用**真实中间件容器**验证代码行为，而非 Mock 或嵌入式替代品。核心价值：

- **SQL 语法正确性**：Mock 不执行真实 SQL，SQL 错误在 UT 中无法发现
- **JPA 映射正确性**：Entity → Table 字段映射在 H2 和 MySQL 间存在差异
- **序列化/反序列化**：Redis/ES 的真实序列化行为与 Mock 完全不同
- **事务边界正确性**：`@Transactional` 回滚/提交行为依赖真实数据库
- **迁移脚本验证**：Flyway/Liquibase 脚本必须在与生产一致的数据库上执行

### 【强制】遵守 AIR 原则（容器级适配）

| 原则 | 容器级 DT 适配 |
|------|-------------|
| Automatic（自动化） | 容器自动启动/停止，无需人工干预 |
| Independent（独立性） | 每个测试方法独立准备和清理数据，容器共享但数据隔离 |
| Repeatable（可重复） | 容器每次启动状态一致，测试结果可重复 |

### 【强制】遵守 FIRST 原则（容器级适配）

| 原则 | 容器级 DT 适配 |
|------|-------------|
| Fast（快速） | 使用 Singleton Container 避免重复启动；使用 Alpine 镜像 |
| Independent（独立性） | @BeforeEach/@AfterEach 数据隔离，统一前缀清理 |
| Repeatable（可重复） | withInitScript 确保初始状态一致 |
| Self-Validating（自验证） | AssertJ 断言，禁止 System.out |
| Timely（及时） | 与开发同步编写 |

---

## 二、容器生命周期模式

### 2.1 三种生命周期模式对比

| 模式 | 启动时机 | 停止时机 | 清理机制 | 清理可靠性 | 适用场景 | 性能 |
|------|---------|---------|---------|-----------|---------|------|
| **Singleton（static 块）** | 基类首次加载 | JVM 正常退出时 | Ryuk 监控 JVM socket + JVM shutdown hook | ⚠️ JVM 非正常退出（`kill -9`、IDE 强制终止）时容器残留 | 多测试类共享容器（**推荐**） | ⭐⭐⭐⭐⭐ |
| **@Container 注解** | static：beforeAll；实例：beforeEach | static：afterAll 后；实例：afterEach 后 | JUnit 扩展显式调用 `stop()`（kill + remove + remove volumes） | ✅ 测试结束后立即清理，不依赖 Ryuk | static：类级共享；实例：方法级隔离 | static：⭐⭐⭐⭐；实例：⭐⭐ |
| **Reuse（容器复用）** | 首次 `start()` 创建；后续 hash 匹配复用已运行容器 | 测试结束后**不停止** | 无自动清理（未注册 ResourceReaper） | ❌ 需手动 `docker rm` 或 Docker 重启 | 本地开发快速迭代（**不适用 CI**） | ⭐⭐⭐⭐⭐（复用后） |

### 2.2 Singleton Container 模式
特点：整个启动，所有测试类内共享，JVM退出才会清理和关闭容器；

容器在基类 `static` 块中启动一次，所有子类共享。容器**不会显式调用 `stop()`**，清理依赖以下机制：

1. **Ryuk 容器（默认）**：Testcontainers 启动一个独立的 Ryuk 容器，挂载 Docker socket，监控 JVM socket 连接。JVM 正常退出时，Ryuk 检测到连接断开，自动清理所有 Testcontainers 创建的容器、网络和卷。
2. **JVM shutdown hook（Ryuk 禁用时）**：`TESTCONTAINERS_RYUK_DISABLED=true` 时，Testcontainers 注册 JVM shutdown hook，在 JVM 退出时执行清理。


```java
public abstract class AbstractTestcontainersTestcontainers {

    static final MySQLContainer<?> MYSQL;

    static {
        MYSQL = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("mysql/testcontainers/init_charset.sql");
        MYSQL.start();
        // 容器已注册到 ResourceReaper，由 Ryuk 在 JVM 退出时清理
        // 无需手动调用 stop() 或注册 shutdown hook
    }
}
```


### 2.3 @Container 注解模式
特点：JUnit 5 扩展自动管理生命周期，通过字段修饰符控制共享范围；

JUnit 5 扩展（`@Testcontainers`）自动管理 `@Container` 标注的容器。**同一模式，两种执行策略**——字段修饰符决定共享范围：

| 字段类型 | 启动/停止 | 共享范围 | 适用场景 | 性能 |
|---------|---------|---------|---------|------|
| `static` 字段 | beforeAll / afterAll 后 | 类级共享 | 单个测试类共享（**推荐**） | ⭐⭐⭐⭐ |
| 实例字段 | beforeEach / afterEach 后 | 方法级隔离 | 需要隔离的测试 | ⭐⭐ |

`stop()` 的完整行为：`kill` 容器进程 → `removeContainer(withRemoveVolumes=true, withForce=true)` → 清理关联卷。容器被**完全删除**，不会残留。

```java
@Testcontainers
class XxxServiceImplTest {

    // 策略1：static 字段 — 类级共享（推荐）
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    // 策略2：实例字段 — 方法级隔离（DT 场景避免使用）
    // @Container
    // MySQLContainer<?> mysql = new MySQLContainer<>(...);
}
```

> **与 Singleton 模式的关键区别**：`@Container` 的清理由 JUnit 扩展**主动执行**，不依赖 Ryuk 或 JVM shutdown hook。即使 JVM 被强制终止，只要测试类正常执行完毕，容器已被 `stop()` 清理。

> **局限性**：`@Container static` 的容器**仅在单个测试类内共享**，无法跨多个测试类复用。如需跨类共享，必须使用 Singleton 模式。


### 2.4 Reuse 容器复用模式
特点：跨 JVM 复用同一容器，后续运行跳过容器创建和启动；

通过 `withReuse(true)` 声明容器可复用。首次运行创建新容器，后续运行通过容器配置的 SHA1 hash 匹配已运行的容器并直接复用，跳过镜像拉取、容器创建和启动。

**启用条件**（缺一不可）：
1. 容器声明 `.withReuse(true)`
2. `~/.testcontainers.properties` 中配置 `testcontainers.reuse.enable=true`

```java
static {
    mysql = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
        .withReuse(true)
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    mysql.start();
    // 首次：创建并启动容器；后续：hash 匹配后直接复用，跳过创建和启动
}
```

> **⚠️ classpath 配置不生效**：Testcontainers 有意仅从 `~/.testcontainers.properties` 读取复用开关，防止 CI 意外启用。如需从 classpath 读取，可通过 `TestcontainersConfiguration` 编程注入：
> ```java
> TestcontainersConfiguration tcConfig = TestcontainersConfiguration.getInstance();
> String reuse = tcConfig.getClasspathProperties().getProperty("testcontainers.reuse.enable", "false");
> if ("true".equalsIgnoreCase(reuse)) {
>     tcConfig.updateUserConfig("testcontainers.reuse.enable", "true");
> }
> ```

> **与 Singleton 模式的关键区别**：复用容器**未注册到 ResourceReaper**，测试结束后不自动停止。容器跨 JVM 持续运行，直到手动 `docker rm` 或 Docker 重启。后续测试运行几乎零启动开销，但需手动管理清理。

> **⚠️ 实验性功能**（`@UnstableAPI`）：资源清理和网络功能未完全支持。**不适合 CI 环境**——容器持续累积导致磁盘耗尽。


## 三、@DynamicPropertySource 属性注入

### 【强制】使用 @DynamicPropertySource 注入容器属性

Spring Boot 2.4+ 提供 `@DynamicPropertySource`，是 Testcontainers 属性注入的**推荐方式**。

```java
@DynamicPropertySource
static void containerProperties(DynamicPropertyRegistry registry) {
    // MySQL
    registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
    registry.add("spring.datasource.username", MYSQL::getUsername);
    registry.add("spring.datasource.password", MYSQL::getPassword);
    // Redis
    registry.add("spring.data.redis.host", REDIS::getHost);
    registry.add("spring.data.redis.port", REDIS::getFirstMappedPort);
    // Elasticsearch
    registry.add("spring.elasticsearch.uris", ES::getHttpHostAddress);
}
```

### 多容器属性注入

在同一个 `@DynamicPropertySource` 方法中注入所有中间件属性：

```java
@DynamicPropertySource
static void containerProperties(DynamicPropertyRegistry registry) {
    Startables.deepStart(MYSQL, REDIS, ES).join();  // 先启动

    registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
    registry.add("spring.datasource.username", MYSQL::getUsername);
    registry.add("spring.datasource.password", MYSQL::getPassword);
    registry.add("spring.data.redis.host", REDIS::getHost);
    registry.add("spring.data.redis.port", REDIS::getFirstMappedPort);
    registry.add("spring.elasticsearch.uris", ES::getHttpHostAddress);
}
```

---

## 四、Spring Boot 测试注解选择

### 【推荐】DT 测试注解组合

```java
// 模式1：纯上下文 DT（最常用，不启动 Web 服务器）
@SpringBootTest(classes = XxxApplication.class, webEnvironment = NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
class XxxServiceImplTest extends AbstractTestcontainersTestcontainers { ... }

// 模式2：Web 端到端 DT（需要测试 HTTP 端点）
@SpringBootTest(classes = XxxApplication.class, webEnvironment = RANDOM_PORT)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
class XxxControllerTest extends AbstractTestcontainersTestcontainers { ... }

// 模式3：JPA 层 DT（仅测试 Repository）
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("testcontainers")
class XxxRepositoryTest extends AbstractMySQLTestcontainers { ... }
```

---

## 五、数据隔离策略

### 【强制】容器共享、数据隔离

容器在测试类间共享，但每个测试方法的数据必须独立。三种策略：


### 策略1：@Sql 注解【推荐】

```java
@Test
@Sql(scripts = {
    "classpath:data/mysql/fixtures/global/schema.sql",
    "classpath:data/mysql/fixtures/tests/single_record.sql"
}, executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "classpath:data/mysql/fixtures/tests/cleanup_single_record.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void findById_existingUser_returnUser() { ... }
```


### 策略2：@BeforeEach/@AfterEach 清理【推荐】

```java
@BeforeEach
void setUp() {
    User testUser = createTestUser("DT_TEST_user_001");
    userRepository.save(testUser);
}

@AfterEach
void tearDown() {
    userRepository.deleteByUserIdStartingWith("DT_TEST_");
}
```
### 策略3：TRUNCATE 全表清理

```java
@Autowired
protected JdbcTemplate jdbcTemplate;

@BeforeEach
void cleanUp() {
    jdbcTemplate.execute("SET FOREIGN_KEY_CHECKS = 0");
    jdbcTemplate.execute("TRUNCATE TABLE t_user");
    jdbcTemplate.execute("TRUNCATE TABLE t_order");
    jdbcTemplate.execute("SET FOREIGN_KEY_CHECKS = 1");
}
```

### 【强制】测试数据使用统一前缀

```java
private User createTestUser(String userId, String userName) {
    User user = new User();
    user.setUserId("DT_TEST_" + userId);  // 统一前缀
    user.setUserName(userName);
    return user;
}

@AfterEach
void tearDown() {
    userRepository.deleteByUserIdStartingWith("DT_TEST_");
}
```

---

## 六、DT vs UT 决策树

```
需要编写测试
├─ 是否涉及数据库/Redis/ES 操作？
│  ├─ YES → 是否需要验证 SQL/序列化/映射正确性？
│  │  ├─ YES → 是否需要与生产完全一致的环境？
│  │  │  ├─ YES → Testcontainers DT（真实 MySQL/ES/Redis 容器）
│  │  │  └─ NO → Embedded DT（H2/jedis-mock）
│  │  └─ NO → UT（Mock Repository/RedisService）
│  └─ NO → 是否涉及外部 HTTP 调用？
│     ├─ YES → DT（WireMock 真实 HTTP 模拟）
│     └─ NO → UT（Mockito Mock 依赖）
│
├─ 是否需要验证完整业务流程？
│  ├─ YES → DT（@SpringBootTest + 真实基础设施）
│  └─ NO → UT（方法级 Mock 测试）
│
└─ 是否需要验证数据库迁移脚本？
   ├─ YES → Testcontainers DT（migrations 目录 + 真实数据库）
   └─ NO → UT
```

---

## 七、DT 覆盖范围

### DT 应覆盖 UT 无法覆盖的场景

| 覆盖场景 | UT | Embedded DT | Testcontainers DT |
|---------|-----|-------------|------------------|
| SQL 语法正确性 | ❌ | ⚠️ H2 兼容性 | ✅ 真实 MySQL |
| JPA 映射正确性 | ❌ | ⚠️ H2 差异 | ✅ 真实 MySQL |
| 事务边界正确性 | ❌ | ⚠️ | ✅ |
| Redis 序列化/反序列化 | ❌ | ⚠️ jedis-mock | ✅ 真实 Redis |
| Redis TTL 过期机制 | ❌ | ⚠️ | ✅ |
| Elasticsearch 索引映射 | ❌ | ❌ | ✅ 真实 ES |
| ES 分词器配置 | ❌ | ❌ | ✅ 自定义镜像 |
| 完整业务流程 | ❌ | ✅ | ✅ |
| 数据库迁移脚本 | ❌ | ❌ | ✅ 与生产一致 |


---

## 八、反模式

### 【强制】禁止每个测试方法启动/停止容器

```java
// ❌ 每个测试方法创建新容器（启动耗时 5-10 秒）
@Test
void test1() {
    try (MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")) {
        mysql.start();  // 5-10 秒
    }
}

// ✅ static 容器，所有方法共享
@Container
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
```

### 【强制】禁止测试间数据未清理

```java
// ❌ 测试 A 插入的数据影响测试 B
@Test
void testA_insertUser() {
    userRepository.save(new User("user_001", "张三"));
    // 未清理
}

// ✅ 每个测试前后清理数据
@AfterEach
void tearDown() {
    userRepository.deleteByUserIdStartingWith("DT_TEST_");
}
```

### 【强制】禁止在 DT 中 Mock 容器化基础设施

```java
// ❌ 在 DT 中 Mock RedisService，失去真实验证
@Mock
private RedisService redisService;
when(redisService.get("key")).thenReturn(value);

// ✅ 使用真实 Redis 容器
@Autowired
private RedisTemplate<String, Object> redisTemplate;
redisTemplate.opsForValue().set("key", value);
Object result = redisTemplate.opsForValue().get("key");
assertThat(result).isEqualTo(value);
```

### 【强制】禁止硬编码端口

```java
// ❌ 硬编码端口，并发执行时冲突
MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
    .withFixedExposedPort(3306);

// ✅ 使用动态端口
MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
int port = mysql.getFirstMappedPort();
```

### 【强制】禁止在 DT 中过度断言 Mock 行为

```java
// ❌ DT 应验证业务结果，而非 Mock 调用次数
verify(repository, times(1)).findById(any());

// ✅ DT 验证业务结果 + 真实数据状态
User dbUser = userRepository.findById("DT_TEST_user_001").orElseThrow();
assertThat(dbUser.getLastSearchTime()).isNotNull();
```

### 【强制】Docker 不可用时优雅降级

```java
// ❌ Docker 不可用时测试直接失败
@Testcontainers

// ✅ Docker 不可用时跳过
@Testcontainers(disabledWithoutDocker = true)
```

---

## 九、容器启动顺序与依赖

### dependsOn — 声明式依赖

```java
GenericContainer<?> app = new GenericContainer<>("my-app:latest")
    .dependsOn(mysql)  // app 在 mysql 之后启动
    .withExposedPorts(8080);
```

### Network — 容器间通信

```java
Network network = Network.newNetwork();

KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("apache/kafka:3.7.0"))
    .withNetwork(network)
    .withNetworkAliases("kafka");

GenericContainer<?> app = new GenericContainer<>("my-app:latest")
    .withNetwork(network)
    .dependsOn(kafka);
```

---

## 十、性能优化

### 总表

| 优化项 | 方法 | 效果 | 注意事项 |
|--------|------|------|---------|
| Singleton Container | static 块启动一次 | 避免重复启动 | 清理依赖 Ryuk，JVM 非正常退出时容器残留（见 2.2 节） |
| 减少 ES 堆内存 | `withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m")` | 降低内存占用 | — |
| 容器复用 | `withReuse(true)` + `~/.testcontainers.properties` 中 `testcontainers.reuse.enable=true` | 实验性，跨测试运行复用 | ⚠️ 容器**不会被自动清理**，持续运行直到手动 `docker rm` 或 Docker 重启。不适合 CI 环境 |
| 镜像预拉取 | CI 中提前 `docker pull` | 避免首次拉取延迟 | — |


### 并行启动多个容器

使用 `Startables.deepStart` 并行启动多个容器，比串行快数倍：

```java
static {
    Startables.deepStart(MYSQL, REDIS, ES).join();
}
```


