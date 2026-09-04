# Elasticsearch 容器测试规范

本文档定义使用 Testcontainers Elasticsearch 容器进行 DT 测试的编写规范，适用于需要真实 ES 交互的测试场景。

---

## 一、依赖配置

### 【强制】Maven 依赖

> **推荐**：使用 `testcontainers-bom` 统一管理版本，详见 [pom-dependencies.md](../../assets/config-templates/pom-dependencies.md)

```xml
<!-- Testcontainers 核心 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.16.2</version>
    <scope>test</scope>
</dependency>

<!-- JUnit 5 集成 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.16.2</version>
    <scope>test</scope>
</dependency>

<!-- Elasticsearch 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>1.16.2</version>
    <scope>test</scope>
</dependency>
```

---

## 二、ElasticsearchContainer 配置

### 2.1 ES 8.x 配置（默认启用安全认证）

```java
static final ElasticsearchContainer ES = new ElasticsearchContainer(
    DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
)
.withPassword("changeme")                                // 设置密码
.withEnv("xpack.security.http.ssl.enabled", "false");   // 关闭 SSL（测试用）
```

### 2.2 ES 7.x 配置

```java
static final ElasticsearchContainer ES = new ElasticsearchContainer(
    DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:7.17.15")
);
```

### 2.3 常用配置项

| 方法 | 用途 | 示例 |
|------|------|------|
| `withPassword()` | 设置安全认证密码 | `.withPassword("changeme")` |
| `withEnv()` | 设置环境变量 | `.withEnv("xpack.security.http.ssl.enabled", "false")` |
| `withClasspathResourceMapping()` | 映射配置文件到容器 | `.withClasspathResourceMapping("jvm.options", "/usr/share/.../jvm.options.d/custom.options", BindMode.READ_ONLY)` |

### 2.4 自定义 JVM 堆内存

```java
static final ElasticsearchContainer ES = new ElasticsearchContainer(
    DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
)
.withPassword("changeme")
.withEnv("xpack.security.http.ssl.enabled", "false")
.withEnv("ES_JAVA_OPTS", "-Xms512m -Xmx512m");  // 限制堆内存
```

---

## 三、索引初始化

### 3.1 代码方式初始化索引

```java
@Autowired
private ElasticsearchRestTemplate esRestTemplate;

@BeforeEach
void initIndex() {
    // 创建索引映射
    IndexOperations indexOps = esRestTemplate.indexOps(UserDocument.class);
    if (indexOps.exists()) {
        indexOps.delete();
    }
    indexOps.create();
    indexOps.putMapping(indexOps.createMapping(UserDocument.class));
}
```

### 3.2 JSON 文件方式初始化

将索引映射定义在 `css/fixtures/global/` 目录下：

```java
@BeforeEach
void initIndex() throws Exception {
    String mappingJson = Resources.toString(
        Resources.getResource("css/fixtures/global/init-schema.json"), StandardCharsets.UTF_8);

    RestHighLevelClient client = esRestTemplate.getClient();
    // 创建索引
    CreateIndexRequest request = new CreateIndexRequest("user_index");
    request.source(mappingJson, XContentType.JSON);
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

### 3.3 Fixture 三层组织

| 层级 | 目录 | 变更频率 | 用途 |
|------|------|---------|------|
| 全局 | `fixtures/global/` | 低 | 索引映射 + 基础文档 |
| 模块 | `fixtures/modules/` | 中 | 按业务模块组织的可复用场景 |
| 用例 | `fixtures/tests/` | 高 | 特定测试专用文档 |

**命名规范**：

| 文件命名 | 用途 |
|---------|------|
| `init-schema.json` | 初始化索引映射 |
| `cleanup-schema.json` | 删除索引 |
| `init-{entity}-data.json` | 初始化实体文档 |
| `cleanup-{entity}-data.json` | 清理实体文档 |

### 3.4 索引映射 JSON 示例

```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "tokenizer": {
        "ik_smart_tokenizer": {
          "type": "ik_smart"
        }
      },
      "analyzer": {
        "ik_smart_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart_tokenizer"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "userId": { "type": "keyword" },
      "userName": {
        "type": "text",
        "analyzer": "ik_smart_analyzer",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "email": { "type": "keyword" },
      "createdAt": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss||epoch_millis" }
    }
  }
}
```

---

## 四、分词器插件

### 4.1 自定义镜像（含 IK 分词器）

当需要使用 IK 分词器等插件时，需构建自定义 ES 镜像。

```dockerfile
# css/testcontainers/image/Dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:8.12.0

# 安装 IK 分词器
RUN elasticsearch-plugin install --batch https://github.com/infinilabs/analysis-ik/releases/download/v8.12.0/elasticsearch-analysis-ik-8.12.0.zip

