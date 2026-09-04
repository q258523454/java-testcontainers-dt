# Java DT 用例编写规范

本文档定义 Java 语言容器级 DT 测试编写规范，整合了《SpringBoot单元测试规范参考文档》和业界 Java 开发者测试编程规范，并面向 Testcontainers 容器级 DT 测试场景进行适配。

> **适用环境**：Java 21 + Spring Boot 3.1+ + JUnit 5 + Testcontainers 1.20+ + WireMock Standalone + Docker
>
> **配套文档**：
> - [TestcontainersDT核心规范](./TestcontainersDT核心规范.md) — 容器生命周期、属性注入、数据隔离、反模式
> - [WireMock用例编写规范](./WireMock用例编写规范.md) — WireMock Stub、请求匹配、验证、文件式Stub
> - [MySQL容器测试规范](./component/MySQL容器测试规范.md) — MySQL容器配置、初始化脚本、完整示例
> - [Redis容器测试规范](./component/Redis容器测试规范.md) — Redis容器配置、数据隔离、完整示例

---

## 测试原则

### 【强制】遵守AIR原则（Automatic、Independent、Repeatable）

单元测试必须具备自动化、独立性、可重复执行的特点：

- **Automatic（自动化）**：单元测试应该是全自动执行的，并且非交互式的。Testcontainers容器自动启动/停止，无需人工干预
- **Independent（独立性）**：单元测试用例之间决不能互相调用，也不能依赖执行的先后次序。容器共享但数据隔离，每个测试方法独立准备和清理数据
- **Repeatable（可重复）**：单元测试是可以重复执行的，不能受到外界环境的影响。容器每次启动状态一致，测试结果可重复

**正例（Testcontainers容器级DT）：**

```java
@Test
@DisplayName("正常用例-搜索用户返回正确结果")
void personSearch_normalInput_returnSuccess() {
    // 每个测试独立准备数据，使用真实 MySQL 容器
    User testUser = createTestUser("user_001", "张三", "zhangsan@example.com");
    userRepository.save(testUser);

    PersonSearchResult result = personSearchService.personSearch(createParam("张三"));

    assertThat(result).isNotNull();
    assertThat(result.getUserName()).isEqualTo("张三");
}
```

**反例：**

```java
@Test
void method1_test_returnValue() {
    sharedState = "value1";  // 修改共享状态
}

@Test
void method2_test_checkValue() {
    assertThat(sharedState).isEqualTo("value1");  // 依赖method1的执行结果
}
```

### 【一般】遵守FIRST原则补充要求

FIRST原则中的Independent（独立性）和Repeatable（可重复）已在AIR原则中说明，此处补充其他要求：

- **Fast（快速）**：测试应该快速执行。Testcontainers使用Singleton Container模式避免重复启动容器，使用Alpine镜像减少启动开销
- **Self-Validating（自验证）**：测试应该自动判断通过或失败，不需要人工检查。使用AssertJ断言，禁止System.out人肉验证
- **Timely（及时）**：测试应该及时编写，与开发同步进行

---

## 命名规范

### 【强制】测试类命名与包路径规范

测试类名称采用`被测试类名 + Test`的形式，与被测试类放在同一包结构下，便于访问包级私有成员。单元测试代码必须写在`src/test/java`目录下，不允许写在业务代码目录下。

**正例：**

```
src/main/java/com/example/search/proxy/delegate/PersonSearchDelegateImpl.java
src/test/java/com/example/search/proxy/delegate/PersonSearchDelegateImplTest.java           # UT 单元测试
src/test/java/com/example/search/proxy/delegate/PersonSearchDelegateImplTest.java # 容器级 DT 测试

命名示例：
PersonSearchDelegateImpl → PersonSearchDelegateImplTest              # UT 单元测试
PersonSearchDelegateImpl → PersonSearchDelegateImplTest              # 容器级 DT 测试
UserServiceImpl → UserServiceImplTest / UserServiceImplTest
OrderController → OrderControllerTest / OrderControllerTest
StatusCodeEnum → StatusCodeEnumTest
```

**反例：**

```
src/main/java/com/example/search/proxy/delegate/PersonSearchDelegateImplTest.java  # 测试代码写在业务目录

命名反例：
PersonSearchDelegateImpl → PersonSearchTest（缺少Impl）
UserServiceImpl → UserServiceTest（缺少Impl）
OrderController → TestOrderController（前缀Test）
```

### 【强制】测试方法命名采用蛇形命名法，清晰表达"测什么、什么情况、期望什么"

测试方法名称采用`方法名_测试场景_预期行为`的蛇形命名法，清晰表达测试意图。格式为：`被测方法名_测试场景描述_预期结果`。

