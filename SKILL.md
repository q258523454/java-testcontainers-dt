---
name: java-testcontainers-dt
description: Spring Boot 容器级 DT 测试编写指导（Testcontainers 模式）。触发场景：编写容器级 DT 测试、创建 Testcontainers 测试类、补充容器级用例。即使用户未明确说"Testcontainers"，只要涉及容器级测试需求（"写容器测试"、"写测试用例"、"写DT"、"补充 DT 测试"、"测试 MySQL/ES/Redis 真实交互"），都应使用此技能。基于 AIR、FIRST、BCDE 原则，指导 JUnit 5 + Testcontainers 容器级 DT 编写。
compatibility: JUnit 5, Testcontainers 1.16+, Spring Boot 3.1+, Java 21, Docker
---

# Spring Boot 容器级 DT 测试编写指导（Testcontainers 模式）

## UT、DT 与集成测试的区别

| 测试类型 | 目标 | 测试范围 | 逻辑实现 | 适用场景 | 前置条件 |
|---------|------|---------|---------|---------|---------|
| UT 单元测试 | 验证代码逻辑 | 单个方法、类 | Mockito Mock 所有依赖，不加载 Spring 上下文 | 纯逻辑验证、参数校验 | 无 |
| DT 开发测试（embedded） | 验证业务流程 | API 接口、业务场景 | @SpringBootTest + H2 + jedis-mock + WireMock | 业务流程验证，环境差异可接受 | 无 |
| DT 开发测试（Testcontainers） | 验证真实基础设施交互 | API 接口、完整业务场景 | @SpringBootTest + 真实容器（MySQL/Redis/ES）+ WireMock | SQL 语法、JPA 映射、序列化、迁移脚本验证 | Docker 环境 |


## 触发场景

- 编写容器级 DT 测试、创建 Testcontainers 测试类、补充容器级用例
- 需要验证 SQL 语法、JPA 映射、Redis 序列化、ES 索引等真实基础设施交互
- 创建 `{被测类}Test` 测试类

## 依赖要求

Java 21、JUnit 5、Testcontainers 1.16+、Spring Boot 3.1+、Docker

## 工作流程

1. **确认测试模式**：判断是否需要 Testcontainers（参考 [DT决策树](./references/TestcontainersDT核心规范.md#六dt-vs-ut-决策树)）
2. **分析被测试代码**：读取源码，识别依赖的中间件（MySQL/ES/Redis/Kafka），选择对应基类
3. **设计测试用例（BCDE 原则）**：Border 边界值、Correct 正常场景、Design 结合设计文档、Error 异常场景。正常场景写在最前
4. **编写测试代码**：继承对应基类，遵循规范编写，参考下方文档
5. **验证测试**：运行测试确保通过，检查命名规范

## 规范文档（按需加载）

### 1. 核心规范（必读）

**[TestcontainersDT核心规范.md](./references/TestcontainersDT核心规范.md)** — 所有容器级 DT 必读

包含：DT vs UT 决策树、容器生命周期模式、Singleton Container 模式、@DynamicPropertySource 属性注入、基类设计、数据隔离策略、反模式

### 2. 组件容器测试规范（按中间件加载）

| 组件 | 文档 | 加载时机 |
|------|------|---------|
| MySQL | [MySQL容器测试规范.md](./references/component/MySQL容器测试规范.md) | MySQL 交互时 |
| Elasticsearch | [Elasticsearch容器测试规范.md](./references/component/Elasticsearch容器测试规范.md) | ES 交互时 |
| Redis | [Redis容器测试规范.md](./references/component/Redis容器测试规范.md) | Redis 交互时 |


### 3. WireMock + Testcontainers（外部 HTTP 调用时加载）

**[WireMock用例编写规范.md](./references/WireMock用例编写规范.md)** — WireMock Standalone 指导

包含：WireMock + Testcontainers 组合模式、Stubbing、Request Matching、Verification

**分工原则**：WireMock 模拟外部 HTTP API，Testcontainers 提供真实中间件，Mockito 模拟本地依赖

### 4. 用例编写通用规范（编写用例时加载）

**[Java用例编写规范.md](./references/Java用例编写规范.md)** — 命名、断言、GWT 结构等通用规范

包含：AIR 原则、FIRST 原则、命名规范、GWT 结构、断言规范、异常测试

### 5. 目录结构规范（创建文件/目录时加载）

**[DT目录结构规范.md](./references/DT目录结构规范.md)** — 测试目录组织、文件放置、命名约定

包含：src/test/ 顶层结构、中间件测试数据目录、fixtures/migrations/seeds/testcontainers 四层组织

## 代码模板（按需加载）

### 基类模板

| 文件 | 用途 |
|------|------|
| [AbstractTestcontainersTestcontainers.md](./assets/base-classes/AbstractTestcontainersTestcontainers.md) | 通用容器级 DT 基类（MySQL + Redis + ES） |
| [AbstractMySQLTestcontainers.md](./assets/base-classes/AbstractMySQLTestcontainers.md) | 仅 MySQL 容器基类 |
| [AbstractElasticsearchTestcontainers.md](./assets/base-classes/AbstractElasticsearchTestcontainers.md) | 仅 ES 容器基类 |
| [AbstractRedisTestcontainers.md](./assets/base-classes/AbstractRedisTestcontainers.md) | 仅 Redis 容器基类 |

### 配置模板

| 文件 | 用途 |
|------|------|
| [testcontainers-properties.md](./assets/config-templates/testcontainers-properties.md) | Testcontainers 全局配置 |
| [pom-dependencies.md](./assets/config-templates/pom-dependencies.md) | Maven 依赖清单 |