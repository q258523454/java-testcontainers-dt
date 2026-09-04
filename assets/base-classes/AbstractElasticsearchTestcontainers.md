# Elasticsearch 容器级 DT 基类

> **适用场景**：仅需要 ES 容器的测试
>
> **如需多容器**：使用 [AbstractTestcontainersTestcontainers.md](./AbstractTestcontainersTestcontainers.md)

本文基类提供三种容器生命周期模式，对应 [TestcontainersDT核心规范](../../references/TestcontainersDT核心规范.md) 2.1 节三种生命周期模式对比。

| 模式 | 启动时机 | 停止时机 | 清理可靠性 | 适用场景 | 性能 |
|------|---------|---------|-----------|---------|------|
| **Singleton** | 基类首次加载 | JVM 退出时 | ⚠️ 依赖 Ryuk | 多测试类共享（推荐） | ⭐⭐⭐⭐⭐ |
| **@Container** | beforeAll | afterAll 后 | ✅ JUnit 主动清理 | 单测试类共享 | ⭐⭐⭐⭐ |
| **Reuse** | 首次 start() | 不停止 | ❌ 需手动清理 | 本地快速迭代 | ⭐⭐⭐⭐⭐（复用后） |

---

## 模式一：Singleton Container（推荐）

> 容器在 `static` 块中启动一次，所有子类共享。不调用 `stop()`，清理依赖 Ryuk。详见规范 2.2 节。

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractElasticsearchTestcontainers {

    protected static final ElasticsearchContainer ES;

    static {
        ES = new ElasticsearchContainer(
            DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
        )
        .withPassword("changeme")
        .withEnv("xpack.security.http.ssl.enabled", "false")
        .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");
        ES.start();
        // 不调用 stop()：由 Ryuk 在 JVM 退出时自动清理
    }

    @DynamicPropertySource
    static void esProperties(DynamicPropertyRegistry registry) {
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

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractElasticsearchTestcontainers {

    // static 字段：类级共享（推荐）
    // JUnit 扩展自动管理 start()/stop()，测试结束后容器被完全删除
    @Container
    static ElasticsearchContainer ES = new ElasticsearchContainer(
        DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
    )
    .withPassword("changeme")
    .withEnv("xpack.security.http.ssl.enabled", "false")
    .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");

    @DynamicPropertySource
    static void esProperties(DynamicPropertyRegistry registry) {
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
> 1. 容器声明 `.withReuse(true)`
> 2. `~/.testcontainers.properties` 中配置 `testcontainers.reuse.enable=true`
>
> **classpath 配置绕过**：Testcontainers 默认仅从 `~/.testcontainers.properties` 读取复用开关，防止 CI 意外启用。如需从 classpath `testcontainers.properties` 读取，通过 `TestcontainersConfiguration` 编程注入。

```java
package {你的基础包}.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;
import org.testcontainers.utility.TestcontainersConfiguration;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractElasticsearchTestcontainers {

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

        // withReuse(true)：跨 JVM 复用容器
        // 首次运行创建并启动；后续 hash 匹配后直接复用，跳过创建和启动
        ES = new ElasticsearchContainer(
            DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
        )
        .withReuse(true)
        .withPassword("changeme")
        .withEnv("xpack.security.http.ssl.enabled", "false")
        .withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");
        ES.start();
        // 不调用 stop()：复用容器未注册到 ResourceReaper，跨 JVM 持续运行
    }

    @DynamicPropertySource
    static void esProperties(DynamicPropertyRegistry registry) {
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

> **⚠️ ES 索引与 Reuse**：复用模式下 ES 中的索引和数据跨 JVM 持久存在。测试需在 `@BeforeEach`/`@AfterEach` 中管理索引生命周期（创建/删除），避免数据污染。

---

## 使用方式

1. 选择合适的生命周期模式，继承对应基类
2. 测试类添加 `@Testcontainers(disabledWithoutDocker = true)`（@Container 模式必须）
3. 按需 `@Autowired` 注入 ElasticsearchOperations/RestHighLevelClient
4. `@BeforeEach` 准备索引和数据，`@AfterEach` 清理索引