**命名结构：**
- **方法名**：被测试的方法名（如 `personSearch`、`calculateDiscount`）
- **测试场景**：具体的测试条件或输入状态（如 `normalInput`、`nullParam`、`vipUser`）
- **预期行为**：期望的输出或行为（如 `returnSuccess`、`throwException`、`returnDiscount`）

**正例：**

```java
personSearch_normalInput_returnSuccess()
personSearch_emptyParam_throwException()
calculateDiscount_vipUser_returnDiscount()
validateToken_expiredToken_throwException()
execute_fileNotExist_throwFileNotFoundException()
withdrawMoney_invalidAccount_throwInvalidOperationException()
```

**反例：**

```java
test1()、test2()  // 无意义编号
testPersonSearch()  // 缺少场景和预期行为
personSearch()  // 缺少场景和预期行为
personSearchSuccessTest()  // 驼峰命名，不符合蛇形命名法
testExecute01()  // 无法体现用例关键信息
execute_should_fail_when_file_not_exist()  // 过长，不够简洁
```

### 【强制】测试方法必须使用@DisplayName注解提供可读描述

测试方法应当使用`@DisplayName`注解提供清晰的业务场景描述，帮助理解测试意图。描述格式为"用例类型-业务场景描述"。

**正例：**

```java
@Test
@DisplayName("正常用例-搜索成功并返回结果")
void personSearch_normalInput_returnSuccess() {
    // 测试代码
}

@Test
@DisplayName("异常用例-搜索userId为null并抛出TOKEN_VALID异常")
void personSearch_nullUserId_throwTokenValidException() {
    // 测试代码
}
```

**反例：**

```java
@Test
void personSearch_normalInput_returnSuccess() {  // 缺少@DisplayName注解
    // 测试代码
}

@Test
@DisplayName("测试搜索功能")  // 描述过于笼统，未明确用例类型和具体场景
void personSearch_normalInput_returnSuccess() {
    // 测试代码
}
```

---

## 用例结构规范

### 【强制】用例结构符合GWT原则（Given、When、Then）

每个测试方法遵循「准备Given → 执行When → 断言Then」三段式结构，根据需要增加第四段verify。

**正例（Testcontainers容器级DT）：**

```java
@Test
@DisplayName("正常用例-根据ID查询用户返回正确结果")
@Sql(scripts = "classpath:data/mysql/fixtures/global/init_xxx.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "classpath:data/mysql/fixtures/global/cleanup_xxx.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void findById_existingUser_returnUser() {
    // 1. Given：使用真实 MySQL 容器准备测试数据
    User testUser = createTestUser("user_001", "张三", "zhangsan@example.com");
    userRepository.save(testUser);

    // 2. When：调用被测试方法
    User result = userService.findById("DT_TEST_user_001");

    // 3. Then：验证真实数据库查询结果
    assertThat(result).isNotNull();
    assertThat(result.getUserId()).isEqualTo("DT_TEST_user_001");
    assertThat(result.getUserName()).isEqualTo("张三");
    assertThat(result.getEmail()).isEqualTo("zhangsan@example.com");

    // 4. verify（可选）
    assertThat(userRepository.existsById("DT_TEST_user_001")).isTrue();
}
```

**正例（WireMock + Mockito组合场景）：**

```java
@Test
@DisplayName("组合Mock-支付流程成功")
@Sql(scripts = "classpath:data/mysql/fixtures/global/init_xxx.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "classpath:data/mysql/fixtures/global/cleanup_xxx.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void processPayment_normalInput_returnSuccess() {
    // 1. Given：WireMock模拟外部支付网关 + Mockito模拟本地依赖
    wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
            .withHeader("Authorization", equalTo("Bearer merchant-token"))
            .willReturn(okJson("{\"txnId\":\"txn-123\",\"status\":\"SUCCESS\"}")));

    Payment savedPayment = new Payment("txn-123", "SUCCESS", 100.0);
    when(repository.save(any(Payment.class))).thenReturn(savedPayment);

    // 2. When：调用被测试方法
    Payment result = paymentService.processPayment(100.0, "merchant-token");

    // 3. Then：验证业务结果
    assertThat(result.getTxnId()).isEqualTo("txn-123");
    assertThat(result.getStatus()).isEqualTo("SUCCESS");

    // 4. verify：双重验证（WireMock HTTP验证 + Mockito 调用验证）
    wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments"))
            .withHeader("Authorization", equalTo("Bearer merchant-token")));
    verify(repository).save(any(Payment.class));
}
```

