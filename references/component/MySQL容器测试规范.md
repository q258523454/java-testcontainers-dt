# MySQL 容器测试规范

本文档定义使用 Testcontainers MySQL 容器进行 DT 测试的编写规范，适用于需要真实 MySQL 数据库交互的测试场景。

---

## 一、依赖配置

### 【强制】Maven 依赖

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

<!-- MySQL 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <version>1.16.2</version>
    <scope>test</scope>
</dependency>
```

---

## 二、MySQLContainer 配置

### 2.1 基础配置

```java
static final MySQLContainer<?> MYSQL = new MySQLContainer<>(
    DockerImageName.parse("mysql:8.0"))
    .withDatabaseName("testdb")
    .withUsername("test")
    .withPassword("test");
```

### 2.2 常用配置项

| 方法 | 用途 | 示例 |
|------|------|------|
| `withDatabaseName()` | 设置数据库名 | `.withDatabaseName("testdb")` |
| `withUsername()` | 设置用户名 | `.withUsername("test")` |
| `withPassword()` | 设置密码 | `.withPassword("test")` |
| `withInitScript()` | 容器启动时执行初始化 SQL | `.withInitScript("mysql/testcontainers/init_charset.sql")` |
| `withInitScripts()` | 多脚本按序执行 | `.withInitScripts("a.sql", "b.sql")` |
| `withConfigurationOverride()` | 自定义 my.cnf | `.withConfigurationOverride("db/mysql_conf_override")` |
| `withUrlParam()` | JDBC URL 参数 | `.withUrlParam("useSSL", "false")` |
| `withCommand()` | 覆盖启动命令 | `.withCommand("mysqld --character-set-server=utf8mb4")` |
| `withLogConsumer()` | 日志输出 | `.withLogConsumer(new Slf4jLogConsumer(logger))` |

### 2.3 字符集配置

```java
static final MySQLContainer<?> MYSQL = new MySQLContainer<>(
    DockerImageName.parse("mysql:8.0"))
    .withDatabaseName("testdb")
    .withUsername("test")
    .withPassword("test")
    .withCommand(
        "mysqld",
        "--character-set-server=utf8mb4",
        "--collation-server=utf8mb4_unicode_ci"
    );
```

或通过初始化脚本（`mysql/testcontainers/init_charset.sql`）：

```sql
SET NAMES utf8mb4;
SET CHARACTER SET utf8mb4;
ALTER DATABASE testdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

---

## 三、初始化脚本加载

### 3.1 withInitScript — 容器启动时执行

脚本在容器启动后、测试执行前自动运行。

```java
new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
    .withDatabaseName("testdb")
    .withInitScript("mysql/testcontainers/init_charset.sql");
```

### 3.2 withInitScripts — 多脚本按序执行

```java
new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
    .withDatabaseName("testdb")
    .withInitScripts(
        "mysql/testcontainers/init_charset.sql",   // 1. 字符集设置
        "mysql/fixtures/global/schema.sql"          // 2. 建表
    );
```

### 3.3 @Sql 注解 — 测试方法级数据加载

```java
@Test
@Sql(scripts = {
    "classpath:data/mysql/fixtures/global/schema.sql",       // 全局：建表
    "classpath:data/mysql/fixtures/modules/user_data.sql",   // 模块：用户数据
    "classpath:data/mysql/fixtures/tests/single_record.sql"  // 用例：特定场景
})
void findById_existingUser_returnUser() { ... }
```

### 3.4 Fixture 三层组织

| 层级 | 目录 | 变更频率 | 用途 |
|------|------|---------|------|
| 全局 | `fixtures/global/` | 低 | 建表 + 基础数据 |
| 模块 | `fixtures/modules/` | 中 | 按业务模块组织的可复用场景 |
| 用例 | `fixtures/tests/` | 高 | 特定测试专用数据 |

**命名规范**：

| 文件命名 | 用途 |
|---------|------|
| `init-schema.sql` | 初始化表结构 |
| `cleanup-schema.sql` | 清理表结构 |
| `init-{entity}-data.sql` | 初始化实体数据 |
| `cleanup-{entity}-data.sql` | 清理实体数据 |

---

## 四、自定义镜像

当需要预装插件、配置或大量初始化数据时，使用自定义 Docker 镜像。

### 4.1 Dockerfile

```dockerfile
# mysql/testcontainers/image/Dockerfile
FROM mysql:8.0

# 字符集配置
COPY custom.cnf /etc/mysql/conf.d/

# 初始化脚本（docker-entrypoint-initdb.d 下的脚本容器启动时自动执行）
COPY init.sql /docker-entrypoint-initdb.d/01-init.sql
COPY data.sql /docker-entrypoint-initdb.d/02-data.sql
```

### 4.2 使用自定义镜像

```java
static final MySQLContainer<?> MYSQL = new MySQLContainer<>(
    DockerImageName.parse("my-custom-mysql:latest")
        .asCompatibleSubstituteFor("mysql")
)
.withDatabaseName("testdb")
.withUsername("test")
.withPassword("test");
```


---

## 五、完整示例

### 5.1 仅 MySQL 容器基类

```java
package com.example.dt;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

/**
 * MySQL 容器级 DT 基类
 *
 * 特点：
 * - Singleton Container：所有子类共享同一 MySQL 容器
 * - @DynamicPropertySource：自动注入数据源属性
 * - @Testcontainers(disabledWithoutDocker = true)：Docker 不可用时跳过
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("testcontainers")
@Testcontainers(disabledWithoutDocker = true)
public abstract class AbstractMySQLTestcontainers {

    protected static final MySQLContainer<?> MYSQL;

    static {
        MYSQL = new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("mysql/testcontainers/init_charset.sql");
        MYSQL.start();
    }

    @DynamicPropertySource
    static void mysqlProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
    }
}
```

