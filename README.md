# 🚀 OJ Online Judge 在线判题系统

<p align="center">
  <img src="https://img.shields.io/badge/Spring%20Boot-2.7+-green" alt="Spring Boot">
  <img src="https://img.shields.io/badge/Vue-3.x-blue" alt="Vue">
  <img src="https://img.shields.io/badge/Microservices-Cloud-orange" alt="Microservices">
  <img src="https://img.shields.io/badge/Docker-Container-red" alt="Docker">
</p>

> 基于微服务架构的在线编程评测系统，支持多种编程语言提交与自动判题

---

## 📋 目录

- [项目概述](#-项目概述)
- [系统架构](#-系统架构)
- [服务清单](#-服务清单)
- [数据流向](#-数据流向)
- [核心流程](#-核心流程)
- [技术栈](#-技术栈)
- [快速开始](#-快速开始)

---

## 🎯 项目概述

本项目是一个完整的 **Online Judge (OJ)** 系统，采用微服务架构设计，包含：

- **前端层**: Vue3 用户界面
- **网关层**: Spring Cloud Gateway 统一入口
- **业务层**: 用户服务、题目服务、判题服务
- **基础设施层**: 代码沙箱、消息队列、数据库、缓存

**核心功能**: 用户提交代码 → 消息队列 → 判题服务 → 代码沙箱执行 → 返回评测结果

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              前端层 (Vue3)                               │
│                    ┌─────────────────────────────────┐                  │
│                    │      用户界面 / 题目列表         │                  │
│                    │      代码编辑器 / 提交记录       │                  │
│                    └──────────────┬──────────────────┘                  │
└───────────────────────────────────┼─────────────────────────────────────┘
                                    │ HTTP
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           网关层 (Gateway)                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Spring Cloud Gateway (Port: 8101)                                │  │
│  │  ├── 路由: /api/user/**    → User Service                         │  │
│  │  ├── 路由: /api/question/** → Question Service                    │  │
│  │  └── 路由: /api/judge/**   → Judge Service                        │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  User Service   │    │Question Service │    │  Judge Service  │
│    (8102)       │◄──►│    (8103)       │    │    (8104)       │
│                 │    │                 │    │                 │
│ • 用户注册/登录  │    │ • 题目CRUD      │    │ • 接收MQ消息    │
│ • 用户管理      │    │ • 代码提交      │───►│ • 调用代码沙箱  │
│ • 权限验证      │    │ • 提交记录      │ MQ │ • 判题策略      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
          │                                              │
          │         ┌───────────────────────┐            │
          │         │    Nacos (8848)       │            │
          │         │  服务注册与发现        │            │
          │         └───────────────────────┘            │
          │                         │                    │
          │    ┌────────────────────┴────────────────┐   │
          │    │                                    │   │
          │    ▼                                    ▼   │
          │ ┌──────────────────────────────────────────┐ │
          │ │      RabbitMQ MQ (5672 / 15672)          │ │
          │ │  ┌──────────────┐    ┌──────────────┐    │ │
          │ │  │code_exchange │───►│ code_queue   │    │ │
          │ │  └──────────────┘    └──────┬───────┘    │ │
          │ └──────────────────────────────┼────────────┘ │
          │                                │              │
          └────────────────────────────────┼──────────────┘
                                           │ HTTP + 鉴权
                                           ▼
                          ┌──────────────────────────────────┐
                          │    Code Sandbox 代码沙箱         │
                          │         (Port: 8090)             │
                          │                                  │
                          │  • Docker 容器隔离               │
                          │  • 执行用户代码                  │
                          │  • 返回执行结果                  │
                          └──────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                           数据层                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐      │
│  │   MySQL (3306)   │  │  Redis (6379)    │  │   Docker Host    │      │
│  │                  │  │                  │  │                  │      │
│  │ • user           │  │ • Session 存储   │  │ • 代码执行容器   │      │
│  │ • question       │  │ • 缓存           │  │ • 资源隔离       │      │
│  │ • question_submit│  │                  │  │ • 安全管理       │      │
│  │ • post           │  │                  │  │                  │      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 服务清单

| 服务名称 | 端口 | 技术栈 | 职责描述 |
|---------|------|--------|---------|
| **Gateway** | 8101 | Spring Cloud Gateway | API 网关、路由转发、统一鉴权 |
| **User Service** | 8102 | Spring Boot + MyBatis-Plus | 用户注册、登录、信息管理 |
| **Question Service** | 8103 | Spring Boot + MyBatis-Plus | 题目管理、代码提交、MQ 生产者 |
| **Judge Service** | 8104 | Spring Boot + RabbitMQ | 判题逻辑、MQ 消费者、策略模式 |
| **Code Sandbox** | 8090 | Spring Boot + Docker | 代码执行、沙箱隔离、安全管理 |
| **Nacos** | 8848 | Alibaba Nacos | 服务注册中心、配置管理 |
| **MySQL** | 3306 | MySQL 8.0 | 业务数据持久化 |
| **Redis** | 6379 | Redis | Session 存储、缓存 |
| **RabbitMQ** | 5672/15672 | RabbitMQ | 消息队列、异步解耦 |

---

## 🔄 数据流向

### 核心业务流程：提交代码 → 判题

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   用户    │────►│   提交代码    │────►│ QuestionService│     │   创建记录   │
│          │     │              │     │              │────►│ (status=0)   │
└──────────┘     └──────────────┘     └──────┬───────┘     └──────┬───────┘
                                              │                     │
                                              │ MQ Producer         │ MySQL
                                              ▼                     ▼
                                       ┌──────────────┐     ┌──────────────┐
                                       │code_exchange │────►│question_submit│
                                       └──────────────┘     └──────────────┘
                                              │
                                              │ routingKey: my_routingKey
                                              ▼
                                       ┌──────────────┐
                                       │  code_queue  │
                                       └──────┬───────┘
                                              │ MQ Consumer
                                              ▼
                                       ┌──────────────┐
                                       │ JudgeService │
                                       └──────┬───────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │获取题目信息   │        │  更新状态    │        │调用代码沙箱  │
            │(Feign调用)   │        │ status=1    │        │(HTTP:8090)   │
            └──────────────┘        └──────────────┘        └──────┬───────┘
                    │                                              │
                    │                                              ▼
                    │                                     ┌──────────────┐
                    │                                     │ Docker 执行  │
                    │                                     │ • 编译代码   │
                    │                                     │ • 运行测试   │
                    │                                     │ • 收集结果   │
                    │                                     └──────┬───────┘
                    │                                              │
                    │                                              ▼
                    │                                     ┌──────────────┐
                    │                                     │ 返回执行结果  │
                    │                                     │ • 输出       │
                    │                                     │ • 时间/内存  │
                    │                                     │ • 错误信息   │
                    │                                     └──────┬───────┘
                    │                                              │
                    └─────────────────────────┬────────────────────┘
                                              │
                                              ▼
                                       ┌──────────────┐
                                       │  判题策略    │
                                       │ • 比对输出   │
                                       │ • 判定结果   │
                                       │ (AC/WA/TLE)  │
                                       └──────┬───────┘
                                              │
                                              ▼
                                       ┌──────────────┐
                                       │  更新结果    │
                                       │ status=2    │
                                       │ judgeInfo   │
                                       └──────────────┘
```

---

## 📊 核心流程详解

### 1️⃣ 代码提交流程

| 步骤 | 执行服务 | 操作 | 数据变化 |
|------|---------|------|---------|
| 1 | QuestionService | 校验编程语言合法性 | - |
| 2 | QuestionService | 检查题目存在性 | - |
| 3 | QuestionService | 创建提交记录 | `question_submit` 表插入，status=0 (WAITING) |
| 4 | QuestionService | 发送 MQ 消息 | `code_exchange` → `code_queue` |

### 2️⃣ 异步判题流程

| 步骤 | 执行服务 | 操作 | 关键技术 |
|------|---------|------|---------|
| 1 | JudgeService | 消费 MQ 消息 | `@RabbitListener(queues = "code_queue")` |
| 2 | JudgeService | 获取提交信息 | Feign 调用 QuestionService |
| 3 | JudgeService | 更新状态 | status=1 (RUNNING)，防止重复执行 |
| 4 | JudgeService | 调用代码沙箱 | HTTP POST localhost:8090/executeCode |
| 5 | CodeSandbox | Docker 执行代码 | 容器隔离、资源限制 |
| 6 | JudgeService | 策略判题 | JudgeManager + JudgeStrategy 模式 |
| 7 | JudgeService | 保存结果 | status=2 (SUCCEED)，更新 judge_info |

### 3️⃣ 服务间通信

**同步调用 (Feign)**:
```
JudgeService ──Feign──► QuestionService (获取题目/提交信息)
JudgeService ──Feign──► UserService (获取用户信息)
```

**异步调用 (RabbitMQ)**:
```
QuestionService ──MQ──► JudgeService
Exchange: code_exchange
Queue: code_queue
RoutingKey: my_routingKey
```

---

## 🧰 技术栈

### 后端技术

| 技术 | 版本 | 用途 |
|------|------|------|
| Spring Boot | 2.7.x | 基础框架 |
| Spring Cloud Gateway | 3.x | 网关 |
| Spring Cloud OpenFeign | 3.x | 服务间调用 |
| MyBatis-Plus | 3.5.x | ORM 框架 |
| RabbitMQ | 3.x | 消息队列 |
| Nacos | 2.x | 服务注册与配置 |
| Docker | 20.x | 代码沙箱容器 |

### 数据存储

| 技术 | 用途 |
|------|------|
| MySQL 8.0 | 业务数据 |
| Redis | Session、缓存 |

### 前端技术

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue | 3.x | 前端框架 |
| Arco Design | - | UI 组件库 |
| Monaco Editor | - | 代码编辑器 |

---

## 🚀 快速开始

### 环境检查

```bash
# 1. 检查 Docker 容器状态
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 2. 检查 Java 服务进程
jps -lvm | grep yuoj

# 3. 检查端口监听
ss -tlnp | grep -E "(8101|8102|8103|8104|8090)"

# 4. 检查数据库
docker exec guyu_mysql mysql -uroot -p123456 -e "SHOW DATABASES;"
```

### 服务启动顺序

```bash
# 1. 启动基础设施 (Docker)
docker start guyu_mysql guyu_redis rabbitmq nacos-quick

# 2. 启动 Code Sandbox (Port: 8090)
cd yuoj-code-sandbox-master
nohup java -jar target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar > sandbox.log 2>&1 &

# 3. 启动业务服务
cd yuoj-backend-microservice-master

# Gateway (Port: 8101)
nohup java -jar yuoj-backend-gateway/target/*.jar > gateway.log 2>&1 &

# User Service (Port: 8102)
nohup java -jar yuoj-backend-user-service/target/*.jar > user.log 2>&1 &

# Question Service (Port: 8103)
nohup java -jar yuoj-backend-question-service/target/*.jar > question.log 2>&1 &

# Judge Service (Port: 8104)
nohup java -jar yuoj-backend-judge-service/target/*.jar > judge.log 2>&1 &
```

### 验证服务状态

```bash
# Gateway 健康检查
curl http://localhost:8101/

# Code Sandbox 健康检查
curl http://localhost:8090/health

# Nacos 控制台
open http://localhost:8848/nacos
```

---

## 📝 项目结构

```
yuoj-backend-microservice-master/
├── yuoj-backend-gateway/          # 网关服务
├── yuoj-backend-user-service/     # 用户服务
├── yuoj-backend-question-service/ # 题目服务
├── yuoj-backend-judge-service/    # 判题服务
├── yuoj-backend-service-client/   # Feign 客户端
├── yuoj-backend-common/           # 公共组件
├── yuoj-backend-model/            # 数据模型
└── pom.xml

yuoj-code-sandbox-master/          # 代码沙箱
├── src/
│   ├── JavaNativeCodeSandbox.java # Java 代码沙箱
│   ├── controller/
│   └── model/
└── pom.xml

yuoj-frontend-master/              # 前端项目
├── src/
│   ├── views/                     # 页面
│   ├── components/                # 组件
│   └── api/                       # API 接口
└── package.json
```

---

## 🎓 学习要点

### 1. 微服务设计
- **服务拆分**: 按业务领域拆分（用户、题目、判题）
- **服务注册**: Nacos 实现服务发现
- **负载均衡**: Gateway 基于 Nacos 服务名路由

### 2. 异步处理
- **削峰填谷**: MQ 解耦提交和判题流程
- **可靠性**: 手动 ACK 确保消息不丢失
- **幂等性**: 判题状态机防止重复执行

### 3. 代码安全
- **沙箱隔离**: Docker 容器执行用户代码
- **资源限制**: 时间、内存限制
- **鉴权机制**: Header 密钥验证

### 4. 判题策略
- **策略模式**: JudgeStrategy 支持多语言
- **结果判定**: AC(通过)/WA(错误)/TLE(超时)/MLE(超内存)/RE(运行错误)

---

## 📚 相关文档

- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)
- [Nacos](https://nacos.io/)
- [RabbitMQ](https://www.rabbitmq.com/)
- [Docker Java API](https://github.com/docker-java/docker-java)

---

<p align="center">
  <b>Built with ❤️ for Learning Microservices Architecture</b>
</p>

## 🧪 测试示例

### Java 代码提交示例

以下是通过测试的正确 Java 代码示例（使用命令行参数方式）：

```java
public class Main {
    public static void main(String[] args) {
        int a = Integer.parseInt(args[0]);
        int b = Integer.parseInt(args[1]);
        System.out.println(a + b);
    }
}
```

**说明**：
- 类名必须是 `Main`
- 通过 `args[0]`, `args[1]` 等方式读取输入参数
- 使用 `System.out.println()` 输出结果

---

<p align="center">
  <i>Last Updated: 2026-03-11</i>
</p>
