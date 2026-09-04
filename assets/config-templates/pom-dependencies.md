# Testcontainers DT Maven 依赖清单

> **使用方式**：复制所需依赖到项目 `pom.xml` 的 `<dependencies>` 中
>
> **推荐**：使用 `testcontainers-bom` 统一管理版本

## 依赖模板

### Testcontainers 核心（必选）

```xml
<!-- Testcontainers 核心 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>

<!-- JUnit 5 集成 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### 中间件模块（按需引入）

```xml
<!-- MySQL 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>

<!-- Elasticsearch 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>elasticsearch</artifactId>
    <scope>test</scope>
</dependency>

<!-- Kafka 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>

<!-- MongoDB 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mongodb</artifactId>
    <scope>test</scope>
</dependency>

<!-- PostgreSQL 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

<!-- MockServer 模块（可选，替代 WireMock） -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mockserver</artifactId>
    <scope>test</scope>
</dependency>
```

### Spring Boot Testcontainers 集成

```xml
<!-- Spring Boot Testcontainers（Spring Boot 3.1+） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

### 版本管理（推荐）

```xml
<!-- 在 <dependencyManagement> 中统一管理版本 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.16.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```