### 【强制】测试用例必须包含至少一例正例，多个反例，并且正例写到第一个

测试类中必须包含至少一个正例（正常场景测试）和多个反例（异常场景测试），并且正例测试方法应写在第一个，便于快速理解正常业务流程。

**正例：**

```java
class UserServiceImplTest extends AbstractMySQLTestcontainers {

    // ===== 正例（写在第一个） =====

    @Test
    @DisplayName("正常用例-根据ID查询用户返回正确结果")
    void findById_existingUser_returnUser() {
        // 正常场景测试
    }

    // ===== 反例 =====

    @Test
    @DisplayName("异常用例-查询不存在的用户返回空")
    void findById_nonExistentUser_returnEmpty() {
        // 异常场景1
    }

    @Test
    @DisplayName("异常用例-创建重复用户抛出唯一约束异常")
    void createUser_duplicateUser_throwException() {
        // 异常场景2
    }
}
```

**反例：**

```java
class PersonSearchDelegateImplTest {

    // 反例写在第一个，正例在后面
    @Test
    void personSearch_nullParam_throwException() {
        assertThatThrownBy(() -> personSearchDelegate.personSearch(null))
            .isInstanceOf(SearchException.class);
    }

    @Test
    void personSearch_normalInput_returnSuccess() {  // 正例应该写在第一个
        // 正常场景测试
    }

    // 只有反例，缺少正例
    @Test
    void personSearch_invalidParam_throwException() { }

    // 只有正例，缺少反例
    @Test
    void personSearch_normalInput_returnSuccess() { }
}
```

### 【强制】单个测试方法不超过150行

单个测试方法不超过150行。如果测试逻辑复杂，拆分为多个测试方法。

**反例：**

```java
// 一个测试验证多个场景（超过50行）
@Test
void personSearch_allScenarios_returnVariousResults() {
    // 场景一：正常查询
    // ... 20行代码

    // 场景二：参数为空
    // ... 20行代码

    // 场景三：Token失效
    // ... 20行代码
}
```

---

## 断言规范

### 【强制】用例必须要有断言

单元测试中的断言，是用来验证程序行为正确性的一种机制。假如一个用例没有任何断言，仅依靠被测代码执行是否抛异常来判断程序行为是否正确，那么该用例将无法发现逻辑上的问题。

**反例：**

```java
@Test
void personSearch_normalInput_returnSuccess() {
    when(userRepository.findById("user_123"))
        .thenReturn(Optional.of(createMockUser()));

    PersonSearchResult result = personSearchDelegate.personSearch(createParam());

    // 缺少断言
}

@Test
void save_normalInput_returnSuccess() {
    // 使用assert关键字进行断言
    assert components.size() == 1;
}
```

### 【强制】禁止永不失败的断言

若断言中存在变量为固定值等问题，会导致结果永远为true，断言永不失败。当代码出现问题时，测试结果仍然通过，不会给出相应的错误告警。

**反例：**

```java
@Test
void save_normalInput_alwaysSuccess() {
    // 此断言永远成功，毫无意义
    assertTrue(true);
}
```

### 【强制】禁止在用例中提前返回

在用例中提前返回，可能导致用例没有真正运行到被测对象、结果校验代码块，而用例执行结果仍然通过，无法起到验证被测对象正确性的作用。

**反例：**

```java
@Test
void generateWordReport_normalInput_createFile() {
    List<WordData> allWordData = new ArrayList<>();
    prepareData(allWordData);

    WordCommonUtils.generateWordReport(SOURCE_TEMPLATE_PATH, DEST_REPORT_PATH, allWordData);

    File generatedFile = new File(DEST_REPORT_PATH);
    boolean isFileExist = generatedFile.exists();

    if (isFileExist) {
        // 不应当由于某个条件导致用例提前返回
        return;
    }

    // 其他测试代码 ...
}
```

### 【强制】使用AssertJ流式断言，禁止使用System.out人肉验证

单元测试必须使用assert来验证，不允许使用System.out来进行人肉验证。推荐使用AssertJ流式断言，而非JUnit原生断言。

**正例（容器级DT断言真实数据库结果）：**

```java
// AssertJ流式断言 — 验证真实 MySQL 查询结果
assertThat(result).isNotNull();
assertThat(result.getUserId()).isEqualTo("DT_TEST_user_001");
assertThat(result.getUserName()).startsWith("张");
assertThat(result.getAge()).isBetween(18, 60);

// 验证真实数据库持久化
User dbUser = userRepository.findById("DT_TEST_user_001").orElseThrow();
assertThat(dbUser.getUserName()).isEqualTo("张三");
assertThat(dbUser.getCreatedAt()).isNotNull();
```

