---
name: java-tdd-guide
description: Java测试驱动开发完整指南。使用JUnit 5、Mockito、MockMvc、Testcontainers进行单元测试、集成测试。覆盖率要求80%+。
---

# Java 测试驱动开发工作流

适用于 Java Spring Boot + MyBatis + MySQL 项目的 TDD 指南。

## 何时激活

- 编写新功能或接口
- 修复错误或问题
- 重构现有代码
- 添加 API 端点
- 创建新服务或 Mapper

## 核心原则

### 1. 测试优先于代码

始终先编写测试，然后实现代码使测试通过。

### 2. 覆盖率要求

- 最低 80% 覆盖率（单元 + 集成）
- 覆盖所有边缘情况
- 测试异常场景
- 验证边界条件

### 3. 测试类型

#### 单元测试

- 单个 Service 方法
- 业务逻辑
- 工具类
- VO/DTO 转换

#### 集成测试

- Controller API
- Mapper 数据库操作
- 服务间调用

## TDD 工作流步骤

### 步骤 1: 编写用户故事

```
作为 [角色], 我想要 [功能], 以便于 [收益]

示例:
作为管理员，我想要创建市场，
以便于用户可以参与预测市场。
```

### 步骤 2: 生成测试用例

针对每个用户故事，创建全面的测试用例：

```java
class MarketServiceTest {
  
  @Test
  void shouldCreateMarketSuccessfully() {
    // 测试正常创建
  }

  @Test
  void shouldThrowExceptionWhenNameIsBlank() {
    // 测试边界条件 - 空名称
  }

  @Test
  void shouldThrowExceptionWhenEndDateIsPast() {
    // 测试业务规则 - 结束日期不能是过去
  }

  @Test
  void shouldReturnMarketById() {
    // 测试查询功能
  }

  @Test
  void shouldThrowExceptionWhenMarketNotFound() {
    // 测试异常场景
  }
}
```

### 步骤 3: 运行测试（应该失败）

```bash
./mvnw test -Dtest=MarketServiceTest
# Tests should fail - we haven't implemented yet
```

### 步骤 4: 实现代码

编写最少的代码以使测试通过：

```java
@Service
public class MarketService {
  
  public Market create(CreateMarketRequest request) {
    if (request.name() == null || request.name().isBlank()) {
      throw new IllegalArgumentException("名称不能为空");
    }
    // Implementation here
  }
}
```

### 步骤 5: 再次运行测试

```bash
./mvnw test
# Tests should now pass
```

### 步骤 6: 重构

在保持测试通过的同时提高代码质量：

- 消除重复
- 改进命名
- 优化性能

### 步骤 7: 验证覆盖率

```bash
./mvnw test jacoco:report
# Verify 80%+ coverage achieved
```

## 测试模式

### 单元测试模式 (JUnit 5 + Mockito)

```java
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {
  
  @Mock
  private MarketMapper marketMapper;
  
  @InjectMocks
  private MarketService marketService;

  @Test
  void shouldCreateMarketSuccessfully() {
    // Arrange
    CreateMarketRequest request = new CreateMarketRequest(
      "Test Market",
      "Description",
      Instant.now().plusSeconds(86400),
      List.of("category")
    );
    
    when(marketMapper.insert(any(Market.class)))
        .thenReturn(1);

    // Act
    Market result = marketService.create(request);

    // Assert
    assertThat(result).isNotNull();
    assertThat(result.getName()).isEqualTo("Test Market");
    verify(marketMapper).insert(any(Market.class));
  }

  @Test
  void shouldThrowExceptionWhenNameIsBlank() {
    // Arrange
    CreateMarketRequest request = new CreateMarketRequest(
      "",
      "Description",
      Instant.now().plusSeconds(86400),
      List.of()
    );

    // Act & Assert
    assertThatThrownBy(() -> marketService.create(request))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("名称不能为空");
  }

  @ParameterizedTest
  @CsvSource({
    "active, true",
    "closed, false",
    "pending, true"
  })
  void shouldCheckMarketStatus(String status, boolean expectedResult) {
    Market market = new Market();
    market.setStatus(status);
    
    boolean result = marketService.isActive(market);
    
    assertThat(result).isEqualTo(expectedResult);
  }
}
```

### Controller 测试模式 (MockMvc)

