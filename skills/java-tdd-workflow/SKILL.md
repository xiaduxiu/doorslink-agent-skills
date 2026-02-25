---
name: java-tdd-workflow
description: Java Spring Boot项目的测试驱动开发工作流。使用JUnit 5、Mockito、MockMvc进行单元测试、集成测试，覆盖率要求80%+。适用于编写新功能、修复bug、重构代码。
---

# Java TDD 工作流

本技能确保所有Java代码开发遵循TDD原则，并具备全面的测试覆盖率。

## 何时激活

* 编写新功能或业务逻辑
* 修复bug或问题
* 重构现有代码
* 添加REST API端点
* 创建新Service/Mapper/Controller

## 核心原则

### 1. 测试优先于代码

始终先编写测试，然后实现代码以使测试通过。

### 2. 覆盖率要求

* 最低80%覆盖率（行覆盖 + 分支覆盖）
* 覆盖所有边界情况
* 测试异常场景
* 验证参数校验

### 3. 测试类型

#### 单元测试

* Service层业务逻辑
* 工具类（Utils）
* 纯函数
* 辅助方法

#### 集成测试

* Controller层（使用MockMvc）
* 数据库操作（使用@MybatisTest或@DataJpaTest）
* Mapper接口
* 外部服务调用

## TDD 工作流步骤

### 步骤 1: 编写用户故事

```
作为 [角色]，我想要 [功能]，以便 [收益]

示例：
作为用户，我想要通过关键词搜索商品，
以便快速找到我需要的商品。
```

### 步骤 2: 生成测试用例

针对每个用户故事，创建全面的测试用例：

```java
@DisplayName("商品搜索服务测试")
class ProductSearchServiceTest {

  @Test
  @DisplayName("根据关键词返回相关商品")
  void shouldReturnRelevantProductsForKeyword() {
    // 测试实现
  }

  @Test
  @DisplayName("空关键词时返回所有商品")
  void shouldReturnAllProductsWhenKeywordIsEmpty() {
    // 测试边界情况
  }

  @Test
  @DisplayName("数据库异常时返回空列表")
  void shouldReturnEmptyListWhenDatabaseError() {
    // 测试异常处理
  }

  @Test
  @DisplayName("按价格排序返回结果")
  void shouldSortResultsByPrice() {
    // 测试排序逻辑
  }
}
```

### 步骤 3: 运行测试（应该失败）

```bash
# Maven
mvn test

# Gradle
./gradlew test

# 测试应该失败 - 我们还没有实现代码
```

### 步骤 4: 实现代码

编写最少的代码以使测试通过：

```java
@Service
public class ProductSearchService {

  @Autowired
  private ProductMapper productMapper;

  public List<Product> searchByKeyword(String keyword) {
    // 根据测试驱动实现
    if (StringUtils.isBlank(keyword)) {
      return productMapper.selectAll();
    }
    return productMapper.searchByName(keyword);
  }
}
```

### 步骤 5: 再次运行测试

```bash
# Maven
mvn test

# Gradle
./gradlew test

# 测试现在应该通过
```

### 步骤 6: 重构

在保持测试通过的同时提高代码质量：

* 消除重复代码
* 改进命名
* 优化性能
* 增强可读性

### 步骤 7: 验证覆盖率

```bash
# Maven + JaCoCo
mvn clean test jacoco:report

# 查看覆盖率报告
cat target/site/jacoco/index.html

# 验证是否达到80%+覆盖率
```

## 测试模式

### Service 单元测试模式

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("用户服务测试")
class UserServiceTest {

  @Mock
  private UserMapper userMapper;

  @Mock
  private PasswordEncoder passwordEncoder;

  @InjectMocks
  private UserService userService;

  @Test
  @DisplayName("根据ID获取用户成功")
  void shouldGetUserByIdSuccessfully() {
    // Arrange
    Long userId = 1L;
    User mockUser = new User();
    mockUser.setId(userId);
    mockUser.setUsername("testuser");
    when(userMapper.selectById(userId)).thenReturn(mockUser);

    // Act
    User result = userService.getById(userId);

    // Assert
    assertNotNull(result);
    assertEquals("testuser", result.getUsername());
    verify(userMapper).selectById(userId);
  }