**反例：**

```java
// JUnit原生断言（不推荐）
Assertions.assertNotNull(result);
Assertions.assertEquals("user_123", result.getUserId());

// 使用System.out人肉验证
System.out.println(result.getUserId());
```

---

## Mock使用规范

### 【强制】使用@Mock注解和MockitoExtension初始化Mock对象

使用`@Mock`注解和`@ExtendWith(MockitoExtension.class)`初始化Mock对象，而非手动创建Mock。

**正例：**

```java
@ExtendWith(MockitoExtension.class)
class PersonSearchDelegateImplTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private TokenService tokenService;

    @InjectMocks
    private PersonSearchDelegateImpl personSearchDelegate;

    @Test
    void personSearch_normalInput_returnSuccess() {
        when(userRepository.findById("user_123"))
            .thenReturn(Optional.of(createMockUser()));

        PersonSearchResult result = personSearchDelegate.personSearch(createParam());
        assertThat(result).isNotNull();
    }
}
```

**反例：**

```java
// 手动创建Mock（不推荐）
class PersonSearchDelegateImplTest {

    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository = Mockito.mock(UserRepository.class);
        personSearchDelegate = new PersonSearchDelegateImpl();
        personSearchDelegate.setUserRepository(userRepository);
    }
}
```

### 【强制】Mockito、WireMock与Testcontainers分工边界

容器级DT测试中，三类工具各有职责，禁止混用：

| 工具 | 职责 | Mock内容 | 典型示例 |
|------|------|---------|---------|
| **Testcontainers** | 真实中间件 | 不Mock，使用真实MySQL/Redis/ES容器 | 验证SQL语法、JPA映射、Redis序列化 |
| **WireMock** | 外部HTTP服务 | HTTP响应、状态码、Header | 支付API、短信服务、OAuth认证 |
| **Mockito** | 本地依赖 | 数据返回、异常抛出 | `when(repo.findById()).thenReturn()` |

**分工决策树：**

```
测试场景是否涉及中间件（MySQL/Redis/ES）？
├─ YES → Testcontainers（真实容器，禁止Mock）
│   └─ 需要验证SQL语法、序列化、映射正确性
└─ NO → 是否涉及外部HTTP调用？
    ├─ YES → WireMock（模拟外部API响应）
    └─ NO → Mockito（模拟本地依赖）
```

**正例（三者组合使用）：**

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest extends AbstractMySQLTestcontainers {

    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort().configureStaticDsl(true))
            .build();

    @Mock
    private NotificationService notificationService;  // Mockito: 本地通知服务

    @Autowired
    private OrderRepository orderRepository;            // Testcontainers: 真实 MySQL

    @Autowired
    private OrderService orderService;

    @Test
    @DisplayName("组合Mock-创建订单并调用外部支付API")
    void createOrder_normalInput_returnSuccess() {
        // 1. WireMock: 模拟外部支付网关
        wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
                .willReturn(okJson("{\"txnId\":\"txn-123\",\"status\":\"SUCCESS\"}")));

        // 2. Mockito: 模拟本地通知服务
        when(notificationService.sendNotification(any())).thenReturn(true);

        // 3. Testcontainers: 真实MySQL持久化
        Order result = orderService.createOrder(createOrderParam(100.0));

        // 4. 验证真实数据库状态
        Order dbOrder = orderRepository.findById(result.getId()).orElseThrow();
        assertThat(dbOrder.getStatus()).isEqualTo("PAID");
        assertThat(dbOrder.getPaymentTxnId()).isEqualTo("txn-123");

        // 5. 双重验证
        wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments")));
        verify(notificationService).sendNotification(any());
    }
}
```

**反例：**

```java
// 在容器级DT中Mock容器化基础设施，失去真实验证价值
@Mock
private RedisService redisService;  // 应使用真实Redis容器
when(redisService.get("key")).thenReturn(value);

// 在容器级DT中过度断言Mock行为
verify(repository, times(1)).findById(any());  // 应验证真实数据库结果
```

### 【强制】禁止在DT中Mock容器化基础设施

容器级DT的核心价值是使用真实中间件验证代码行为。Mock容器化基础设施会使DT失去意义。详细反模式说明见 [TestcontainersDT核心规范 - 反模式](./TestcontainersDT核心规范.md#八反模式)。

**正例：**

```java
// 使用真实 Redis 容器
@Autowired
private RedisTemplate<String, Object> redisTemplate;

