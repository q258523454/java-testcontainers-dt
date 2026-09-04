# 容器级 DT 通用基类（MySQL + Redis + Elasticsearch）

> **适用场景**：测试类同时需要 MySQL、Redis、Elasticsearch 三个中间件容器
>
> **如仅需单一中间件**：使用 [AbstractMySQLTestcontainers.md](./AbstractMySQLTestcontainers.md)、[AbstractElasticsearchTestcontainers.md](./AbstractElasticsearchTestcontainers.md)、[AbstractRedisTestcontainers.md](./AbstractRedisTestcontainers.md)

本文基类提供三种容器生命周期模式，对应 [TestcontainersDT核心规范](../../references/TestcontainersDT核心规范.md) 2.1 节三种生命周期模式对比。

| 模式 | 启动时机 | 停止时机 | 清理可靠性 | 适用场景 | 性能 |
|------|---------|---------|-----------|---------|------|
| **Singleton** | 基类首次加载 | JVM 退出时 | ⚠️ 依赖 Ryuk | 多测试类共享（推荐） | ⭐⭐⭐⭐⭐ |
| **@Container** | beforeAll | afterAll 后 | ✅ JUnit 主动清理 | 单测试类共享 | ⭐⭐⭐⭐ |
| **Reuse** | 首次 start() | 不停止 | ❌ 需手动清理 | 本地快速迭代 | ⭐⭐⭐⭐⭐（复用后） |

## 设计要点

- **@DynamicPropertySource**：自动注入所有中间件属性
- **@Testcontainers(disabledWithoutDocker = true)**：Docker 不可用时跳过
- **Startables.deepStart**：Singleton 和 Reuse 模式下并行启动所有容器，比串行快数倍
- **@Container 模式注意**：JUnit 扩展按字段声明顺序串行启动容器，无法并行。如需并行启动，使用 Singleton 或 Reuse 模式

---

## 模式一：Singleton Container（推荐）

> 容器在 `static` 块中启动一次，所有子类共享。不调用 `stop()`，清理依赖 Ryuk。详见规范 2.2 节。

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.lifecycle.Startables;
import org.testcontainers.utility.DockerImageName;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractTestcontainersTestcontainers {

    // ==================== 容器定义（Singleton） ====================

    protected static final MySQLContainer<?> MYSQL;

    protected static final GenericContainer<?> REDIS;

    protected static final ElasticsearchContainer ES;

    static {
        // MySQL 容器
        MYSQL = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("mysql/testcontainers/init_charset.sql");

        // Redis 容器
        REDIS = new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);

        // Elasticsearch 容器
        ES = new ElasticsearchContainer(
            DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
        )
        .withPassword("changeme")
        .withEnv("xpack.security.http.ssl.enabled", "false")
        .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");

        // 并行启动所有容器
        Startables.deepStart(MYSQL, REDIS, ES).join();
        // 不调用 stop()：由 Ryuk 在 JVM 退出时自动清理
    }

    // ==================== 属性注入 ====================

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
        registry.add("spring.elasticsearch.username", () -> "elastic");
        registry.add("spring.elasticsearch.password", () -> "changeme");
    }
}
```

---

## 模式二：@Container 注解

> JUnit 5 扩展自动管理 `@Container` 标注的容器。`static` 字段为类级共享，测试结束后 JUnit 主动调用 `stop()` 完全删除容器。详见规范 2.3 节。
>
> **局限性**：`@Container static` 容器仅在单个测试类内共享，无法跨多个测试类复用。如需跨类共享，使用 Singleton 模式。
>
> **多容器注意**：JUnit 扩展按字段声明顺序串行启动容器。如需并行启动（`Startables.deepStart`），使用 Singleton 或 Reuse 模式。

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractTestcontainersTestcontainers {

    // ==================== 容器定义（@Container） ====================

    // static 字段：类级共享（推荐）
    // JUnit 扩展自动管理 start()/stop()，测试结束后容器被完全删除
    // 注意：多容器按声明顺序串行启动，非并行
    @Container
    static MySQLContainer<?> MYSQL = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("mysql/testcontainers/init_charset.sql");

    @Container
    static GenericContainer<?> REDIS = new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
        .withExposedPorts(6379);

    @Container
    static ElasticsearchContainer ES = new ElasticsearchContainer(
        DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
    )
    .withPassword("changeme")
    .withEnv("xpack.security.http.ssl.enabled", "false")
    .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");

    // ==================== 属性注入 ====================

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
        registry.add("spring.elasticsearch.username", () -> "elastic");
        registry.add("spring.elasticsearch.password", () -> "changeme");
    }
}
```

