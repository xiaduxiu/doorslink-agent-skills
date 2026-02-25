---
name: java-best-practices
description: Java开发规范指南，包含命名约定、异常处理、Spring Boot最佳实践、代码组织等。编写Java代码、审查代码、重构时应使用此技能。
---

# Java 开发规范

## 命名约定

### 包名

- 全部小写，使用反向域名
- 格式：`com.company.module.submodule`
- 示例：`net.doorslink.user.service`

### 类名

- 大驼峰命名（UpperCamelCase）
- 名词，体现职责
- 示例：`UserService`, `OrderController`, `ProductMapper`

### 方法名

- 小驼峰命名（lowerCamelCase）
- 动词开头，描述行为
- 查询：`getXXX`, `findXXX`, `queryXXX`
- 创建：`createXXX`, `saveXXX`, `insertXXX`
- 更新：`updateXXX`, `modifyXXX`
- 删除：`deleteXXX`, `removeXXX`
- 检查：`isXXX`, `hasXXX`, `canXXX`
- 示例：`getUserById`, `createOrder`, `isValidEmail`

### 变量名

- 小驼峰命名
- 有意义，避免缩写（除非是公认缩写如id, url）
- 常量：全大写 + 下划线 `MAX_RETRY_COUNT`
- 布尔：使用`is`, `has`, `can`前缀
- 示例：`userList`, `orderTotalAmount`, `isActive`

### 数据库相关

- 表名：小写 + 下划线 `user_order`
- 字段名：小写 + 下划线 `created_at`
- 实体类名：对应表名转驼峰 `UserOrder`
- 实体属性：驼峰 `createdAt`

## 代码组织

### 包结构

```
net.doorslink.module/
├── config/          # 配置类
├── controller/      # REST控制器
│   └── request/     # 请求DTO
│   └── response/    # 响应DTO
├── service/         # 业务逻辑
│   └── impl/        # 实现类
├── mapper/          # MyBatis Mapper
├── entity/          # 数据库实体
├── dto/             # 数据传输对象
├── vo/              # 视图对象
├── util/            # 工具类
├── constant/        # 常量定义
├── enums/           # 枚举
└── exception/       # 自定义异常
```

### 类组织顺序

```java
public class UserService {
    // 1. 静态常量
    private static final int MAX_RETRY = 3;
    
    // 2. 实例常量
    private final UserMapper userMapper;
    
    // 3. 依赖注入（构造函数注入）
    @Autowired
    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }
    
    // 4. 公共方法
    public User getById(Long id) { }
    
    // 5. 私有方法
    private void validateUser(User user) { }
}
```

## Spring Boot 最佳实践

### 依赖注入

优先使用构造函数注入：

```java
@Service
public class OrderService {
    private final OrderMapper orderMapper;
    private final UserService userService;
    
    @Autowired
    public OrderService(OrderMapper orderMapper, UserService userService) {
        this.orderMapper = orderMapper;
        this.userService = userService;
    }
}
```

### Controller 层

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    @GetMapping("/{id}")
    public Result<UserResponse> getById(@PathVariable Long id) {
        User user = userService.getById(id);
        return Result.success(UserConverter.toResponse(user));
    }
    
    @PostMapping
    public Result<Void> create(@Valid @RequestBody CreateUserRequest request) {
        userService.create(request);
        return Result.success();
    }
}
```

### Service 层

```java
@Service
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {
    
    @Override
    public User getById(Long id) {
        if (id == null || id <= 0) {
            throw new IllegalArgumentException("用户ID无效");
        }
        return userMapper.selectById(id);
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void create(CreateUserRequest request) {
        // 业务校验
        validateCreateRequest(request);
        
        // 执行业务
        User user = convertToEntity(request);
        userMapper.insert(user);
        
        // 发送事件
        eventPublisher.publishEvent(new UserCreatedEvent(user));
    }
}
```

## 异常处理

### 统一异常体系

```java
// 基础业务异常
public class BusinessException extends RuntimeException {
    private final int code;
    
    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
    }
}

// 具体业务异常
public class UserNotFoundException extends BusinessException {
    public UserNotFoundException(Long userId) {
        super(404, "用户不存在: " + userId);
    }
}
```

### 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return Result.fail(400, message);
    }
    
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统繁忙，请稍后重试");
    }
}
```

## 日志规范

### 日志级别

- **ERROR**: 系统错误，需要立即处理
- **WARN**: 警告，不影响主流程但需要注意
- **INFO**: 业务信息，记录重要操作
- **DEBUG**: 调试信息，开发环境使用

### 日志格式