@Test
void getUser_cacheHit_returnCachedData() {
    redisTemplate.opsForValue().set(cacheKey, cachedUser);
    User result = userCacheService.getUser("DT_TEST_user_001");
    assertThat(result).isNotNull();
    assertThat(redisTemplate.hasKey(cacheKey)).isTrue();
}
```

**反例：**

```java
// Mock Redis，失去真实验证价值
@Mock
private RedisService redisService;
when(redisService.get("key")).thenReturn(value);
```

### 【一般】避免过度Mock

只Mock必要的依赖，其他依赖使用真实对象。在容器级DT中，中间件依赖应使用真实容器而非Mock。

**正例：**

```java
@Test
void personSearch_normalInput_returnSuccess() {
    when(userRepository.findById("user_123"))
        .thenReturn(Optional.of(createMockUser()));

    // 只Mock必要的依赖，其他依赖使用真实对象
}
```

**反例：**

```java
@Test
void personSearch_normalInput_returnSuccess() {
    when(userRepository.findById("user_123"))
        .thenReturn(Optional.of(createMockUser()));
    when(userRepository.count())  // 不需要Mock
        .thenReturn(100L);
    when(userRepository.findAll())  // 不需要Mock
        .thenReturn(Collections.emptyList());
}
```

### 【一般】Mock返回值与测试数据应贴近真实场景

Mock返回值和测试数据应贴近真实业务场景，避免使用空对象、明显不合理的值或过于简单的测试数据。

**正例：**

```java
// Mock返回真实数据
when(userRepository.findById("user_123"))
    .thenReturn(Optional.of(createMockUser("张三", "zhangsan@example.com")));

// 测试数据贴近真实场景
User user = createMockUser("张三", "zhangsan@example.com");
Order order = createOrder("order_001", 1000, 3);
```

**反例：**

```java
// Mock返回空对象或不合理数据
when(userRepository.findById("user_123"))
    .thenReturn(Optional.empty());

when(userRepository.findById("user_123"))
    .thenReturn(Optional.of(new User()));  // 空对象，缺少必要属性

// 不合理的测试数据
User user = createMockUser("test", "test@test.com");  // 太简单
Order order = createOrder("1", 0, 0);  // 不合理值
User user = createMockUser("张三三三三三三", "a");  // 超长、无效邮箱
```

### 【强制】禁止Mock被测试类本身

Mock只针对外部依赖，不要Mock被测试类的部分方法。

**正例：**

```java
@InjectMocks
private PersonSearchDelegateImpl personSearchDelegate;

@Mock
private UserRepository userRepository;

@Test
void personSearch_normalInput_returnSuccess() {
    when(userRepository.findById("user_123"))
        .thenReturn(Optional.of(createMockUser()));

    PersonSearchResult result = personSearchDelegate.personSearch(createParam());
    assertThat(result).isNotNull();  // 验证真实逻辑
}
```

**反例：**

```java
// Mock被测试类本身
@Mock
private PersonSearchDelegateImpl personSearchDelegate;

@Test
void personSearch_normalInput_returnSuccess() {
    when(personSearchDelegate.personSearch(any()))
        .thenReturn(createResult());  // 这不是单元测试

    PersonSearchResult result = personSearchDelegate.personSearch(createParam());
    assertThat(result).isNotNull();  // 没有验证真实逻辑
}
```

---


## 数据库规范

### 【强制】数据库使用Testcontainers MySQL容器，真实模拟

使用Testcontainers MySQL容器替代H2等嵌入式数据库，确保SQL语法、JPA映射、事务边界与生产环境一致。

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

<!-- MySQL 模块 -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>

<!-- Spring Boot Testcontainers 集成（Spring Boot 3.1+） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

> **推荐**：使用`testcontainers-bom`统一管理版本，详见[pom-dependencies.md](../assets/config-templates/pom-dependencies.md)

### 【强制】数据库脚本统一放在`src/test/resources/data/mysql/`目录下

采用`fixtures / migrations / seeds / testcontainers`四层组织，详见[DT目录结构规范](./DT目录结构规范.md)。

```
src/test/resources/data/mysql/
├── fixtures/
│   ├── global/          # 全局夹具（建表+基础数据，变更频率低）
│   ├── modules/         # 模块夹具（按业务模块复用）
│   └── tests/           # 用例夹具（特定测试专用）
├── migrations/          # 迁移脚本（与生产环境严格一致）
├── seeds/               # 种子数据
└── testcontainers/
    └── image/           # 自定义镜像构建资源