```java
@WebMvcTest(MarketController.class)
class MarketControllerTest {
  
  @Autowired
  private MockMvc mockMvc;
  
  @MockBean
  private MarketService marketService;

  @Test
  void shouldReturnMarketsList() throws Exception {
    Page<Market> page = new PageImpl<>(List.of(
        createMarket(1L, "Market 1"),
        createMarket(2L, "Market 2")
    ));
    
    when(marketService.list(any(Pageable.class))).thenReturn(page);

    mockMvc.perform(get("/api/markets")
            .param("page", "0")
            .param("size", "20"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.content").isArray())
        .andExpect(jsonPath("$.content.length()").value(2))
        .andExpect(jsonPath("$.content[0].name").value("Market 1"));
  }

  @Test
  void shouldReturn400WhenRequestIsInvalid() throws Exception {
    when(marketService.list(any(Pageable.class)))
        .thenThrow(new IllegalArgumentException("Invalid param"));

    mockMvc.perform(get("/api/markets")
            .param("page", "-1"))
        .andExpect(status().isBadRequest());
  }

  @Test
  void shouldCreateMarket() throws Exception {
    when(marketService.create(any()))
        .thenAnswer(inv -> {
          Market m = inv.getArgument(0);
          m.setId(1L);
          return m;
        });

    mockMvc.perform(post("/api/markets")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {
                  "name": "Test Market",
                  "description": "Description",
                  "endDate": "2025-12-31T00:00:00Z",
                  "categories": ["tech"]
                }
                """))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.id").value(1))
        .andExpect(header().string("Location", "/api/markets/1"));
  }
}
```

