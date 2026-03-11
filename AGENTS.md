# AGENTS.md - Agent Coding Guidelines

This document provides guidelines for agents working on this codebase.

## Project Overview

This is a Java-based Online Judge (OJ) system consisting of:
- **yuoj-backend-microservice-master**: Spring Cloud microservices backend (user, question, judge services)
- **yuoj-code-sandbox-master**: Code execution sandbox using Docker

## Build & Test Commands

### Backend Microservices

```bash
# Build all modules
cd yuoj-backend-microservice-master
mvn clean package -DskipTests

# Build specific module
cd yuoj-backend-microservice-master/yuoj-backend-user-service
mvn clean package

# Run all tests
cd yuoj-backend-microservice-master
mvn test

# Run single test class
cd yuoj-backend-microservice-master/yuoj-backend-user-service
mvn test -Dtest=YuojBackendUserServiceApplicationTests

# Run single test method
mvn test -Dtest=YuojBackendUserServiceApplicationTests#contextLoads

# Skip tests (for building)
mvn clean package -DskipTests

# Run Spring Boot application
cd yuoj-backend-microservice-master/yuoj-backend-user-service
mvn spring-boot:run
```

### Code Sandbox

```bash
cd yuoj-code-sandbox-master
mvn clean package -DskipTests
mvn test
mvn spring-boot:run
```

## Code Style Guidelines

### General Conventions

- **Language**: Java 1.8
- **Framework**: Spring Boot 2.6.x/2.7.x
- **Build Tool**: Maven
- **Documentation**: Chinese comments (see existing code patterns)

### Project Structure

```
src/
├── main/java/com/yupi/
│   ├── [modulename]/
│   │   ├── controller/     # REST endpoints
│   │   ├── service/        # Business logic (interface + impl/)
│   │   ├── mapper/         # MyBatis mappers
│   │   ├── model/         # DTOs, entities, enums, VO
│   │   └── config/        # Configuration classes
│   └── common/            # Shared utilities, exceptions
├── main/resources/        # Application configs
└── test/                  # Test classes
```

### Naming Conventions

- **Classes**: PascalCase (e.g., `UserController`, `UserServiceImpl`)
- **Methods**: camelCase (e.g., `getUserById`, `userLogin`)
- **Variables**: camelCase (e.g., `userAccount`, `userPassword`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `USER_LOGIN_STATE`, `ADMIN_ROLE`)
- **Packages**: lowercase (e.g., `com.yupi.yuojbackenduserservice`)

### Class Organization

1. Package declaration
2. Import statements (grouped: java, org, com, javax)
3. Class-level annotations
4. Class documentation comment
5. Class declaration
6. Constants (static final)
7. Member variables
8. Constructors
9. Public methods
10. Private methods

### Response Patterns

Use `BaseResponse<T>` for all API responses:

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @GetMapping("/get")
    public BaseResponse<User> getUserById(long id, HttpServletRequest request) {
        if (id <= 0) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR);
        }
        User user = userService.getById(id);
        return ResultUtils.success(user);
    }
}
```

### Error Handling

- Use `BusinessException` for business logic errors
- Use `ErrorCode` enum for standardized error codes
- Use `ThrowUtils.throwIf()` for conditional throwing
- Always validate input parameters at controller/service layer

```java
// Throw with error code
throw new BusinessException(ErrorCode.PARAMS_ERROR);

// Throw with custom message
throw new BusinessException(ErrorCode.PARAMS_ERROR, "用户账号不能为空");