```

### 【强制】数据处理必须使用@AfterEach还原现场，保证测试独立性

容器在测试类间共享，但每个测试方法的数据必须独立。三种数据隔离策略详见 [TestcontainersDT核心规范 - 数据隔离策略](./TestcontainersDT核心规范.md#五数据隔离策略)。

### 【强制】测试数据使用统一前缀

```java
private User createTestUser(String userId, String userName, String email) {
    User user = new User();
    user.setUserId("DT_TEST_" + userId);  // 统一前缀
    user.setUserName(userName);
    user.setEmail(email);
    return user;
}

@AfterEach
void tearDown() {
    userRepository.deleteByUserIdStartingWith("DT_TEST_");
}
```

### 【一般】数据库测试数据准备与清理

对于数据库相关的测试，必须使用程序插入或导入方式准备数据，并在测试完成后清理现场，避免影响其他用例。不能假设数据库数据存在，也不能直接操作数据库插入不符合业务规则的数据。

**正例：**

```java
@BeforeEach
void setUp() {
    // 使用Repository插入测试数据
    User testUser = createTestUser("user_001", "张三", "zhangsan@example.com");
    userRepository.save(testUser);
}

@AfterEach
void tearDown() {
    // 清理测试数据，还原现场
    userRepository.deleteByUserIdStartingWith("DT_TEST_");
}
```

**反例：**

```java
@Test
void deleteUser_existingUser_deleteSuccess() {
    // 直接删除数据库中已有的数据，可能不存在或不符合业务规则
    userService.deleteUser("existing_user_id");
}
```

---

## HTTP请求模拟规范

> **详细规范请参考**：[WireMock用例编写规范](./WireMock用例编写规范.md)

### 【强制】HTTP请求模拟使用WireMock Standalone组件

使用WireMock模拟外部HTTP服务（第三方API、微服务调用），而非Mockito Mock HTTP客户端。

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <scope>test</scope>
</dependency>
```

### 【强制】WireMock编程式Stub使用@RegisterExtension

使用`@RegisterExtension`和`WireMockExtension`管理WireMock生命周期，动态端口避免冲突。

**正例：**

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort().configureStaticDsl(true))
            .build();

    @Mock
    private PaymentRepository repository;

    @InjectMocks
    private PaymentService paymentService;

    @BeforeEach
    void setUp() {
        paymentService.setApiUrl(wireMock.getRuntimeInfo().getHttpBaseUrl());
    }

    @Test
    @DisplayName("正常用例-支付流程成功")
    void processPayment_normalInput_returnSuccess() {
        // 1. Given：WireMock模拟外部支付网关
        wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
                .withHeader("Authorization", equalTo("Bearer merchant-token"))
                .withRequestBody(matchingJsonPath("$.amount"))
                .willReturn(okJson("{\"txnId\":\"txn-123\",\"status\":\"SUCCESS\"}")));

        // Mockito模拟本地数据库
        Payment savedPayment = new Payment("txn-123", "SUCCESS", 100.0);
        when(repository.save(any(Payment.class))).thenReturn(savedPayment);

        // 2. When：执行真实业务逻辑
        Payment result = paymentService.processPayment(100.0, "merchant-token");

        // 3. Then：验证业务结果
        assertThat(result.getTxnId()).isEqualTo("txn-123");
        assertThat(result.getStatus()).isEqualTo("SUCCESS");

        // 4. verify：双重验证
        wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments"))
                .withHeader("Authorization", equalTo("Bearer merchant-token")));
        verify(repository).save(any(Payment.class));
    }
}
```

### 【一般】文件式Stub用于复杂响应体

当响应体较大或需要录制回放时，使用`WireMockServer` + 文件式Stub（`mappings/` + `__files/`）。

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class WireMockFileTest {

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
}
```

> **文件命名规范**：`{API端点}-{场景描述}-{响应类型}.json`，详见[WireMock用例编写规范](./WireMock用例编写规范.md)

### 【一般】错误场景模拟

WireMock可模拟多种外部HTTP错误场景，覆盖异常测试需求：

```java
// 模拟500服务器错误
wireMock.stubFor(get("/api/error")
        .willReturn(serverError()
                .withJsonBody("{\"error\":\"Internal Server Error\"}")));

// 模拟超时（5秒延迟）
wireMock.stubFor(get("/api/timeout")
        .willReturn(aResponse()
                .withStatus(200)
                .withFixedDelay(5000)));

// 模拟连接重置
wireMock.stubFor(get("/api/reset")
        .willReturn(aResponse()
                .withFault(Fault.CONNECTION_RESET_BY_PEER)));

// 模拟401未授权
wireMock.stubFor(get("/api/users")
        .withHeader("Authorization", equalTo("Bearer invalid-token"))
        .willReturn(unauthorized()));
```

