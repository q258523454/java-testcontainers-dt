# Java Testcontainers DT 指导 Skill

一个面向 Java 服务端的 DT 编写指导 skill，基于 Spring Boot + JUnit 5 + Testcontainers，快速生成容器级 DT 测试。

## 它能做什么

- 验证真实基础设施交互：MySQL、Redis、Elasticsearch 容器测试
- 组合 WireMock 模拟外部 HTTP API、Mockito 模拟本地依赖
- 遵循 AIR / FIRST / BCDE 原则，生成规范、可复用的测试用例

## 适用场景

- 编写容器级 DT 测试、创建 Testcontainers 测试类
- 验证 SQL 语法、JPA 映射、Redis 序列化、ES 索引
- 任何涉及"写测试用例 / 写 DT / 测试 MySQL / ES / Redis 真实交互"的需求

## 环境要求

Java 21 · JUnit 5 · Testcontainers 1.16+ · Spring Boot 3.1+ · Docker

## 使用方法

将该 skill 安装到 opencode 后，直接提出测试需求即可，例如：

- "帮我写一个 UserService 的容器级 DT"
- "补充 Order 模块的 MySQL + Redis 测试用例"

skill 会根据 [SKILL.md](./SKILL.md) 的工作流，分析被测代码、选择基类、生成测试代码，并加载对应中间件的规范文档与代码模板。

## 目录结构

| 路径 | 内容 |
|------|------|
| `SKILL.md` | 技能入口与工作流 |
| `references/` | 核心规范与各中间件测试规范 |
| `assets/base-classes/` | 测试基类代码模板 |
| `assets/config-templates/` | pom 依赖与 Testcontainers 配置模板 |
