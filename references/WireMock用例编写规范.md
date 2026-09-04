# WireMock 用例编写规范

> **适用环境**: JDK 21 + Spring Boot 3.x + JUnit 5 + Mockito  
> **WireMock 版本**: 3.13.2 (standalone)  
> **文档来源**: WireMock 官方文档 (https://wiremock.org/) 及业界公认最佳实践

---

## 目录

1. [核心概念](#一核心概念)
2. [环境配置](#二环境配置)
3. [JUnit 5 集成](#三junit-5-集成)
4. [Stubbing（桩设置）](#四stubbing桩设置)
5. [文件式 Stub（__files 和 mappings）](#五文件式-stub__files-和-mappings)
6. [Request Matching（请求匹配）](#六request-matching请求匹配)
7. [Verification（验证）](#七verification验证)
8. [录制模式（Recording）](#八录制模式recording)
9. [最佳实践](#九最佳实践)
10. [完整示例](#十完整示例)
11. [常见问题](#十一常见问题)
12. [参考资料](#十二参考资料)

---

## 一、核心概念

### 1.1 什么是 WireMock

WireMock 是一个灵活的 API 模拟工具，用于在单元测试中模拟外部 HTTP 服务。

**核心特性**：
- **Stubbing**: 为匹配请求返回预设响应
- **Request Matching**: 基于 URL、方法、请求头、请求体等多维度匹配
- **Verification**: 验证请求是否按预期发送
- **文件式 Stub**: 从 `mappings/` 和 `__files/` 目录加载预置的 Stub 配置
- **录制模式**: 代理真实 API 请求，自动生成 mappings 和响应体文件

### 1.2 为什么使用 wiremock-standalone

`wiremock-standalone` 包含所有依赖，独立运行，不依赖 Spring 容器，适合纯单元测试场景。

**优势**：
- ✅ 测试速度快（毫秒级）
- ✅ 可以与 Mockito 配合使用
- ✅ 独立进程，不影响其他测试
- ✅ 支持文件式 Stub，复杂响应体可独立管理

### 1.3 两种 Stub 模式对比

| 特性 | 编程式 Stub（Programmatic） | 文件式 Stub（File-based） |
|------|---------------------------|--------------------------|
| 定义方式 | Java 代码中 `stubFor(...)` | JSON 文件（`mappings/` + `__files/`） |
| 响应体管理 | 内联在代码中 | 独立文件，支持大型 JSON/XML |
| 适用场景 | 简单响应、动态逻辑、多场景匹配 | 复杂响应体、录制回放、多场景 |
| 可维护性 | 代码与测试耦合 | 数据与逻辑分离，便于管理 |
| 团队协作 | 需要阅读代码 | 非开发人员也可维护 JSON |

> 两种模式的选择决策参见 [9.3 编程式 Stub vs 文件式 Stub 选择指南](#93-编程式-stub-vs-文件式-stub-选择指南)。

---

## 二、环境配置

### 2.1 Maven 依赖

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

### 2.2 目录结构规范

采用文件式 Stub 时，推荐以下目录结构：

```
src/test/resources/wiremock/
├── searchperson/                                         # 按业务域分组
│   ├── mappings/                                         # 请求-响应映射规则
│   │   ├── combine-search-zhangsan-success.json          # 搜索张三-成功
│   │   ├── combine-search-default-success.json           # 搜索默认关键词-成功
│   │   └── combine-search-empty-result.json             # 搜索无结果
│   └── __files/                                          # 响应体文件
│       ├── combine-search-zhangsan-success.json
│       ├── combine-search-default-success.json
│       └── combine-search-empty-result.json
├── payment/                                              # 另一个业务域
│   ├── mappings/
│   │   ├── create-payment-success.json                  # 创建支付-成功
│   │   └── create-payment-insufficient-balance-error.json  # 创建支付-余额不足
│   └── __files/
│       ├── create-payment-success.json
│       └── create-payment-insufficient-balance-error.json
```

### 2.3 文件命名规范

文件名是快速识别业务场景和请求内容的第一标识。采用以下三段式命名约定：

**命名公式**: `{API端点}-{场景描述}-{响应类型}.json`

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| **API端点** | URL 路径最后一段的语义化简写，kebab-case | `combine-search`、`create-payment`、`login` |
| **场景描述** | 请求的关键区分参数或业务场景 | `zhangsan`、`default`、`invalid-token`、`empty-result` |
| **响应类型** | 响应结果分类 | `success`、`error`、`empty`、`timeout`、`unauthorized` |

**命名示例**：

| 文件名 | 业务含义 |
|--------|---------|
| `combine-search-zhangsan-success.json` | 人员搜索接口，搜索"张三"，成功响应 |
| `combine-search-default-success.json` | 人员搜索接口，搜索默认关键词，成功响应 |
| `combine-search-empty-result.json` | 人员搜索接口，无搜索结果 |
| `create-payment-success.json` | 创建支付接口，支付成功 |
| `create-payment-insufficient-balance-error.json` | 创建支付接口，余额不足错误 |
| `login-valid-token-success.json` | 登录接口，有效令牌，登录成功 |
| `login-invalid-token-unauthorized.json` | 登录接口，无效令牌，未授权 |

**规则约束**：
- mapping 文件名与对应的 `__files` 响应体文件名**必须完全一致**
- 全小写，kebab-case 分隔（`-`），不允许大写字母和下划线
- 不包含 UUID、时间戳等无意义后缀
- 场景描述应使用业务可读词汇（`zhangsan`），避免技术术语（`case1`）

---

## 三、JUnit 5 集成

### 3.1 编程式模式（WireMockExtension）

使用 `@RegisterExtension` 和 `WireMockExtension`，适合简单 Stub 场景。

> 两种集成方式的选择参见 [11.6 WireMockServer vs WireMockExtension 选择](#116-wiremockserver-vs-wiremockextension-选择)。

```java
class ApiTest {
    
    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort().configureStaticDsl(true))
            .build();
    
    @Test
    void test_api_call() {
        wireMock.stubFor(get("/api/resource")
                .willReturn(aResponse()
                        .withStatus(200)
                        .withBody("{\"status\":\"success\"}")));
        
        String baseUrl = wireMock.getRuntimeInfo().getHttpBaseUrl();
        wireMock.verify(getRequestedFor(urlEqualTo("/api/resource")));
    }
}
```

### 3.2 WireMockServer 编程式管理（文件式 Stub 推荐）

当需要从文件系统加载 `mappings/` 和 `__files/` 时，使用 `WireMockServer` + `@BeforeAll`/`@AfterAll` 手动管理生命周期。

**核心配置**: `withRootDirectory()` 指定 mappings 和 __files 的根目录。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class WireMockFileTest {

    private static WireMockServer wireMockServer;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(
                WireMockConfiguration.wireMockConfig()
                        .port(8089)                                              // 固定端口
                        .withRootDirectory("src/test/resources/wiremock/searchperson")  // 指定根目录
        );
        wireMockServer.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }
}
```

> **何时用固定端口**: 当被测代码中 URL 硬编码时（如配置文件指定了 `http://localhost:8089/...`），必须使用固定端口。否则推荐动态端口避免冲突。

### 3.3 WireMock + Mockito 组合使用（核心场景）

**职责分工**（详细决策参见 [9.2 WireMock vs Mockito 分工边界](#92-wiremock-vs-mockito-分工边界核心)）：
- **WireMock**: 模拟外部 HTTP API（第三方服务、微服务调用）
- **Mockito**: 模拟本地依赖（Repository、Service、DAO）

**组合模式核心步骤**：

1. `@ExtendWith(MockitoExtension.class)` + `@RegisterExtension WireMockExtension` 同时启用
2. `@Mock` 标注本地依赖，`@InjectMocks` 注入被测 Service
3. `wireMock.stubFor(...)` 模拟外部 API，`when(...).thenReturn(...)` 模拟本地依赖
4. 执行被测方法后，`wireMock.verify(...)` + `verify(...)` 双重验证

> 完整代码示例参见 [10.1 WireMock + Mockito 组合示例](#101-wiremock--mockito-组合示例)。

### 3.4 WireMockRuntimeInfo 可用信息

```java
@Test
void test_runtime_info(WireMockRuntimeInfo info) {
    int port = info.getHttpPort();              // HTTP 端口
    String baseUrl = info.getHttpBaseUrl();     // HTTP 基础 URL
    WireMock wireMock = info.getWireMock();     // WireMock 实例
}
```

### 3.5 测试生命周期管理

#### 自动重置 vs 手动重置

WireMockExtension 默认会在每个测试方法后自动重置 mappings，但在某些场景需要手动重置以确保测试隔离：

**最佳实践示例**：

```java
class ApiTest {
    
    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();
    
    @BeforeEach
    void setUp() {
        wireMock.resetMappings();  // 确保每个测试从干净状态开始
    }
    
    @Test
    @DisplayName("测试场景1-成功响应")
    void test_scenario1() {
        wireMock.stubFor(get("/api/users").willReturn(ok()));
        // 测试逻辑
    }
    
    @Test
    @DisplayName("测试场景2-错误响应")
    void test_scenario2() {
        // resetMappings() 确保 stub 不会相互干扰
        wireMock.stubFor(get("/api/users").willReturn(serverError()));
        // 测试逻辑
    }
}
```

> **注意**: 使用 `WireMockServer` + 文件式 Stub 时，`resetMappings()` 会清除从文件加载的映射。如需保留文件映射，请勿在 `@BeforeEach` 中调用 `resetMappings()`，或改用编程式追加 Stub。

---

## 四、Stubbing（桩设置）

### 4.1 基础 Stub + 常用响应方法

```java
stubFor(get("/api/users").willReturn(ok()));
stubFor(get("/api/users").willReturn(okJson("[{\"id\":1,\"name\":\"John\"}]")));
stubFor(post("/api/create").willReturn(created()));
stubFor(get("/api/error").willReturn(serverError()));
stubFor(post("/api/auth").willReturn(unauthorized()));
```

**常用响应快捷方法**：
- `ok()` / `okJson()` / `okXml()` - 200 状态码
- `created()` - 201
- `noContent()` - 204
- `badRequest()` - 400
- `unauthorized()` - 401
- `forbidden()` - 403
- `notFound()` - 404
- `serverError()` - 500

### 4.2 响应头 + 响应体

```java
stubFor(get("/api/resource")
        .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withHeader("Cache-Control", "no-cache")));

stubFor(get("/api/text").willReturn(aResponse().withBody("Plain text")));
stubFor(get("/api/json").willReturn(aResponse().withJsonBody("{\"id\":1,\"name\":\"John\"}")));
stubFor(get("/api/file").willReturn(aResponse().withBodyFile("response.json")));  // __files 目录
```

### 4.3 bodyFile - 从文件读取响应体

`withBodyFile()` 引用 `__files/` 目录下的文件，适合大型 JSON/XML 响应体：

```java
// 编程式：引用 __files/user-profile.json
stubFor(get("/api/users/1")
        .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBodyFile("user-profile.json")));
```

**支持的响应体类型**：

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| `withBody(String)` | 内联字符串 | 短文本响应 |
| `withJsonBody(String)` | 内联 JSON | 中等 JSON 响应 |
| `withBodyFile(String)` | 引用 `__files/` 文件 | 大型 JSON/XML/二进制 |
| `withBase64Body(String)` | Base64 编码 | 二进制数据 |

### 4.4 编程式多场景 Stub（同一 URL 根据入参返回不同响应）

通过 `withRequestBody(matchingJsonPath(...))` 在代码中根据请求体字段值匹配不同响应。适用于自定义 `StubFactory` 工厂模式或 `WireMockExtension` 场景。

> **方法说明**：下方示例中 `initStub()` 是自定义 `StubFactory` 接口的实现方法，用于集中注册 Stub 规则；`readContent()` 是工具方法，读取 classpath 下指定路径的 JSON 文件内容为字符串。两者为项目自定义封装，非 WireMock 内置 API。

```java
@Override
public void initStub(WireMockServer server) {
    // 读取预置响应体文件（classpath:searchperson/ 目录下）
    String defaultBody = readContent("searchperson/lx-search-person-default-success.json");
    String zhangsanBody = readContent("searchperson/lx-search-person-zhangsan-success.json");

    // ① 精确匹配：search_txt = "张三" → 返回张三专属响应
    server.stubFor(WireMock.post(WireMock.urlEqualTo(URL))
            .withRequestBody(WireMock.matchingJsonPath("$.search_txt", WireMock.equalTo("张三")))
            .willReturn(WireMock.aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody(zhangsanBody)));

    // ② 兜底响应：其他请求 → 返回默认响应
    server.stubFor(WireMock.post(WireMock.urlEqualTo(URL))
            .willReturn(WireMock.aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody(defaultBody)));
}
```

**注册顺序规则**：WireMock 按注册顺序逐一匹配，第一个匹配成功的 stub 生效。精确匹配的 stub 必须先注册，兜底 stub 后注册。

> 文件式实现同一效果的方式参见 [5.5 同一 URL 多场景映射](#55-同一-url-多场景映射)。

---

## 五、文件式 Stub（__files 和 mappings）

### 5.1 核心概念

文件式 Stub 将请求-响应映射规则和响应体数据存储为 JSON 文件，实现**数据与逻辑分离**。

**目录结构**（单业务域视图，完整多业务域结构参见 [2.2 目录结构规范](#22-目录结构规范)）：

```
src/test/resources/wiremock/searchperson/                       ← withRootDirectory 指向此处
├── mappings/                                                    ← 映射规则文件
│   ├── combine-search-zhangsan-success.json                     # 搜索张三-成功
│   └── combine-search-default-success.json                      # 搜索默认关键词-成功
└── __files/                                                     ← 响应体文件
    ├── combine-search-zhangsan-success.json
    └── combine-search-default-success.json
```

> 文件命名遵循 [2.3 文件命名规范](#23-文件命名规范)，通过文件名即可识别业务场景和请求内容。

**加载机制**：
- WireMock 启动时递归扫描 `mappings/` 目录下所有 `.json` 文件
- 每个文件定义一个或多个 Stub 映射（request → response）
- `response.bodyFileName` 引用 `__files/` 目录下的响应体文件
- JSON 解析错误会抛出 `MappingFileException`，启动即失败

### 5.2 Mapping 文件格式

#### 5.2.1 单映射文件（推荐）

每个文件包含一个独立的映射规则：

```json
{
  "id" : "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "name" : "search-zhangsan",
  "request" : {
    "url" : "/api/v1/person/combine-search",
    "method" : "POST",
    "headers" : {
      "Content-Type" : {
        "equalTo" : "application/json",
        "caseInsensitive" : true
      }
    },
    "bodyPatterns" : [ {
      "equalToJson" : "{\r\n    \"text\": \"张三\",\r\n    \"pageIndex\": 1,\r\n    \"pageSize\": 10,\r\n    \"language\": \"zh\",\r\n    \"dataTypesParam\": {\r\n        \"cross\": {},\r\n        \"employee\": {},\r\n        \"localOuter\": {},\r\n        \"employeeRemark\": {}\r\n    }\r\n}",
      "ignoreArrayOrder" : true,
      "ignoreExtraElements" : true
    } ]
  },
  "response" : {
    "status" : 200,
    "bodyFileName" : "combine-search-zhangsan-success.json",
    "headers" : {
      "Content-Type" : "application/json;charset=UTF-8"
    }
  },
  "uuid" : "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "persistent" : true
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` / `uuid` | String | 唯一标识符（UUID），两者值应一致 |
| `name` | String | 语义化名称，便于定位和录制时自动命名 |
| `request` | Object | 请求匹配规则（URL、method、headers、bodyPatterns） |
| `response` | Object | 响应定义（status、bodyFileName、headers） |
| `persistent` | Boolean | `true` 表示持久化映射，`resetMappings()` 不会清除 |
| `insertionIndex` | Integer | 插入顺序（影响匹配优先级） |

#### 5.2.2 多映射文件

一个文件包含多个映射（适合将相关映射聚合管理）：

```json
{
  "mappings": [
    {
      "name": "login-success",
      "request": { "url": "/api/login", "method": "POST" },
      "response": { "status": 200, "bodyFileName": "login-success.json" }
    },
    {
      "name": "login-failed",
      "request": { "url": "/api/login", "method": "POST" },
      "response": { "status": 401, "bodyFileName": "login-failed.json" }
    }
  ]
}
```

> **限制**: 多映射文件是**只读**的，无法通过 API 动态修改或删除其中的映射。需要动态修改的场景请使用单映射文件格式。

### 5.3 bodyFileName 引用机制

`response.bodyFileName` 引用 `__files/` 目录下的文件，路径相对于 `__files/` 根目录：

```json
{
  "response": {
    "status": 200,
    "bodyFileName": "combine-search-zhangsan-success.json"
  }
}
```

**路径规则**：
- ✅ `"bodyFileName": "user-profile.json"` — 相对路径，引用 `__files/user-profile.json`
- ✅ `"bodyFileName": "users/user-profile.json"` — 子目录路径，引用 `__files/users/user-profile.json`
- ❌ `"bodyFileName": "/absolute/path/to/file.json"` — 不支持绝对路径

**支持的文件类型**：
- JSON / XML / 文本文件 — 直接作为响应体返回
- 二进制文件（PNG、PDF 等）— 以字节流返回

### 5.4 请求体匹配（bodyPatterns）

文件式 Stub 通过 `bodyPatterns` 匹配请求体，支持灵活的 JSON 匹配策略：

```json
"bodyPatterns" : [ {
  "equalToJson" : "{\r\n    \"text\": \"张三\",\r\n    \"pageIndex\": 1,\r\n    \"pageSize\": 10\r\n}",
  "ignoreArrayOrder" : true,
  "ignoreExtraElements" : true
} ]
```

**匹配策略**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `equalToJson` | JSON 结构化比较 | 精确匹配请求体 |
| `ignoreArrayOrder` | 忽略数组元素顺序 | 数组顺序无关的请求 |
| `ignoreExtraElements` | 忽略额外字段 | 请求体可能包含额外字段 |
| `matchingJsonPath` | JSON Path 匹配 | 只匹配关键字段 |
| `matchesJsonPath` | JSON Path 正则匹配 | 模糊匹配字段值 |

**按关键字段匹配（推荐）**：

```json
"bodyPatterns" : [ {
  "matchesJsonPath" : "$.text"
}, {
  "matchesJsonPath" : "$[?(@.text == '张三')]"
} ]
```

### 5.5 同一 URL 多场景映射

通过不同的 `bodyPatterns`，同一 URL 可以返回不同响应，实现多场景测试：

**场景**: 搜索"张三"返回 2 条记录，搜索"测试"返回 720 条记录。

```
mappings/
├── combine-search-zhangsan-success.json     # bodyPatterns 匹配 text="张三" → 返回对应 __files
└── combine-search-default-success.json      # bodyPatterns 匹配 text="测试" → 返回对应 __files
```

WireMock 按映射的插入顺序逐一匹配，第一个匹配成功的映射返回响应。

### 5.6 完整文件式 Stub 示例

#### 测试类

```java
package com.mytest.uti;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONArray;
import com.alibaba.fastjson.JSONObject;
import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.core.WireMockConfiguration;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.HashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class WireMockFileTest {

    // 被测代码中 URL 硬编码，因此使用固定端口
    public static final String URL = "http://localhost:8089/api/v1/person/combine-search";

    private static WireMockServer wireMockServer;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(
                WireMockConfiguration.wireMockConfig()
                        .port(8089)
                        .withRootDirectory("src/test/resources/wiremock/searchperson")
        );
        wireMockServer.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }

    @Test
    @DisplayName("正常用例-搜索张三返回对应员工数据")
    void testServiceMethod_searchZhangsan_returnZhangsanData() {
        // Given: 构造搜索"张三"的请求参数
        JSONObject request = new JSONObject();
        request.put("text", "张三");
        request.put("pageIndex", 1);
        request.put("pageSize", 10);
        request.put("language", "zh");
        JSONObject dataTypesParam = new JSONObject();
        dataTypesParam.put("cross", new JSONObject());
        dataTypesParam.put("employee", new JSONObject());
        dataTypesParam.put("localOuter", new JSONObject());
        dataTypesParam.put("employeeRemark", new JSONObject());
        request.put("dataTypesParam", dataTypesParam);

        Map<String, String> headerMap = new HashMap<>();
        headerMap.put("Content-Type", "application/json");

        // When: 调用接口，WireMock 根据请求体匹配到不同 mapping，返回不同 response
        JSONObject result = HttpClientUtil.post(URL, request, headerMap);

        // Then: 验证返回结果
        assertThat(result).isNotNull();
        JSONObject responseJson = JSON.parseObject(result.getString("message"));
        assertThat(responseJson.getString("code")).isEqualTo("200");
        assertThat(responseJson.getString("message")).isEqualTo("success");

        // 验证员工数据
        JSONObject employeeData = responseJson.getJSONObject("data").getJSONObject("employee");
        JSONArray employeeList = employeeData.getJSONArray("data");
        assertThat(employeeList).hasSize(2);
        assertThat(employeeList.getJSONObject(0).getString("chineseName")).isEqualTo("张三");
        assertThat(employeeList.getJSONObject(0).getString("employeeNumber")).isEqualTo("EMP001");

        // 验证分页信息
        JSONObject pagination = employeeData.getJSONObject("pagination");
        assertThat(pagination.getInteger("page")).isEqualTo(1);
        assertThat(pagination.getInteger("pageSize")).isEqualTo(10);
        assertThat(pagination.getInteger("totalRows")).isEqualTo(2);
    }
}
```

#### Mapping 文件（mappings/combine-search-zhangsan-success.json）

```json
{
  "id" : "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "name" : "combine-search-zhangsan-success",
  "request" : {
    "url" : "/api/v1/person/combine-search",
    "method" : "POST",
    "headers" : {
      "Content-Type" : {
        "equalTo" : "application/json",
        "caseInsensitive" : true
      }
    },
    "bodyPatterns" : [ {
      "equalToJson" : "{\r\n    \"text\": \"张三\",\r\n    \"pageIndex\": 1,\r\n    \"pageSize\": 10,\r\n    \"language\": \"zh\",\r\n    \"dataTypesParam\": {\r\n        \"cross\": {},\r\n        \"employee\": {},\r\n        \"localOuter\": {},\r\n        \"employeeRemark\": {}\r\n    }\r\n}",
      "ignoreArrayOrder" : true,
      "ignoreExtraElements" : true
    } ]
  },
  "response" : {
    "status" : 200,
    "bodyFileName" : "combine-search-zhangsan-success.json",
    "headers" : {
      "Content-Type" : "application/json;charset=UTF-8"
    }
  },
  "uuid" : "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "persistent" : true
}
```

#### 响应体文件（__files/combine-search-zhangsan-success.json）

```json
{
  "code" : "200",
  "message" : "success",
  "data" : {
    "employee" : {
      "data" : [ {
        "chineseName" : "张三",
        "employeeNumber" : "EMP001",
        "deptName" : "网络产品研发部",
        "position" : "高级工程师"
      }, {
        "chineseName" : "李四",
        "employeeNumber" : "EMP002",
        "deptName" : "网络产品研发部",
        "position" : "产品经理"
      } ],
      "pagination" : {
        "page" : 1,
        "pageSize" : 10,
        "totalPages" : 1,
        "totalRows" : 2
      }
    }
  }
}
```

### 5.7 文件式 Stub 最佳实践

#### ✅ 推荐

- **按业务域分目录**: `wiremock/searchperson/`、`wiremock/payment/`，互不干扰
- **规范文件命名**: `combine-search-zhangsan-success.json` 而非 `mapping1.json`，通过文件名即可识别业务场景和请求内容（参见 [2.3 文件命名规范](#23-文件命名规范)）
- **mapping 与 __files 文件名一致**: `mappings/xxx.json` 与 `__files/xxx.json` 同名，便于双向定位
- **使用 `name` 字段**: 录制时自动命名，手动维护时便于定位
- **`persistent: true`**: 防止 `resetMappings()` 误清除文件映射
- **`ignoreExtraElements: true`**: 请求体新增字段时不需要更新 mapping

#### ❌ 避免

- 将大型响应体内联在 mapping 文件中（应使用 `bodyFileName` 引用 `__files/`）
- 多个 mapping 使用相同 UUID（会导致映射覆盖）
- 使用绝对路径引用 `__files` 文件
- 在 `@BeforeEach` 中调用 `resetMappings()` 清除文件映射

---

## 六、Request Matching（请求匹配）

### 6.1 URL 匹配

```java
urlEqualTo("/api/users?id=123")         // 精确匹配（路径+查询）
urlMatching("/api/users/[0-9]+")        // 正则匹配
urlPathEqualTo("/api/users")            // 仅路径精确匹配
urlPathMatching("/api/[a-z]+")          // 仅路径正则匹配
anyUrl()                                // 匹配所有
```

### 6.2 Header 匹配（认证场景核心）

```java
stubFor(get("/api/users")
        .withHeader("Authorization", equalTo("Bearer token123"))  // JWT 认证
        .withHeader("Content-Type", containing("json"))
        .willReturn(ok()));
```

### 6.3 Query 参数匹配

```java
stubFor(get(urlPathEqualTo("/api/users"))
        .withQueryParam("id", equalTo("123"))
        .withQueryParam("status", matching("active|inactive"))
        .willReturn(ok()));
```

### 6.4 JSON Body 匹配

```java
stubFor(post("/api/users")
        .withRequestBody(equalToJson("{\"name\":\"John\"}"))
        .willReturn(created()));

stubFor(post("/api/users")
        .withRequestBody(equalToJson("{\"name\":\"John\"}", true, true))  // 宽松匹配
        .willReturn(created()));

stubFor(post("/api/users")
        .withRequestBody(matchingJsonPath("$.name"))  // JSON Path 匹配
        .willReturn(created()));
```

### 6.5 HTTP 方法匹配

```java
get(urlEqualTo("/api/users"))
post(urlEqualTo("/api/users"))
put(urlEqualTo("/api/users"))
delete(urlEqualTo("/api/users"))
any(urlEqualTo("/api/users"))
```

### 6.6 完整匹配示例

```java
stubFor(post(urlPathEqualTo("/api/users"))
        .withHeader("Content-Type", equalTo("application/json"))
        .withHeader("Authorization", equalTo("Bearer token"))
        .withQueryParam("version", equalTo("v1"))
        .withRequestBody(matchingJsonPath("$.name"))
        .willReturn(created().withJsonBody("{\"id\":1,\"name\":\"John\"}")));
```

---

## 七、Verification（验证）

### 7.1 基础验证 + 次数验证

```java
verify(getRequestedFor(urlEqualTo("/api/users")));           // 至少一次
verify(3, getRequestedFor(urlEqualTo("/api/users")));        // 精确3次
verify(0, postRequestedFor(urlEqualTo("/api/users")));       // 验证零次

verify(lessThan(5), getRequestedFor(urlEqualTo("/api")));    // < 5
verify(exactly(5), getRequestedFor(urlEqualTo("/api")));     // = 5
verify(moreThan(5), getRequestedFor(urlEqualTo("/api")));    // > 5
```

### 7.2 详细验证

```java
verify(postRequestedFor(urlEqualTo("/api/users"))
        .withHeader("Content-Type", equalTo("application/json"))
        .withRequestBody(matchingJsonPath("$.name")));
```

### 7.3 请求日志查询（调试必备）

```java
List<ServeEvent> allEvents = getAllServeEvents();
List<ServeEvent> unmatched = getAllServeEvents(ServeEventQuery.ALL_UNMATCHED);
List<LoggedRequest> requests = findAll(getRequestedFor(urlMatching("/api/.*")));
```

> **调试技巧**: 当请求未匹配到任何 Stub 时，查看 `ALL_UNMATCHED` 事件可以快速定位匹配失败的原因。

---

## 八、录制模式（Recording）

### 8.1 什么是录制模式

录制模式（Recording）允许 WireMock 作为代理，转发请求到真实 API 并自动生成 mapping 文件和 `__files` 响应体文件。适用于：
- 快速生成复杂 API 的 Stub 数据
- 捕获真实 API 的响应结构
- 减少手工编写 mapping 文件的工作量

### 8.2 录制流程

```java
@Test
@DisplayName("录制真实API响应")
void test_recordApi() {
    // 1. 启动 WireMock 并配置录制目标
    WireMockServer wireMockServer = new WireMockServer(
            WireMockConfiguration.wireMockConfig()
                    .port(8089)
                    .withRootDirectory("src/test/resources/wiremock/searchperson")
    );
    wireMockServer.start();
    
    // 2. 开始录制，代理目标为真实 API
    wireMockServer.startRecording("https://api.example.com");
    
    // 3. 发送请求到 WireMock（WireMock 转发到真实 API）
    JSONObject request = new JSONObject();
    request.put("text", "张三");
    // ... 构造请求参数
    HttpClientUtil.post("http://localhost:8089/api/v1/person/combine-search", request, headers);
    
    // 4. 停止录制，自动生成 mapping 和 __files
    List<StubMapping> recordedMappings = wireMockServer.stopRecording().getStubMappings();
    
    // 5. 验证录制结果
    assertThat(recordedMappings).isNotEmpty();
    
    wireMockServer.stop();
}
```

### 8.3 录制生成的文件

录制停止后，WireMock 自动在 `mappings/` 和 `__files/` 目录生成文件：

**文件命名规则**：

录制默认使用 UUID 后缀命名（`{name}-{id}.json`），**建议录制后立即重命名**为规范命名：

| 阶段 | 文件名示例 | 说明 |
|------|-----------|------|
| 录制自动生成 | `api_v1_person_combine-search-c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5.json` | 含 UUID，不可读 |
| **重命名后（推荐）** | `combine-search-zhangsan-success.json` | 遵循 [2.3 文件命名规范](#23-文件命名规范) |

> **重要**: 重命名 mapping 文件后，需同步修改 mapping JSON 内的 `response.bodyFileName` 字段，使其指向新的 `__files` 文件名。

**自动生成的 mapping 文件**：

```json
{
  "id": "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "name": "api_v1_person_combine-search-zhangsan",
  "request": {
    "url": "/api/v1/person/combine-search",
    "method": "POST",
    "bodyPatterns": [{
      "equalToJson": "{...}",
      "ignoreArrayOrder": true,
      "ignoreExtraElements": true
    }]
  },
  "response": {
    "status": 200,
    "bodyFileName": "api_v1_person_combine-search-c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5.json",
    "headers": { ... }
  },
  "uuid": "c7d8e9f0-1a2b-4c3d-5e6f-708192a3b4c5",
  "persistent": true
}
```

### 8.4 录制最佳实践

- **录制后立即重命名**: 将 UUID 文件名改为规范命名（如 `combine-search-zhangsan-success.json`），同步更新 `bodyFileName` 引用
- **录制后审查**: 检查生成的 mapping 文件，调整 `bodyPatterns` 为更宽松的匹配（如 `matchingJsonPath`）
- **清理敏感数据**: 录制的响应体可能包含真实用户数据，需脱敏处理
- **按业务域分目录**: 每次录制使用独立的 `withRootDirectory`，避免文件混杂
- **一次性录制**: 录制是获取初始数据的手段，不应在常规测试中反复录制

---

## 九、最佳实践

### 9.1 测试隔离

#### ✅ 推荐

- 每个测试类使用独立 WireMock 实例
- 每个测试方法前自动重置（默认行为）
- 使用动态端口避免冲突

#### ❌ 避免

- 多个测试共享 WireMock 实例不重置
- 使用固定端口导致并发冲突（文件式 Stub 除外）

### 9.2 WireMock vs Mockito 分工边界（核心）

#### 职责边界决策树

```
测试场景决策：
├─ 是否涉及 HTTP 调用？
│  ├─ YES → WireMock
│  │   ├─ 外部 API（第三方服务） → WireMock
│  │   ├─ 微服务间调用 → WireMock
│  │   └─ 网络行为（延迟、超时） → WireMock
│  └─ NO → Mockito
│      ├─ 本地 Repository → Mockito
│      ├─ 本地 Service → Mockito
│      └─ 业务逻辑返回值 → Mockito
```

#### 分工对比表

| **场景** | **工具** | **Mock 内容** | **典型示例** |
|---------|---------|--------------|------------|
| 外部 REST API | **WireMock** | HTTP 响应、状态码、Header | 支付 API、短信服务、OAuth |
| 微服务调用 | **WireMock** | HTTP 请求/响应细节 | 用户服务 → 订单服务 |
| 本地 Repository | **Mockito** | 数据返回、异常抛出 | `when(repo.findById()).thenReturn()` |
| 本地 Service | **Mockito** | 业务逻辑结果 | `when(service.process()).thenReturn()` |
| **组合场景** | **WireMock + Mockito** | 外部 API + 本地依赖 | Service 调用外部 API + Repository |

### 9.3 编程式 Stub vs 文件式 Stub 选择指南

> 两种模式的基本特性对比参见 [1.3 两种 Stub 模式对比](#13-两种-stub-模式对比)。

| 决策因素 | 编程式 Stub | 文件式 Stub |
|---------|-----------|------------|
| 响应体大小 | < 500 字符 | > 500 字符或复杂嵌套 JSON |
| 响应体数量 | 1-3 个简单响应 | 多个复杂响应，或需录制回放 |
| 团队协作 | 开发人员维护 | 测试/产品人员也可维护 JSON |
| 数据来源 | 手工构造 | 录制真实 API |
| 维护成本 | 代码与数据耦合 | 数据与逻辑分离 |
| 动态行为 | 需要条件逻辑、延迟模拟 | 静态响应回放 |

**推荐策略**: 简单场景用编程式 Stub，复杂 API 响应用文件式 Stub，两者可在同一测试类中混用。

### 9.4 组合使用典型场景

**场景1: Service 层调用外部 API + 本地 Repository**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();
    
    @Mock
    private OrderRepository repository;  // Mockito: 本地依赖
    
    @InjectMocks
    private OrderService service;
    
    @Test
    void createOrder_combinedMocks() {
        // WireMock: 模拟外部支付 API
        wireMock.stubFor(post("/api/payments")
                .willReturn(okJson("{\"txnId\":\"txn123\",\"status\":\"SUCCESS\"}")));
        
        // Mockito: 模拟本地数据库
        Order savedOrder = new Order(1L, "txn123", "SUCCESS");
        when(repository.save(any())).thenReturn(savedOrder);
        
        // 测试真实业务逻辑
        Order result = service.createOrder(100.0);
        
        // 双重验证
        wireMock.verify(postRequestedFor(urlEqualTo("/api/payments")));
        verify(repository).save(any());
    }
}
```

**场景2: 认证授权 API（JWT Header 匹配）**

```java
@Test
void test_jwtAuthentication() {
    wireMock.stubFor(get("/api/users")
            .withHeader("Authorization", equalTo("Bearer valid-token"))
            .willReturn(okJson("{\"id\":1,\"name\":\"John\"}")));
    
    wireMock.stubFor(get("/api/users")
            .withHeader("Authorization", equalTo("Bearer invalid-token"))
            .willReturn(unauthorized()));
    
    // 测试认证成功场景
    User user = service.getUserWithToken("valid-token");
    assertThat(user.getId()).isEqualTo(1);
    
    // 测试认证失败场景
    assertThatThrownBy(() -> service.getUserWithToken("invalid-token"))
            .isInstanceOf(AuthException.class);
}
```

**场景3: 文件式 Stub + 编程式 Verification 混合使用**

文件式 Stub 加载的映射同样支持编程式验证。在 5.6 的完整示例基础上，测试方法中可直接调用 `wireMockServer.verify(...)`：

```java
@Test
@DisplayName("文件式Stub-搜索张三并验证请求")
void test_fileStub_withVerification() {
    // Given: mappings/ 和 __files/ 已预置数据，无需 stubFor

    // When: 发送请求
    JSONObject result = HttpClientUtil.post(URL, buildRequest("张三"), headers);

    // Then: 验证响应内容
    assertThat(result).isNotNull();
    // ... 断言响应数据

    // 额外验证: 确认 WireMock 收到了请求
    wireMockServer.verify(postRequestedFor(urlEqualTo("/api/v1/person/combine-search"))
            .withHeader("Content-Type", equalTo("application/json")));
}
```

> 完整测试类结构（`WireMockServer` 初始化、`@BeforeAll`/`@AfterAll`）参见 [5.6 完整文件式 Stub 示例](#56-完整文件式-stub-示例)。

---

## 十、完整示例

### 10.1 WireMock + Mockito 组合示例

#### 支付流程（外部支付网关 + 本地数据库）

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    
    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();
    
    @Mock
    private PaymentRepository repository;
    
    @Mock
    private NotificationService notificationService;
    
    @InjectMocks
    private PaymentService paymentService;
    
    @BeforeEach
    void setUp() {
        paymentService.setApiUrl(wireMock.getRuntimeInfo().getHttpBaseUrl());
    }
    
    @Test
    @DisplayName("组合Mock-支付流程成功")
    void processPayment_success() {
        // WireMock: 模拟外部支付网关
        wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
                .withHeader("Authorization", equalTo("Bearer merchant-token"))
                .withRequestBody(matchingJsonPath("$.amount"))
                .willReturn(okJson("{\"txnId\":\"txn-123\",\"status\":\"SUCCESS\",\"amount\":100.0}")));
        
        // Mockito: 模拟本地数据库和通知服务
        Payment savedPayment = new Payment("txn-123", "SUCCESS", 100.0);
        when(repository.save(any(Payment.class))).thenReturn(savedPayment);
        when(notificationService.sendPaymentNotification(any())).thenReturn(true);
        
        // 执行真实业务逻辑
        Payment result = paymentService.processPayment(100.0, "merchant-token");
        
        // 验证结果
        assertThat(result.getTxnId()).isEqualTo("txn-123");
        assertThat(result.getStatus()).isEqualTo("SUCCESS");
        
        // 双重验证
        wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments"))
                .withHeader("Authorization", equalTo("Bearer merchant-token")));
        verify(repository).save(any(Payment.class));
        verify(notificationService).sendPaymentNotification(any());
    }
    
    @Test
    @DisplayName("组合Mock-支付网关失败")
    void processPayment_gatewayFailed() {
        // WireMock: 模拟支付失败
        wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
                .willReturn(status(400).withJsonBody("{\"error\":\"INSUFFICIENT_BALANCE\"}")));
        
        // 执行并验证异常
        assertThatThrownBy(() -> paymentService.processPayment(100.0, "token"))
                .isInstanceOf(PaymentException.class)
                .hasMessageContaining("INSUFFICIENT_BALANCE");
        
        // 验证 HTTP 请求发送，但本地依赖未调用
        wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments")));
        verify(repository, never()).save(any());
        verify(notificationService, never()).sendPaymentNotification(any());
    }
}
```

### 10.2 RestClient 调用示例

```java
@Test
@DisplayName("RestClient调用-验证请求细节")
void test_restClient_call() {
    stubFor(post(urlPathEqualTo("/api/users"))
            .withHeader("Content-Type", equalTo("application/json"))
            .withRequestBody(matchingJsonPath("$.name"))
            .willReturn(created().withJsonBody("{\"id\":1,\"name\":\"John\"}")));
    
    RestClient client = RestClient.create();
    UserResponse response = client.post()
            .uri(wireMock.getRuntimeInfo().getHttpBaseUrl() + "/api/users")
            .header("Content-Type", "application/json")
            .body("{\"name\":\"John\",\"age\":25}")
            .retrieve()
            .body(UserResponse.class);
    
    assertThat(response.getId()).isEqualTo(1);
    verify(postRequestedFor(urlPathEqualTo("/api/users"))
            .withHeader("Content-Type", equalTo("application/json")));
}
```

### 10.3 文件式 Stub 完整示例

参见 [5.6 完整文件式 Stub 示例](#56-完整文件式-stub-示例)。

---

## 十一、常见问题

### 11.1 端口冲突

**问题**: `java.net.BindException: Address already in use`

**解决**: 使用动态端口

```java
// 编程式 Stub
@RegisterExtension
static WireMockExtension wireMock = WireMockExtension.newInstance()
        .options(wireMockConfig().dynamicPort())
        .build();

// 文件式 Stub（URL 硬编码时需固定端口）
WireMockServer wireMockServer = new WireMockServer(
        WireMockConfiguration.wireMockConfig()
                .port(8089)  // 固定端口
                .withRootDirectory("src/test/resources/wiremock/searchperson")
);
```

### 11.2 Mapping 文件未生效

**问题**: WireMock 启动成功但请求返回 404

**排查步骤**：

1. **检查目录结构**: `mappings/` 和 `__files/` 必须在 `withRootDirectory` 指定的根目录下
2. **检查文件命名**: mapping 文件名与 `bodyFileName` 引用必须完全一致（参见 [2.3 文件命名规范](#23-文件命名规范)）
3. **检查 JSON 格式**: mapping 文件 JSON 语法错误会导致 `MappingFileException`
4. **检查 bodyFileName**: 确保引用的 `__files/` 文件存在且路径正确（相对路径，不含绝对路径前缀）
5. **查看 unmatched 请求**: 通过 `getAllServeEvents(ServeEventQuery.ALL_UNMATCHED)` 查看未匹配的请求
6. **检查请求匹配**: `bodyPatterns` 中的 `equalToJson` 需与实际请求体完全匹配（除非启用 `ignoreExtraElements`）

```java
// 调试：查看未匹配的请求
List<ServeEvent> unmatched = wireMockServer.getAllServeEvents(ServeEventQuery.ALL_UNMATCHED);
unmatched.forEach(event -> System.out.println(event.getRequest()));
```

### 11.3 equalToJson 匹配失败

**问题**: 请求体看起来一样但匹配不到 mapping

**原因**: JSON 序列化差异（字段顺序、空白字符、类型差异）

**解决方案**:

```json
"bodyPatterns" : [ {
  "equalToJson" : "{\"text\":\"张三\",\"pageSize\":10}",
  "ignoreArrayOrder" : true,
  "ignoreExtraElements" : true
} ]
```

- `ignoreArrayOrder: true` — 忽略数组元素顺序
- `ignoreExtraElements: true` — 允许请求体包含额外字段
- 或改用 `matchingJsonPath` 只匹配关键字段

### 11.4 resetMappings 清除文件映射

**问题**: 调用 `resetMappings()` 后文件式 Stub 全部失效

**原因**: `resetMappings()` 清除所有非 `persistent` 的映射

**解决方案**:

- 文件式 mapping 中设置 `"persistent": true`
- 或避免在文件式 Stub 测试中调用 `resetMappings()`
- 或在 `@BeforeEach` 中重新加载文件

### 11.5 中文乱码

**问题**: `__files/` 中的 JSON 文件包含中文，响应乱码

**解决**:

- 确保 mapping 文件的 `response.headers` 中设置 `"Content-Type": "application/json;charset=UTF-8"`
- 确保所有文件使用 UTF-8 编码保存

### 11.6 WireMockServer vs WireMockExtension 选择

| 场景 | 推荐 | 原因 |
|------|------|------|
| 简单编程式 Stub | `WireMockExtension` | 自动生命周期管理 |
| 文件式 Stub（mappings/__files） | `WireMockServer` | 需要 `withRootDirectory()` |
| 录制模式 | `WireMockServer` | 需要 `startRecording()`/`stopRecording()` |
| 固定端口需求 | `WireMockServer` | URL 硬编码时需固定端口 |
| 动态端口 + 编程式 | `WireMockExtension` | 默认动态端口，自动重置 |

---

## 十二、参考资料

- **WireMock 官方文档**: https://wiremock.org/docs/
- **WireMock GitHub**: https://github.com/wiremock/wiremock
- **WireMock Standalone Maven**: https://central.sonatype.com/artifact/org.wiremock/wiremock-standalone
- **JSON Schema (映射文件格式)**: https://github.com/wiremock/wiremock/blob/master/schemas/wiremock-stub-mapping-or-mappings.json