---

## 模式三：Reuse 容器复用

> 通过 `withReuse(true)` 跨 JVM 复用同一容器。首次运行创建新容器，后续运行通过 hash 匹配直接复用，跳过创建和启动。详见规范 2.4 节。
>
> **⚠️ 实验性功能**：复用容器未注册到 ResourceReaper，测试结束后不自动停止。需手动 `docker rm` 或 Docker 重启清理。**不适合 CI 环境**。
>
> **启用条件**（缺一不可）：
> 1. 每个容器声明 `.withReuse(true)`
> 2. `~/.testcontainers.properties` 中配置 `testcontainers.reuse.enable=true`
>
> **classpath 配置绕过**：Testcontainers 默认仅从 `~/.testcontainers.properties` 读取复用开关，防止 CI 意外启用。如需从 classpath `testcontainers.properties` 读取，通过 `TestcontainersConfiguration` 编程注入。

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.lifecycle.Startables;
import org.testcontainers.utility.DockerImageName;
import org.testcontainers.utility.TestcontainersConfiguration;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractTestcontainersTestcontainers {

    // ==================== 容器定义（Reuse） ====================

    protected static final MySQLContainer<?> MYSQL;

    protected static final GenericContainer<?> REDIS;

    protected static final ElasticsearchContainer ES;

    static {
        // 从 classpath testcontainers.properties 读取 reuse 配置
        // Testcontainers 默认仅从 ~/.testcontainers.properties 读取复用开关
        // 通过 updateUserConfig() 编程注入，绕过此限制
        TestcontainersConfiguration tcConfig = TestcontainersConfiguration.getInstance();
        String classpathReuse = tcConfig.getClasspathProperties()
            .getProperty("testcontainers.reuse.enable", "false");
        if ("true".equalsIgnoreCase(classpathReuse)) {
            tcConfig.updateUserConfig("testcontainers.reuse.enable", "true");
        }

        // MySQL 容器
        MYSQL = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
            .withReuse(true)
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("mysql/testcontainers/init_charset.sql");

        // Redis 容器
        REDIS = new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
            .withReuse(true)
            .withExposedPorts(6379);

        // Elasticsearch 容器
        ES = new ElasticsearchContainer(
            DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
        )
        .withReuse(true)
        .withPassword("changeme")
        .withEnv("xpack.security.http.ssl.enabled", "false")
        .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");

        // 并行启动所有容器
        // 首次：创建并启动；后续：hash 匹配后直接复用，跳过创建和启动
        Startables.deepStart(MYSQL, REDIS, ES).join();
        // 不调用 stop()：复用容器未注册到 ResourceReaper，跨 JVM 持续运行
    }

    // ==================== 属性注入 ====================

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
        registry.add("spring.elasticsearch.username", () -> "elastic");
        registry.add("spring.elasticsearch.password", () -> "changeme");
    }
}
```

> **classpath 配置文件**：在 `src/test/resources/testcontainers.properties` 中添加：
> ```properties
> # 容器复用开关（withReuse(true) 跨 JVM 复用的前提条件）
> testcontainers.reuse.enable=true
> ```

> **⚠️ withInitScript 与 Reuse**：`withInitScript` 仅在容器首次创建时执行。复用模式下后续运行不会重新执行初始化脚本，测试需在 `@BeforeEach`/`@AfterEach` 中自行管理数据隔离。

> **⚠️ Redis/ES 数据与 Reuse**：复用模式下 Redis 和 ES 中的数据跨 JVM 持久存在。测试需在 `@BeforeEach`/`@AfterEach` 中通过统一前缀管理 Key（Redis）和索引生命周期（ES），避免数据污染。

---

## 使用方式

1. 选择合适的生命周期模式，继承对应基类
2. 测试类添加 `@Testcontainers(disabledWithoutDocker = true)`（@Container 模式必须）
3. 按需 `@Autowired` 注入 Repository/Service
4. `@BeforeEach` 准备数据，`@AfterEach` 清理数据