### 【一般】WireMock请求验证

```java
// 基础验证
wireMock.verify(postRequestedFor(urlEqualTo("/api/payments")));

// 次数验证
wireMock.verify(exactly(1), postRequestedFor(urlEqualTo("/api/payments")));
wireMock.verify(0, postRequestedFor(urlEqualTo("/api/payments")));  // 验证零次

// 详细验证
wireMock.verify(postRequestedFor(urlEqualTo("/api/payments"))
        .withHeader("Content-Type", equalTo("application/json"))
        .withRequestBody(matchingJsonPath("$.amount")));
```

---

## Redis处理规范

> **详细规范请参考**：[Redis容器测试规范](./component/Redis容器测试规范.md)

### 【强制】Redis使用Testcontainers Redis容器，真实模拟

使用Testcontainers Redis容器替代jedis-mock等嵌入式替代品，确保序列化、TTL过期、Lua脚本等行为与生产环境一致。

Testcontainers无专用Redis模块，使用`GenericContainer` + 官方Redis镜像：

```xml
<!-- Testcontainers 核心（已包含在父依赖中） -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// Redis容器配置
static final GenericContainer<?> REDIS = new GenericContainer<>(
    DockerImageName.parse("redis:7-alpine"))
    .withExposedPorts(6379);
```

### 【强制】Redis属性注入使用@DynamicPropertySource

```java
@DynamicPropertySource
static void redisProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.data.redis.host", REDIS::getHost);
    registry.add("spring.data.redis.port", REDIS::getFirstMappedPort);
}
```

### 【强制】Redis数据隔离与清理

每个测试方法前后清理Redis数据，保证测试独立性。

```java
@Autowired
protected RedisTemplate<String, Object> redisTemplate;

private static final String KEY_PREFIX = "DT_TEST:";

@BeforeEach
void setUp() {
    // 清空所有数据，确保测试隔离
    redisTemplate.getConnectionFactory().getConnection().flushAll();
}

@AfterEach
void tearDown() {
    // 按前缀批量删除测试数据
    Set<String> keys = redisTemplate.keys(KEY_PREFIX + "*");
    if (keys != null && !keys.isEmpty()) {
        redisTemplate.delete(keys);
    }
}

// 测试Key使用统一前缀
String cacheKey = KEY_PREFIX + "user:" + userId;  // DT_TEST:user:001
```

### 【一般】Redis缓存测试完整示例

```java
@Test
@DisplayName("正常用例-缓存未命中查询数据库并回填缓存")
void getUser_cacheMiss_queryDbAndFillCache() {
    // Given：缓存为空（setUp中已flushAll）
    String cacheKey = KEY_PREFIX + "user:DT_TEST_user_001";

    // When
    User result = userCacheService.getUser("DT_TEST_user_001");

    // Then：验证数据库查询 + 缓存回填（真实Redis操作）
    assertThat(result).isNotNull();
    assertThat(result.getUserName()).isEqualTo("张三");
    assertThat(redisTemplate.hasKey(cacheKey)).isTrue();
    Object cachedValue = redisTemplate.opsForValue().get(cacheKey);
    assertThat(cachedValue).isNotNull();
}

@Test
@DisplayName("正常用例-TTL过期后重新查询数据库")
void getUser_ttlExpired_queryDbAgain() throws InterruptedException {
    // Given：设置短TTL
    String cacheKey = KEY_PREFIX + "user:DT_TEST_user_001";
    User cachedUser = createTestUser("user_001", "张三");
    redisTemplate.opsForValue().set(cacheKey, cachedUser, Duration.ofSeconds(1));

    // When：等待TTL过期
    Thread.sleep(1100);
    User result = userCacheService.getUser("DT_TEST_user_001");

    // Then：验证TTL过期后重新查询
    assertThat(result).isNotNull();
}
```

---

## 测试粒度与覆盖率规范

### 【强制】测试粒度要足够小，遵循单一原则，一次只验证一个功能

单元测试要保证测试粒度足够小，有助于精确定位问题。单测粒度至多是类级别，一般是方法级别。

### 【强制】覆盖率要求

新增代码：
- 方法覆盖率要求100%
- 代码行覆盖率要求100%
- 分支覆盖率要求100%

### 【强制】所有测试用例必须执行成功，验证成功

所有测试用例必须能够成功执行并通过验证，不允许存在失败的测试用例。测试失败意味着被测代码存在问题或测试本身编写错误，必须立即修复。

**反例：**