// Conditional throw
ThrowUtils.throwIf(user == null, ErrorCode.NOT_FOUND_ERROR);
ThrowUtils.throwIf(!result, ErrorCode.OPERATION_ERROR);
```

### Entity & DTO Patterns

**Entity (Database model)**:
```java
@TableName(value = "user")
@Data
public class User implements Serializable {
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    private String userAccount;
    private Date createTime;
    private Date updateTime;
    @TableLogic
    private Integer isDelete;
}
```

**Request DTO**:
```java
@Data
public class UserAddRequest implements Serializable {
    private String userName;
    private String userAccount;
    private String userAvatar;
    private String userRole;
    private static final long serialVersionUID = 1L;
}
```

**VO (Response)**:
```java
@Data
public class UserVO implements Serializable {
    private Long id;
    private String userName;
    private String userAvatar;
    private String userProfile;
}
```

### Service Layer

- Always use interface + implementation pattern
- Use MyBatis-Plus `ServiceImpl` base class
- Validate parameters at service entry
- Use `synchronized` for critical sections (e.g., user registration)

```java
@Service
@Slf4j
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    
    private static final String SALT = "yupi";
    
    @Override
    public long userRegister(String userAccount, String userPassword, String checkPassword) {
        if (StringUtils.isAnyBlank(userAccount, userPassword, checkPassword)) {
            throw new BusinessException(ErrorCode.PARAMS_ERROR, "参数为空");
        }
        // ...
    }
}
```

### Controller Layer

- Use `@RestController` and `@RequestMapping`
- Use `@Resource` (not `@Autowired`) for injection
- Add `@AuthCheck` for admin-only endpoints
- Use region comments (`// region`, `// endregion`) for grouping

```java
@RestController
@RequestMapping("/")
@Slf4j
public class UserController {

    @Resource
    private UserService userService;

    // region 登录相关

    @PostMapping("/register")
    public BaseResponse<Long> userRegister(@RequestBody UserRegisterRequest request) {
        // ...
    }

    // endregion
}
```

### Database Queries

- Use MyBatis-Plus QueryWrapper for dynamic queries
- Always validate sort fields with `SqlUtils.validSortField()`
- Use `StringUtils.isNotBlank()` for null/empty checks

```java
public QueryWrapper<User> getQueryWrapper(UserQueryRequest request) {
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq(id != null, "id", id);
    queryWrapper.like(StringUtils.isNotBlank(userName), "userName", userName);
    queryWrapper.orderBy(SqlUtils.validSortField(sortField), 
        sortOrder.equals(CommonConstant.SORT_ORDER_ASC), sortField);
    return queryWrapper;
}
```

### Required Annotations

- `@Data` - Lombok for getters/setters
- `@TableName` - MyBatis-Plus entity mapping
- `@TableId` - Primary key configuration
- `@TableLogic` - Soft delete
- `@TableField(exist = false)` - Non-database fields
- `@Slf4j` - Logging
- `@RestController` - REST API
- `@RequestMapping` - URL mapping
- `@AuthCheck` - Role-based access control

### Import Order

1. `java.*`
2. `javax.*`
3. `org.*` (external)
4. `com.*` (project)
5. Static imports last

### Testing

- Place tests in `src/test/java` matching main package structure
- Use `@SpringBootTest` for integration tests
- Test class naming: `[ClassName]Tests`

```java
@SpringBootTest
class YuojBackendUserServiceApplicationTests {

    @Test
    void contextLoads() {
    }
}
```

### Best Practices

1. **Never hardcode secrets** - Use environment variables or config files
2. **Always validate input** - Check null, empty, length constraints
3. **Use appropriate HTTP status codes** - 200 for success, 4xx for client errors
4. **Log important operations** - Use `@Slf4j` logger
5. **Use DTOs for requests** - Don't expose entities directly
6. **Use VOs for responses** - Hide internal fields from clients
7. **Use enums for constants** - Don't use magic strings/numbers
8. **Implement soft delete** - Use `@TableLogic` for isDelete field
9. **Handle exceptions globally** - Use `GlobalExceptionHandler`

### Database Conventions

- Table names: snake_case (e.g., `user`, `question_submit`)
- Column names: snake_case (e.g., `user_account`, `create_time`)
- Use `IdType.ASSIGN_ID` for distributed ID generation
- Always include `create_time`, `update_time`, `is_delete` columns