  @Test
  @DisplayName("用户不存在时抛出异常")
  void shouldThrowExceptionWhenUserNotFound() {
    // Arrange
    Long userId = 999L;
    when(userMapper.selectById(userId)).thenReturn(null);

    // Act & Assert
    assertThrows(UserNotFoundException.class, () -> {
      userService.getById(userId);
    });
  }

  @Test
  @DisplayName("注册用户时密码加密")
  void shouldEncryptPasswordWhenRegisterUser() {
    // Arrange
    UserRegisterRequest request = new UserRegisterRequest();
    request.setUsername("newuser");
    request.setPassword("plainpassword");
    when(passwordEncoder.encode("plainpassword")).thenReturn("hashedpassword");

    // Act
    userService.register(request);

    // Assert
    ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
    verify(userMapper).insert(userCaptor.capture());
    assertEquals("hashedpassword", userCaptor.getValue().getPassword());
  }
}
```

### Controller 集成测试模式（MockMvc）

```java
@WebMvcTest(ProductController.class)
@AutoConfigureMockMvc
@DisplayName("商品控制器测试")
class ProductControllerTest {

  @Autowired
  private MockMvc mockMvc;

  @MockBean
  private ProductService productService;

  @Test
  @DisplayName("获取商品列表成功")
  void shouldGetProductListSuccessfully() throws Exception {
    // Arrange
    List<Product> products = Arrays.asList(
      new Product(1L, "商品1", new BigDecimal("100.00")),
      new Product(2L, "商品2", new BigDecimal("200.00"))
    );
    when(productService.list()).thenReturn(products);

    // Act & Assert
    mockMvc.perform(get("/api/products"))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$.code").value(200))
      .andExpect(jsonPath("$.data").isArray())
      .andExpect(jsonPath("$.data.length()").value(2))
      .andExpect(jsonPath("$.data[0].name").value("商品1"));
  }

  @Test
  @DisplayName("缺少必要参数时返回400")
  void shouldReturn400WhenMissingRequiredParam() throws Exception {
    mockMvc.perform(get("/api/products/search"))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.code").value(400));
  }

  @Test
  @DisplayName("创建商品成功")
  void shouldCreateProductSuccessfully() throws Exception {
    // Arrange
    CreateProductRequest request = new CreateProductRequest();
    request.setName("新商品");
    request.setPrice(new BigDecimal("150.00"));

    // Act & Assert
    mockMvc.perform(post("/api/products")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$.code").value(200));

    verify(productService).create(any(CreateProductRequest.class));
  }

  @Test
  @DisplayName("处理服务层异常")
  void shouldHandleServiceException() throws Exception {
    // Arrange
    when(productService.getById(999L))
      .thenThrow(new ProductNotFoundException("商品不存在"));

    // Act & Assert
    mockMvc.perform(get("/api/products/999"))
      .andExpect(status().isNotFound())
      .andExpect(jsonPath("$.code").value(404))
      .andExpect(jsonPath("$.message").value("商品不存在"));
  }
}
```

### Mapper 测试模式（使用内存数据库）

```java
@MybatisTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
@DisplayName("用户Mapper测试")
class UserMapperTest {

  @Autowired
  private UserMapper userMapper;

  @Test
  @DisplayName("插入用户成功")
  void shouldInsertUserSuccessfully() {
    // Arrange
    User user = new User();
    user.setUsername("testuser");
    user.setEmail("test@example.com");

    // Act
    int rows = userMapper.insert(user);

    // Assert
    assertEquals(1, rows);
    assertNotNull(user.getId());
  }