### 集成测试模式 (SpringBootTest)

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class MarketIntegrationTest {
  
  @Autowired
  private MockMvc mockMvc;
  
  @Autowired
  private MarketMapper marketMapper;

  @Test
  void shouldCreateAndRetrieveMarket() throws Exception {
    // Create market via API
    mockMvc.perform(post("/api/markets")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {
                  "name": "Integration Test Market",
                  "description": "Testing",
                  "endDate": "2025-12-31T00:00:00Z",
                  "categories": ["test"]
                }
                """))
        .andExpect(status().isCreated());

    // Verify from database
    List<Market> markets = marketMapper.selectList(null);
    assertThat(markets).hasSize(1);
    assertThat(markets.get(0).getName()).isEqualTo("Integration Test Market");
  }

  @Test
  void shouldReturn404WhenMarketNotFound() throws Exception {
    mockMvc.perform(get("/api/markets/99999"))
        .andExpect(status().isNotFound());
  }
}
```

### Mapper 测试 (使用 H2 或 Testcontainers)

```java
@MapperTest
class MarketMapperTest {
  
  @Autowired
  private MarketMapper marketMapper;

  @Test
  void shouldInsertAndSelectMarket() {
    Market market = new Market();
    market.setName("Mapper Test");
    market.setDescription("Test Description");
    market.setStatus("active");
    market.setCreatedAt(Instant.now());

    marketMapper.insert(market);

    Market found = marketMapper.selectById(market.getId());
    assertThat(found).isNotNull();
    assertThat(found.getName()).isEqualTo("Mapper Test");
  }

  @Test
  void shouldUpdateMarket() {
    Market market = createTestMarket();
    marketMapper.insert(market);

    market.setName("Updated Name");
    marketMapper.updateById(market);

    Market found = marketMapper.selectById(market.getId());
    assertThat(found.getName()).isEqualTo("Updated Name");
  }

  private Market createTestMarket() {
    Market market = new Market();
    market.setName("Test");
    market.setStatus("active");
    market.setCreatedAt(Instant.now());
    return market;
  }
}
```

## 测试文件组织

```
doorslink-db/
├── src/
│   └── test/
│       └── java/
│           └── net/doorslink/
│               └── mapper/
│                   └── MarketMapperTest.java

doorslink-admin/
├── src/
│   ├── main/
│   │   └── java/...
│   └── test/
│       └── java/
│           └── net/doorslink/
│               ├── controller/
│               │   └── MarketControllerTest.java
│               ├── service/
│               │   └── MarketServiceTest.java
│               └── integration/
│                   └── MarketIntegrationTest.java
```

## 模拟外部服务

### Mapper 模拟

```java
@ExtendWith(MockitoExtension.class)
class MarketServiceTest {
  
  @Mock
  private MarketMapper marketMapper;

  @Test
  void shouldHandleDatabaseError() {
    when(marketMapper.selectList(any()))
        .thenThrow(new DataAccessException("DB error") {});

    assertThatThrownBy(() -> marketService.listAll())
        .isInstanceOf(ServiceException.class)
        .hasMessageContaining("数据库错误");
  }
}
```

### 事务模拟

```java
@Test
void shouldRollbackOnException() {
  when(marketMapper.insert(any(Market.class)))
      .thenThrow(new RuntimeException("DB error"));

  assertThatThrownBy(() -> marketService.createWithRollback(request))
      .isInstanceOf(ServiceException.class);
  
  verify(marketMapper, never()).commit();
}
```

## 测试覆盖率验证

### 运行覆盖率报告

```bash
./mvnw test jacoco:report
```

### 查看报告

```bash
open target/site/jacoco/index.html
```

### Maven 配置 (pom.xml)

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.11</version>
  <configuration>
    <rules>
      <rule>
        <element>BUNDLE</element>
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
</plugin>
```

## 应避免的常见测试错误

### ❌ 错误：测试实现细节

```java
// Don't test internal state
assertThat(service.getCache()).containsKey("key");
```

### ✅ 正确：测试外部行为

```java
// Test what users see
assertThat(result).isEqualTo(expectedMarket);
```

### ❌ 错误：没有测试隔离

```java
// Tests depend on each other
@Test
void test1() { market = createMarket(); }
@Test
void test2() { market.setName("Updated"); } // Depends on test1
```

### ✅ 正确的：独立的测试

```java
// Each test sets up its own data
@Test
void shouldCreateMarket() {
  Market market = createTestMarket();
  // Test logic
}

@Test
void shouldUpdateMarket() {
  Market market = createTestMarket();
  // Test update logic
}
```

### ❌ 错误：硬编码数据

```java
// Fragile test
assertThat(market.getId()).isEqualTo(1L);
```

### ✅ 正确：动态验证

```java
// Resilient test
assertThat(market.getId()).isGreaterThan(0);
```

## 常用测试工具

### 测试数据构建器

```java
class MarketTestBuilder {
  private Market market = new Market();
  
  public MarketTestBuilder name(String name) {
    market.setName(name);
    return this;
  }
  
  public MarketTestBuilder status(String status) {
    market.setStatus(status);
    return this;
  }
  
  public Market build() {
    return market;
  }
}

// 使用
Market market = new MarketTestBuilder()
    .name("Test Market")
    .status("active")
    .build();
```

### JSON 序列化测试

```java
@Test
void shouldSerializeToJson() throws JsonProcessingException {
  Market market = createMarket(1L, "Test");
  
  String json = objectMapper.writeValueAsString(market);
  
  assertThat(json).contains("\"name\":\"Test\"");
}
```

## 持续测试

### 开发期间的监视模式

```bash
./mvnw test -Dwatch
# Tests run automatically on file changes (if using dev tools)
```

### 预提交检查

```bash
./mvnw test verify
# Runs tests before verify phase
```

### CI/CD 集成 (GitHub Actions)

```yaml
- name: Run Tests
  run: ./mvnw test verify

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./target/site/jacoco/jacoco.xml
```

## 最佳实践

1. **先写测试** - 始终遵循 TDD
2. **每个测试一个断言** - 专注于单一行为
3. **描述性的测试名称** - `shouldCreateMarketWhenNameIsValid`
4. **Arrange-Act-Assert** - 清晰的测试结构
5. **模拟外部依赖** - 隔离单元测试
6. **测试边缘情况** - Null、空字符串、边界值
7. **测试异常路径** - 不仅仅是正常路径
8. **保持测试快速** - 单元测试 < 100ms
9. **测试后清理** - 无副作用
10. **审查覆盖率报告** - 识别空白

## 成功指标

- 达到 80%+ 代码覆盖率
- 所有测试通过
- 没有跳过的测试
- 快速测试执行（单元测试 < 60秒）
- 集成测试覆盖关键 API
- 测试在生产前捕获错误

## 项目测试配置参考

### 依赖 (pom.xml)

```xml
<dependencies>
  <!-- Test dependencies -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  
  <!-- H2 for mapper tests -->
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
  </dependency>
  
  <!-- JaCoCo for coverage -->
  <dependency>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

### 测试配置 (application-test.yml)

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  mybatis:
    mapper-locations: classpath:mapper/*.xml
```

***

**记住**：测试不是可选的。它们是安全网，能够实现自信的重构、快速的开发和生产的可靠性。