```java
// 测试用例执行失败（断言不通过）
@Test
void personSearch_normalInput_returnSuccess() {
    PersonSearchResult result = personSearchDelegate.personSearch(createParam());
    assertThat(result.getUserId()).isEqualTo("wrong_id");  // 断言失败
}

// 测试用例执行失败（抛出未预期的异常）
@Test
void personSearch_normalInput_returnSuccess() {
    PersonSearchResult result = personSearchDelegate.personSearch(null);  // 抛出异常
    assertThat(result).isNotNull();
}

// 使用@Disabled跳过失败的测试（不推荐）
@Test
@Disabled("临时跳过，待修复")  // 不应使用@Disabled掩盖问题
void personSearch_normalInput_returnSuccess() {
    // 失败的测试代码
}
```

**验证要求：**

1. 所有测试用例必须执行通过（绿色状态）
2. 不允许使用`@Disabled`注解跳过失败的测试
3. 测试失败时必须立即修复被测代码或测试代码
4. 提交代码前必须确保所有测试用例执行成功

---

## 异常测试规范

### 【强制】必须测试异常场景并验证异常内容

必须测试以下异常场景，并验证异常的类型、错误码和消息内容：

- 参数校验失败
- 业务规则违反
- 外部依赖异常
- 空值/空集合处理
- 边界值异常

**正例（Testcontainers容器级DT — 真实数据库约束验证）：**

```java
// 参数校验失败
@Test
@DisplayName("异常用例-搜索参数为null并抛出异常")
void personSearch_nullParam_throwException() {
    assertThatThrownBy(() -> personSearchService.personSearch(null))
        .isInstanceOf(SearchException.class)
        .hasMessageContaining("TEXT_BLANK");
}

// 业务规则违反 — 真实MySQL唯一约束
@Test
@DisplayName("异常用例-创建重复用户抛出唯一约束异常")
void createUser_duplicateUser_throwException() {
    // Given：setUp中已插入DT_TEST_user_001
    User duplicate = createTestUser("user_001", "重复", "dup@test.com");

    // When & Then：验证真实数据库唯一约束
    assertThatThrownBy(() -> userService.createUser(duplicate))
        .isInstanceOf(RuntimeException.class);
}

// 外部依赖异常 — WireMock模拟外部API错误
@Test
@DisplayName("异常用例-支付网关返回500时抛出业务异常")
void processPayment_gatewayError_throwException() {
    // Given：WireMock模拟外部支付网关500错误
    wireMock.stubFor(post(urlPathEqualTo("/api/payments"))
            .willReturn(serverError()
                    .withJsonBody("{\"error\":\"INTERNAL_ERROR\"}")));

    // When & Then
    assertThatThrownBy(() -> paymentService.processPayment(100.0, "token"))
        .isInstanceOf(PaymentException.class)
        .hasMessageContaining("INTERNAL_ERROR");

    // verify：验证HTTP请求已发送，但本地依赖未调用
    wireMock.verify(postRequestedFor(urlPathEqualTo("/api/payments")));
    verify(repository, never()).save(any());
}

// 验证异常类型、错误码和消息内容
@Test
@DisplayName("异常用例-Token过期抛出异常并验证错误码")
void validateToken_expiredToken_throwException() {
    String expiredToken = "expired_token_abc";

    assertThatThrownBy(() -> tokenService.validateToken(expiredToken))
        .isInstanceOf(SearchException.class)  // 验证异常类型
        .hasFieldOrPropertyWithValue("code", "62000")  // 验证错误码
        .hasMessageContaining("token失效")  // 验证中文消息
        .hasMessageContaining("token invalid");  // 验证英文消息
}
```

---

## 日志规范

### 【强制】禁止使用System.out、print等打印日志，必要日志使用log.info、log.error

### 【强制】测试结果禁止使用日志进行人工判断，必须使用assert断言

---

**说明：**

1. 本规范整合了《SpringBoot单元测试规范参考文档》和业界 Java 开发者测试编程规范
2. 【强制】标识的规则必须严格遵守，违反将导致代码审查不通过
3. 【一般】标识的规则建议遵守，违反可能导致潜在问题
4. 本规范适用于所有Spring Boot 3.x + Java 21 + JUnit 5 + Testcontainers 1.20+ + WireMock Standalone项目
5. 容器级DT详细规范请参考[TestcontainersDT核心规范](./TestcontainersDT核心规范.md)
6. WireMock详细用法请参考[WireMock用例编写规范](./WireMock用例编写规范.md)
7. 各中间件容器配置请参考[MySQL容器测试规范](./component/MySQL容器测试规范.md)、[Redis容器测试规范](./component/Redis容器测试规范.md)、[Elasticsearch容器测试规范](./component/Elasticsearch容器测试规范.md)
