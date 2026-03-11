# OJ在线判题系统 - 面试复习文档

## 架构设计

### Spring Cloud微服务架构

本项目采用Spring Cloud微服务架构，包含以下核心组件：

- **Nacos**: 作为服务注册中心和服务配置中心，所有服务启动时向Nacos注册服务，消费者通过Nacos发现服务
- **Spring Cloud Gateway**: 作为统一网关，端口8101，负责路由转发、统一鉴权、跨域配置、负载均衡
- **Spring Cloud OpenFeign**: 实现服务间远程调用，题目服务调用用户服务获取用户信息、调用判题服务获取判题结果

#### 服务划分

| 服务 | 端口 | 主要职责 |
|------|------|----------|
| yuoj-backend-gateway | 8101 | 路由、鉴权、负载均衡 |
| yuoj-backend-user-service | 8102 | 用户管理、权限校验、Session管理 |
| yuoj-backend-question-service | 8103 | 题目CRUD、提交代码、调用判题服务 |
| yuoj-backend-judge-service | 8104 | 判题核心、调用代码沙箱、结果校验 |

#### 网关路由配置

```yaml
/api/user/**     -> lb://yuoj-backend-user-service
/api/question/** -> lb://yuoj-backend-question-service  
/api/judge/**    -> lb://yuoj-backend-judge-service
```

### 代码沙箱设计

代码沙箱是OJ系统的核心，负责安全地执行用户提交的代码。

#### 架构实现

采用模板方法模式+工厂模式+代理模式：

- **CodeSandbox接口**: 定义执行代码的抽象接口
- **JavaCodeSandboxTemplate**: 模板类，定义执行流程（校验→编译→运行→收集结果）
- **JavaDockerCodeSandbox**: Docker容器化实现（核心），使用Docker Java Client调用Docker API
- **JavaNativeCodeSandbox**: 本地执行实现（不安全，仅供测试）
- **CodeSandboxFactory**: 工厂类，根据类型创建沙箱实例
- **CodeSandboxProxy**: 代理类，增强功能（参数校验、错误处理、日志记录）

#### Docker安全策略

```java
hostConfig.withMemory(100 * 1000 * 1000L);  // 内存限制 100MB
hostConfig.withMemorySwap(0L);               // 禁用Swap
hostConfig.withCpuCount(1L);                // CPU限制1核
hostConfig.withNetworkDisabled(true);       // 禁用网络
hostConfig.withReadonlyRootfs(true);         // 只读文件系统
```

#### 执行流程

1. 接收代码和输入用例
2. 创建Docker容器（设置资源限制）
3. 复制必要文件到容器
4. 编译代码（Java: javac）
5. 执行程序，获取输出
6. 收集执行结果（标准输出、错误信息、退出码）
7. 清理容器

### 判题服务设计

#### 判题流程

1. 题目服务创建提交记录，状态设为"待判题"（0）
2. 题目服务发送消息到RabbitMQ
3. 判题服务消费消息，调用代码沙箱执行代码
4. 代码沙箱在Docker容器中运行代码，返回结果
5. 判题服务根据策略校验结果，更新数据库状态
6. 用户查询提交结果

#### 策略模式设计

- **JudgeStrategy接口**: 定义判题的抽象方法
- **DefaultJudgeStrategy**: 默认判题策略，适用于C、C++、Python等语言
- **JavaLanguageJudgeStrategy**: Java语言特殊判题策略（执行类名需为Main）

判题内容：
- 比对输出和预期输出是否完全一致
- 检查时间限制（Time Limit Exceeded）
- 检查内存限制（Memory Limit Exceeded）
- 运行时错误（Runtime Error）

#### 消息队列

使用RabbitMQ实现异步判题：

- **队列名称**: code_queue
- **消息格式**: 包含提交ID、代码、语言、输入用例
- **生产者**: QuestionService
- **消费者**: JudgeService

## 数据库设计

### 表结构

#### user表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键（雪花算法ID） |
| userAccount | varchar | 用户账号 |
| userPassword | varchar | 密码（MD5+盐加密） |
| userName | varchar | 用户昵称 |
| userAvatar | varchar | 头像URL |
| userRole | varchar | 角色（user/admin/ban） |
| createTime | datetime | 创建时间 |
| updateTime | datetime | 更新时间 |
| isDelete | tinyint | 逻辑删除（0未删/1已删） |

#### question表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| title | varchar | 题目标题 |
| content | text | 题目内容（HTML/Markdown） |
| tags | json | 标签（简单/中等/困难） |
| answer | text | 题目答案 |
| submitNum | int | 提交数 |
| acceptedNum | int | 通过数 |
| judgeCase | json | 判题用例（输入+输出） |
| judgeConfig | json | 判题配置（时间限制/内存限制） |
| thumbNum | int | 点赞数 |
| favourNum | int | 收藏数 |
| userId | bigint | 创建用户ID |