### 5.2 Service 层 DT 测试

```java
package com.example.service.impl;

import com.example.dt.AbstractMySQLTestcontainers;
import com.example.entity.User;
import com.example.repository.UserRepository;
import com.example.service.UserService;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * UserService 容器级 DT 测试
 *
 * 测试策略：真实 MySQL 容器，验证 SQL/JPA 映射正确性
 */
class UserServiceImplTest extends AbstractMySQLTestcontainers {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.save(createTestUser("user_001", "张三", "zhangsan@test.com"));
        userRepository.save(createTestUser("user_002", "李四", "lisi@test.com"));
    }

    @AfterEach
    void tearDown() {
        userRepository.deleteByUserIdStartingWith("DT_TEST_");
    }

    // ===== 正例 =====

    @Test
    @DisplayName("正常用例-根据ID查询用户返回正确结果")
    @Sql(scripts = "classpath:data/mysql/fixtures/global/init_student.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    @Sql(scripts = "classpath:data/mysql/fixtures/global/cleanup_student.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void findById_existingUser_returnUser() {
        // Given：数据已在 setUp 中插入

        // When
        User result = userService.findById("DT_TEST_user_001");

        // Then：验证真实 MySQL 查询结果
        assertThat(result).isNotNull();
        assertThat(result.getUserId()).isEqualTo("DT_TEST_user_001");
        assertThat(result.getUserName()).isEqualTo("张三");
        assertThat(result.getEmail()).isEqualTo("zhangsan@test.com");
    }

    @Test
    @DisplayName("正常用例-创建用户并持久化到数据库")
    void createUser_validInput_persistAndReturn() {
        // Given
        User newUser = createTestUser("user_003", "王五", "wangwu@test.com");

        // When
        User result = userService.createUser(newUser);

        // Then：验证真实数据库持久化
        assertThat(result.getUserId()).isEqualTo("DT_TEST_user_003");
        User dbUser = userRepository.findById("DT_TEST_user_003").orElseThrow();
        assertThat(dbUser.getUserName()).isEqualTo("王五");
    }

    // ===== 反例 =====

    @Test
    @DisplayName("异常用例-查询不存在的用户返回空")
    void findById_nonExistentUser_returnEmpty() {
        // When & Then
        assertThatThrownBy(() -> userService.findById("DT_TEST_nonexistent"))
            .isInstanceOf(RuntimeException.class)
            .hasMessageContaining("USER_NOT_FOUND");
    }

    @Test
    @DisplayName("异常用例-创建重复用户抛出异常")
    void createUser_duplicateUser_throwException() {
        // Given：setUp 中已插入 DT_TEST_user_001
        User duplicate = createTestUser("user_001", "重复", "dup@test.com");

        // When & Then：验证真实数据库唯一约束
        assertThatThrownBy(() -> userService.createUser(duplicate))
            .isInstanceOf(RuntimeException.class);
    }

    // ===== 辅助方法 =====

    private User createTestUser(String userId, String name, String email) {
        User user = new User();
        user.setUserId("DT_TEST_" + userId);
        user.setUserName(name);
        user.setEmail(email);
        return user;
    }
}
```

### 5.3 Repository 层 DT 测试

```java
package com.example.repository;

import com.example.dt.AbstractMySQLTestcontainers;
import com.example.entity.User;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * UserRepository 容器级 DT 测试
 *
 * 测试策略：真实 MySQL 容器，验证 JPA 映射和自定义查询
 */
class UserRepositoryTest extends AbstractMySQLTestcontainers {

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.save(createTestUser("user_001", "张三"));
        userRepository.save(createTestUser("user_002", "李四"));
    }

    @AfterEach
    void tearDown() {
        userRepository.deleteByUserIdStartingWith("DT_TEST_");
    }

    @Test
    @DisplayName("正常用例-JPA映射查询验证")
    @Sql(scripts = "classpath:data/mysql/fixtures/global/init_student.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    @Sql(scripts = "classpath:data/mysql/fixtures/global/cleanup_student.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    void findById_existingUser_returnUser() {
        User result = userRepository.findById("DT_TEST_user_001").orElseThrow();

        assertThat(result.getUserId()).isEqualTo("DT_TEST_user_001");
        assertThat(result.getUserName()).isEqualTo("张三");
    }

    @Test
    @DisplayName("正常用例-自定义查询验证")
    void findByUserNameContaining_keyword_returnMatchedUsers() {
        var results = userRepository.findByUserNameContaining("张");

        assertThat(results).hasSize(1);
        assertThat(results.get(0).getUserName()).isEqualTo("张三");
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

## 六、初始化脚本示例

### init_charset.sql

```sql
-- mysql/testcontainers/init_charset.sql
-- 容器启动时执行的字符集初始化脚本

SET NAMES utf8mb4;
SET CHARACTER SET utf8mb4;

ALTER DATABASE testdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### schema.sql（全局 Fixture）

```sql
-- mysql/fixtures/global/schema.sql
-- 全局建表脚本

CREATE TABLE IF NOT EXISTS t_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(64) NOT NULL UNIQUE,
    user_name VARCHAR(100) NOT NULL,
    email VARCHAR(200),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```