# 复制初始化脚本
COPY init/ /docker-entrypoint-initdb.d/
```

### 4.2 使用自定义镜像

```java
static final ElasticsearchContainer ES = new ElasticsearchContainer(
    DockerImageName.parse("my-custom-es:8.12.0")
)
.withPassword("changeme")
.withEnv("xpack.security.http.ssl.enabled", "false");
```

### 4.3 分词器目录结构

```
css/testcontainers/image/
├── Dockerfile
├── build.sh
├── README.md
├── init/                    # 初始化脚本
└── tokenizer/               # 分词器插件
    └── elasticsearch-analysis-ik-8.12.0.zip
```

---

## 五、SSL/认证处理

### 5.1 关闭 SSL（推荐用于测试）

```java
static final ElasticsearchContainer ES = new ElasticsearchContainer(
    DockerImageName.parse("docker.elastic.co/elasticsearch/elasticsearch:8.12.0")
)
.withPassword("changeme")
.withEnv("xpack.security.http.ssl.enabled", "false");  // 关闭 SSL
```

### 5.2 使用 SSL（需要与生产一致时）

```java
// ES 8.x 默认启用 HTTPS，需要获取 CA 证书
SSLContext sslContext = ES.createSslContextFromCa();

// 配置 RestHighLevelClient 使用 SSL
RestClientBuilder builder = RestClient.builder(
    HttpHost.create(ES.getHttpHostAddress())
).setHttpClientConfigCallback(httpClientBuilder ->
    httpClientBuilder.setSSLContext(sslContext)
);
```

---

## 六、完整示例

### 6.1 ES 容器基类

```java
package com.example.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.elasticsearch.ElasticsearchContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

/**
 * Elasticsearch 容器级 DT 基类
 */
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
    }

    @DynamicPropertySource
    static void esProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.elasticsearch.uris", ES::getHttpHostAddress);
        registry.add("spring.elasticsearch.username", () -> "elastic");
        registry.add("spring.elasticsearch.password", () -> "changeme");
    }
}
```

### 6.2 ES 搜索测试

```java
package com.example.search;

import com.example.dt.AbstractElasticsearchTestcontainers;
import com.example.document.UserDocument;
import com.example.repository.UserSearchRepository;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.elasticsearch.core.ElasticsearchRestTemplate;
import org.springframework.data.elasticsearch.core.IndexOperations;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * 用户搜索容器级 DT 测试
 */
class UserSearchServiceTest extends AbstractElasticsearchTestcontainers {

    @Autowired
    private ElasticsearchRestTemplate esRestTemplate;

    @Autowired
    private UserSearchRepository userSearchRepository;

    @BeforeEach
    void setUp() {
        // 重建索引
        IndexOperations indexOps = esRestTemplate.indexOps(UserDocument.class);
        if (indexOps.exists()) {
            indexOps.delete();
        }
        indexOps.create();
        indexOps.putMapping(indexOps.createMapping(UserDocument.class));

        // 插入测试数据
        userSearchRepository.save(createUserDoc("user_001", "张三", "zhangsan@test.com"));
        userSearchRepository.save(createUserDoc("user_002", "李四", "lisi@test.com"));
        userSearchRepository.save(createUserDoc("user_003", "张伟", "zhangwei@test.com"));
    }

    @AfterEach
    void tearDown() {
        userSearchRepository.deleteAll();
    }

    @Test
    @DisplayName("正常用例-按姓名搜索返回匹配结果")
    void searchByName_matchingKeyword_returnResults() {
        // When
        List<UserDocument> results = userSearchRepository.findByUserName("张");

        // Then：验证真实 ES 搜索结果
        assertThat(results).isNotEmpty();
        assertThat(results).allSatisfy(doc ->
            assertThat(doc.getUserName()).contains("张")
        );
    }

    @Test
    @DisplayName("正常用例-搜索无匹配返回空列表")
    void searchByName_noMatch_returnEmptyList() {
        // When
        List<UserDocument> results = userSearchRepository.findByUserName("不存在的名字");

        // Then
        assertThat(results).isEmpty();
    }

    private UserDocument createUserDoc(String userId, String name, String email) {
        UserDocument doc = new UserDocument();
        doc.setUserId("DT_TEST_" + userId);
        doc.setUserName(name);
        doc.setEmail(email);
        return doc;
    }
}
```

---

## 七、CSS（Elasticsearch）目录结构

```
css/
├── fixtures/
│   ├── global/
│   │   ├── init-schema.json           # 索引映射定义
│   │   └── init-user-data.json        # 基础用户文档
│   ├── modules/
│   │   └── search-user-data.json      # 搜索模块测试数据
│   └── tests/
│       └── single-record-data.json    # 单条记录边界场景
├── migrations/
│   ├── v1/                            # v1 版本索引结构
│   └── v2/                            # v2 版本索引结构变更
├── seeds/
│   └── dev_data.json                  # 开发环境种子数据
└── testcontainers/
    └── image/
        ├── Dockerfile                 # 自定义 ES 镜像（含分词器）
        ├── build.sh
        ├── README.md
        ├── init/                      # 初始化脚本
        └── tokenizer/                 # 分词器插件 zip 包
```