#### question_submit表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| language | varchar | 编程语言 |
| code | longtext | 提交代码 |
| judgeInfo | json | 判题信息（状态/时间/内存/错误信息） |
| status | int | 状态（0待判题/1判题中/2成功/3失败） |
| questionId | bigint | 题目ID |
| userId | bigint | 用户ID |

### 雪花算法

使用MyBatis-Plus的雪花算法生成ID：

- 64位：1位符号位 + 41位时间戳 + 5位数据中心 + 5位机器码 + 12位自增序列
- 使用 `@TableId(type = IdType.ASSIGN_ID)` 注解

### 软删除

使用MyBatis-Plus的逻辑删除：

- 使用 `@TableLogic` 注解标记isDelete字段
- 查询时自动拼接 `AND isDelete = 0`
- 删除时自动变为UPDATE操作

## 业务功能

### 用户服务

#### 注册流程

1. 校验参数（账号、密码、确认密码不能为空）
2. 校验账号不重复
3. 密码MD5+盐加密
4. 插入数据库
5. 返回新用户ID

#### 登录流程

1. 校验参数
2. 密码MD5+盐加密
3. 查询数据库匹配
4. 生成Token（使用Redis存储Session）
5. 返回用户信息

#### 权限控制

- 使用自定义注解 `@AuthCheck` 实现角色校验
- 管理员可删除题目、修改题目
- 普通用户只能查看、提交

### 题目服务

#### 题目管理

- CRUD操作
- 分页查询（根据标签、难度、搜索关键词）
- 点赞、收藏功能

#### 题目提交

1. 接收用户提交的代码
2. 创建提交记录（状态=待判题）
3. 发送消息到RabbitMQ
4. 返回提交ID
5. 用户轮询查询结果

### 判题服务

#### 判题状态机

```
待判题(0) -> 判题中(1) -> 成功(2)
                      -> 失败(3)
```

#### 判题信息结构

```json
{
  "status": 2,
  "message": "Accept",
  "time": 100,
  "memory": 2048
}
```

## 技术亮点

### 1. 服务注册与发现

- 使用Nacos作为注册中心
- 服务启动时自动注册
- 消费者从Nacos获取服务列表
- 支持负载均衡

### 2. 服务间通信

- 使用Feign Client进行RPC调用
- 题目服务调用用户服务获取用户信息（验证用户是否登录）
- 内部接口使用 `/inner` 路径标识，通过网关鉴权拦截

### 3. 异步判题

- 使用RabbitMQ实现异步处理
- 避免用户长时间等待
- 支持高并发提交

### 4. 代码沙箱安全

- Docker容器隔离
- 资源限制（CPU、内存、网络）
- 禁止危险操作（文件系统访问、网络请求）

### 5. 策略模式

- 判题策略解耦
- 便于扩展支持新语言

## 常见面试问题

### Q1: 项目整体架构是怎么样的？

采用Spring Cloud微服务架构，包含用户服务、题目服务、判题服务、网关服务，使用Nacos做服务注册发现，Gateway做路由转发，Feign做服务间调用，RabbitMQ做异步判题，Docker做代码沙箱。

### Q2: 代码沙箱如何保证安全性？

使用Docker容器进行隔离，通过设置资源限制（内存100MB、CPU 1核）防止资源耗尽，禁用网络防止外泄数据，禁用文件系统写入防止恶意操作。

### Q3: 判题流程是什么？

用户提交代码 -> 题目服务创建提交记录 -> 发送到RabbitMQ -> 判题服务消费消息 -> 调用代码沙箱执行 -> 策略模式校验结果 -> 更新数据库 -> 用户查询结果。

### Q4: 如何处理高并发提交？

使用RabbitMQ异步处理，将判题任务放入队列，判题服务按顺序消费，避免同步调用导致超时。

### Q5: 如何保证接口安全？

网关统一鉴权，内部接口用 `/inner` 路径标识并拦截，用户接口需要登录态（Redis Session），敏感操作需要管理员权限。

### Q6: 数据库设计有哪些优化？

使用逻辑删除（@TableLogic）、雪花算法分布式ID、JSON字段存储动态数据（tags、judgeCase）、索引优化（questionId、userId、status）。

### Q7: 为什么使用消息队列？

解耦题目服务和判题服务，异步处理避免用户等待，支持高并发削峰，提高系统吞吐量。

### Q8: Feign的工作原理？

基于动态代理，注解声明式HTTP客户端，在运行时会生成代理对象，通过Ribbon做负载均衡，通过HTTP Client发起请求。

### Q9: 如何保证代码沙箱的资源不被耗尽？

Docker容器限制内存100MB、禁用Swap、限制CPU 1核、禁用网络、只读文件系统，超出限制则kill进程。

### Q10: 判题结果如何比对？

使用精确比对（输出完全一致），同时检查时间是否超时、内存是否超限、是否有运行时错误。