  @Test
  @DisplayName("根据用户名查询用户")
  void shouldSelectUserByUsername() {
    // Arrange
    User user = new User();
    user.setUsername("searchuser");
    userMapper.insert(user);

    // Act
    User result = userMapper.selectByUsername("searchuser");

    // Assert
    assertNotNull(result);
    assertEquals("searchuser", result.getUsername());
  }
}
```

### 参数化测试模式

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("订单金额计算测试")
class OrderAmountCalculatorTest {

  @ParameterizedTest
  @CsvSource({
    "100, 0, 100",       // 无折扣
    "100, 10, 90",       // 10%折扣
    "100, 100, 0",       // 100%折扣
    "0, 10, 0",          // 订单金额为0
  })
  @DisplayName("计算折扣后金额")
  void shouldCalculateDiscountedAmount(
      BigDecimal originalAmount,
      BigDecimal discountPercent,
      BigDecimal expectedAmount) {

    // Act
    BigDecimal result = calculator.calculate(originalAmount, discountPercent);

    // Assert
    assertEquals(expectedAmount, result);
  }

  @ParameterizedTest
  @ValueSource(strings = {"", "   ", "invalid"})
  @DisplayName("无效折扣率时抛出异常")
  void shouldThrowExceptionForInvalidDiscount(String invalidDiscount) {
    assertThrows(IllegalArgumentException.class, () -> {
      calculator.calculate(new BigDecimal("100"), new BigDecimal(invalidDiscount));
    });
  }
}
```

## 测试文件组织

```
doorslink-user/
src/
├── main/
│   └── java/net/doorslink/user/
│       ├── controller/
│       │   └── UserController.java
│       ├── service/
│       │   ├── UserService.java
│       │   └── impl/
│       │       └── UserServiceImpl.java
│       ├── mapper/
│       │   └── UserMapper.java
│       └── entity/
│           └── User.java
└── test/
    └── java/net/doorslink/user/
        ├── controller/
        │   └── UserControllerTest.java       # 集成测试
        ├── service/
        │   └── UserServiceTest.java          # 单元测试
        ├── mapper/
        │   └── UserMapperTest.java           # 数据库测试
        └── utils/
            └── PasswordUtilTest.java         # 工具类测试
```

## 模拟外部依赖

### Mapper 模拟

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

  @Mock
  private OrderMapper orderMapper;

  @Mock
  private UserMapper userMapper;

  @InjectMocks
  private OrderService orderService;

  @Test
  void shouldCalculateTotalPrice() {
    // 模拟Mapper返回数据
    when(orderMapper.selectByUserId(1L))
      .thenReturn(Arrays.asList(
        new Order(1L, new BigDecimal("100.00")),
        new Order(2L, new BigDecimal("200.00"))
      ));

    // 测试业务逻辑
    BigDecimal total = orderService.calculateTotalByUserId(1L);
    assertEquals(new BigDecimal("300.00"), total);
  }
}
```

### Redis 模拟

```java
@ExtendWith(MockitoExtension.class)
class CacheServiceTest {

  @Mock
  private StringRedisTemplate redisTemplate;

  @Mock
  private ValueOperations<String, String> valueOperations;

  @InjectMocks
  private CacheService cacheService;

  @BeforeEach
  void setup() {
    when(redisTemplate.opsForValue()).thenReturn(valueOperations);
  }

  @Test
  void shouldGetValueFromCache() {
    when(valueOperations.get("user:1")).thenReturn("cachedData");

    String result = cacheService.get("user:1");

    assertEquals("cachedData", result);
    verify(valueOperations).get("user:1");
  }
}
```

### HTTP 客户端模拟

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

  @Mock
  private RestTemplate restTemplate;

  @InjectMocks
  private PaymentService paymentService;

  @Test
  void shouldProcessPaymentSuccessfully() {
    // 模拟HTTP响应
    PaymentResponse mockResponse = new PaymentResponse();
    mockResponse.setSuccess(true);
    mockResponse.setTransactionId("TXN123456");

    when(restTemplate.postForObject(
      anyString(),
      any(PaymentRequest.class),
      eq(PaymentResponse.class)
    )).thenReturn(mockResponse);

    // 执行测试
    PaymentResult result = paymentService.process(new PaymentRequest());

    assertTrue(result.isSuccess());
    assertEquals("TXN123456", result.getTransactionId());
  }
}
```