```java
@Slf4j
@Service
public class OrderService {
    
    public void createOrder(CreateOrderRequest request) {
        log.info("创建订单开始, userId={}, productId={}", 
            request.getUserId(), request.getProductId());
        
        try {
            // 业务逻辑
            log.debug("订单计算完成, amount={}", amount);
        } catch (Exception e) {
            log.error("创建订单失败, userId={}, request={}", 
                request.getUserId(), request, e);
            throw new OrderCreateException("创建订单失败");
        }
        
        log.info("创建订单完成, orderId={}", orderId);
    }
}
```

### 禁止事项

- ❌ 禁止在生产环境打印敏感信息（密码、token）
- ❌ 禁止使用 `System.out.println`
- ❌ 禁止在循环中大量打印日志
- ✅ 使用占位符 `{}` 而不是字符串拼接

## API 设计规范

### 统一响应格式

```java
@Data
public class Result<T> {
    private int code;
    private String message;
    private T data;
    private Long timestamp;
    
    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setCode(200);
        result.setMessage("success");
        result.setData(data);
        result.setTimestamp(System.currentTimeMillis());
        return result;
    }
    
    public static <T> Result<T> fail(int code, String message) {
        Result<T> result = new Result<>();
        result.setCode(code);
        result.setMessage(message);
        result.setTimestamp(System.currentTimeMillis());
        return result;
    }
}
```

### RESTful API 设计

| 操作 | HTTP方法 | URL | 说明 |
|------|----------|-----|------|
| 查询列表 | GET | `/api/users` | 支持分页参数 |
| 查询详情 | GET | `/api/users/{id}` | |
| 创建 | POST | `/api/users` | |
| 更新 | PUT | `/api/users/{id}` | 全量更新 |
| 部分更新 | PATCH | `/api/users/{id}` | 部分字段更新 |
| 删除 | DELETE | `/api/users/{id}` | |

### 分页参数

```java
@Data
public class PageRequest {
    @Min(1)
    private int pageNum = 1;
    
    @Min(1)
    @Max(100)
    private int pageSize = 20;
}

@Data
public class PageResult<T> {
    private List<T> list;
    private long total;
    private int pageNum;
    private int pageSize;
}
```

## 数据库规范

### MyBatis 最佳实践

```java
@Mapper
public interface UserMapper {
    
    @Select("SELECT * FROM user WHERE id = #{id}")
    User selectById(Long id);
    
    @Insert("INSERT INTO user (username, email) VALUES (#{username}, #{email})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insert(User user);
    
    @Update("UPDATE user SET username = #{username} WHERE id = #{id}")
    int update(User user);
    
    @Delete("DELETE FROM user WHERE id = #{id}")
    int deleteById(Long id);
}
```

### 复杂查询使用 XML

```xml
<!-- UserMapper.xml -->
<mapper namespace="net.doorslink.user.mapper.UserMapper">
    <select id="searchByCondition" resultType="User">
        SELECT * FROM user
        <where>
            <if test="username != null and username != ''">
                AND username LIKE CONCAT('%', #{username}, '%')
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
        </where>
        ORDER BY created_at DESC
    </select>
</mapper>
```

## 常用工具类

### 日期时间

```java
// 使用 LocalDateTime
LocalDateTime now = LocalDateTime.now();
LocalDateTime yesterday = now.minusDays(1);

// 格式化
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String formatted = now.format(formatter);

// 转换为时间戳
long timestamp = now.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli();
```

### 空值处理

```java
// 优先使用 Optional
public User getUserById(Long id) {
    return Optional.ofNullable(userMapper.selectById(id))
        .orElseThrow(() -> new UserNotFoundException(id));
}

// 集合空值处理
List<User> users = Optional.ofNullable(userList).orElse(Collections.emptyList());

// Stream 空安全
List<String> names = Optional.ofNullable(userList)
    .orElse(Collections.emptyList())
    .stream()
    .map(User::getName)
    .collect(Collectors.toList());
```

### 字符串处理

```java
// 判空（使用 Apache Commons Lang3 或 Spring 的 StringUtils）
StringUtils.isBlank(str);    // null, "", " " 都返回 true
StringUtils.isNotBlank(str); // 非空判断

// 默认值
String result = StringUtils.defaultIfBlank(str, "default");
```

## 代码审查清单

- [ ] 命名是否清晰、有意义
- [ ] 方法是否单一职责（不超过50行）
- [ ] 是否有适当的异常处理
- [ ] 是否处理了边界情况（null、空集合）
- [ ] 是否有适当的日志记录
- [ ] 是否避免了魔法数字
- [ ] 是否使用了合适的集合类型
- [ ] 是否避免了重复代码
- [ ] 是否有必要的注释（复杂逻辑）
- [ ] 是否遵循了项目编码规范