## 测试覆盖率验证

### Maven JaCoCo 配置

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <executions>
    <execution>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>
    <execution>
      <id>check</id>
      <goals>
        <goal>check</goal>
      </goals>
      <configuration>
        <rules>
          <rule>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
              <limit>
                <counter>BRANCH</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### 运行覆盖率检查

```bash
# 运行测试并生成覆盖率报告
mvn clean test jacoco:report

# 查看HTML报告
open target/site/jacoco/index.html

# 运行覆盖率检查（会失败如果低于阈值）
mvn clean verify
```

## 应避免的常见测试错误

### ❌ 错误：测试实现细节

```java
// 不要测试内部状态
assertEquals(5, service.getInternalCounter());
```

### ✅ 正确：测试可观察的行为

```java
// 测试可观察的结果
List<Product> products = service.search("keyword");
assertEquals(5, products.size());
```

### ❌ 错误：不验证交互

```java
@Test
void testSaveUser() {
  userService.save(user);
  // 缺少验证
}
```

### ✅ 正确：验证关键交互

```java
@Test
void testSaveUser() {
  userService.save(user);
  verify(userMapper).insert(user);
}
```

### ❌ 错误：测试间有依赖

```java
@Test
void testCreateUser() {
  userService.create(new User("user1"));
}

@Test
void testUpdateUser() {
  // 依赖上面测试创建的用户
  userService.update("user1", newData);
}
```

### ✅ 正确：测试相互独立

```java
@Test
void testCreateUser() {
  User user = createTestUser();
  // 独立设置测试数据
}

@Test
void testUpdateUser() {
  User user = createTestUser(); // 每个测试独立创建数据
  userService.update(user.getId(), newData);
}
```

### ❌ 错误：使用真实的慢依赖

```java
@Test
void testWithRealDatabase() {
  // 连接到真实数据库，测试很慢
}
```

### ✅ 正确：使用模拟或内存数据库

```java
@MybatisTest
class UserMapperTest {
  // 使用H2内存数据库，测试快速且独立
}
```

## 持续测试

### 开发期间运行单个测试

```bash
# Maven - 运行单个测试类
mvn test -Dtest=UserServiceTest

# Maven - 运行单个测试方法
mvn test -Dtest=UserServiceTest#shouldGetUserById

# Gradle - 运行单个测试类
./gradlew test --tests UserServiceTest

# Gradle - 运行单个测试方法
./gradlew test --tests UserServiceTest.shouldGetUserById
```

### 预提交检查

```bash
# 运行所有测试
mvn clean test

# 运行测试和覆盖率检查
mvn clean verify
```

### CI/CD 集成

```yaml
# GitHub Actions
- name: Run Tests
  run: mvn clean test

- name: Generate Coverage Report
  run: mvn jacoco:report

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    files: target/site/jacoco/jacoco.xml
```

## 最佳实践

1. **先写测试** - 始终遵循TDD
2. **每个测试一个断言** - 专注于单一行为
3. **描述性的测试名称** - 使用@DisplayName说明测试目的
4. **Arrange-Act-Assert** - 清晰的测试结构
5. **模拟外部依赖** - 隔离单元测试
6. **测试边界情况** - Null、空字符串、最大值等
7. **测试异常路径** - 不仅仅是正常路径
8. **保持测试快速** - 单元测试每个 < 100ms
9. **测试后清理** - 使用@AfterEach清理数据
10. **审查覆盖率报告** - 识别未测试的代码

## 成功指标

* 达到 80%+ 代码覆盖率（行覆盖和分支覆盖）
* 所有测试通过（绿色）
* 没有跳过或禁用的测试（@Disabled）
* 快速测试执行（单元测试 < 60秒）
* Controller测试覆盖所有API端点
* 测试在生产前捕获bug

***

**记住**：测试不是可选的。它们是安全网，能够实现自信的重构、快速的开发和生产的可靠性。
