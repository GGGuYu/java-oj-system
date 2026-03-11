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

## 🚀 详细部署启动指南（新手必看）

> ⚠️ **重要提示**：这是一个测试学习项目，所有密码、端口都是明文配置的，**不要在生产环境使用！**

### 📋 启动前准备

#### 1. 环境要求

```bash
# 必须安装
- Docker 20.10+
- Java 1.8
- Maven 3.6+

# 验证安装
docker --version
java -version
mvn -version
```

#### 2. 项目结构理解

```
workspace/
├── yuoj-backend-microservice-master/   # 微服务后端（先启动基础设施，再启动Java服务）
├── yuoj-code-sandbox-master/           # 代码沙箱服务（独立部署）
└── yuoj-frontend-master/               # 前端项目（可选）
```

---

### 🔧 第一步：启动基础设施（Docker 容器）

#### 1.1 MySQL（数据库，必需）

```bash
# 检查是否已有 MySQL 容器
docker ps -a | grep mysql

# 如果没有，创建并启动（端口 3306，密码 123456）
docker run -d \
  --name guyu_mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=123456 \
  -v mysql_data:/var/lib/mysql \
  mysql:latest

# 如果已有容器，直接启动
docker start guyu_mysql

# 等待 30 秒让 MySQL 启动完成
sleep 30

# 验证连接
docker exec guyu_mysql mysql -uroot -p123456 -e "SELECT 1;"
```

#### 1.2 初始化数据库

```bash
# 进入项目目录
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-backend-microservice-master

# 执行 SQL 脚本创建数据库和表
docker exec -i guyu_mysql mysql -uroot -p123456 < mysql-init/create_table.sql

# 验证数据库创建成功
docker exec guyu_mysql mysql -uroot -p123456 -e "SHOW DATABASES;"
# 应该看到：yuoj 数据库

docker exec guyu_mysql mysql -uroot -p123456 -e "USE yuoj; SHOW TABLES;"
# 应该看到 6 张表：user, question, question_submit, post, post_thumb, post_favour
```

#### 1.3 Redis（缓存，必需）

```bash
# 检查 Redis 容器
docker ps -a | grep redis

# 如果没有，创建并启动（端口 6379，无密码）
# ⚠️ 注意：我们使用 Redis 5.0.14 版本，latest 版本需要密码认证
docker run -d \
  --name guyu_redis \
  -p 6379:6379 \
  redis:5.0.14

# 如果已有容器，直接启动
docker start guyu_redis

# 验证连接
docker exec guyu_redis redis-cli ping
# 应该返回：PONG
```

#### 1.4 Nacos（服务注册中心，必需）

```bash
# 检查 Nacos 容器
docker ps -a | grep nacos

# 如果没有，创建并启动（端口 8848）
# ⚠️ 注意：使用 v2.1.2 版本，v2.2.3 有 Derby 数据库问题
docker run -d \
  --name nacos-quick \
  -e MODE=standalone \
  -e PREFER_HOST_MODE=hostname \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 9849:9849 \
  nacos/nacos-server:v2.1.2

# 如果已有容器，直接启动
docker start nacos-quick

# 等待 30 秒让 Nacos 启动
sleep 30

# 验证 Nacos 启动成功
curl -s "http://localhost:8848/nacos/v1/ns/operator/metrics"
# 应该返回 JSON 数据

# 浏览器访问 Nacos 控制台
# http://localhost:8848/nacos
# 默认账号密码：nacos / nacos
```

#### 1.5 RabbitMQ（消息队列，必需）

```bash
# 检查 RabbitMQ 容器
docker ps -a | grep rabbitmq

# 如果没有，创建并启动（端口 5672, 15672）
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management

# 如果已有容器，直接启动
docker start rabbitmq

# 等待 20 秒让 RabbitMQ 启动
sleep 20

# 验证管理界面
# http://localhost:15672
# 默认账号密码：guest / guest
```

#### 1.6 基础设施一键启动脚本

```bash
# 如果所有容器都已创建，直接全部启动
docker start guyu_mysql guyu_redis rabbitmq nacos-quick

# 查看所有容器状态
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**预期输出：**
```
NAMES           STATUS              PORTS
guyu_mysql      Up 2 minutes        0.0.0.0:3306->3306/tcp
guyu_redis      Up 2 minutes        0.0.0.0:6379->6379/tcp
rabbitmq        Up 2 minutes        0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
nacos-quick     Up 2 minutes        0.0.0.0:8848->8848/tcp, 0.0.0.0:9848-9849->9848-9849/tcp
```

---

### ☕ 第二步：启动 Java 服务

**重要顺序**：必须先启动所有基础设施，再启动 Java 服务！

#### 2.1 构建项目（首次部署需要）

```bash
# 1. 构建代码沙箱
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-code-sandbox-master
mvn clean package -DskipTests

# 2. 构建后端微服务
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-backend-microservice-master
mvn clean package -DskipTests
```

**构建成功标志：**
- 代码沙箱：`target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar`（约 42MB）
- 网关服务：`yuoj-backend-gateway/target/yuoj-backend-gateway-0.0.1-SNAPSHOT.jar`（约 90MB）
- 用户服务：`yuoj-backend-user-service/target/yuoj-backend-user-service-0.0.1-SNAPSHOT.jar`（约 90MB）
- 题目服务：`yuoj-backend-question-service/target/yuoj-backend-question-service-0.0.1-SNAPSHOT.jar`（约 92MB）
- 判题服务：`yuoj-backend-judge-service/target/yuoj-backend-judge-service-0.0.1-SNAPSHOT.jar`（约 92MB）

#### 2.2 启动 Code Sandbox（代码沙箱，端口 8090）

```bash
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-code-sandbox-master

# 启动代码沙箱（后台运行）
nohup java -jar target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar > /tmp/sandbox.log 2>&1 &

# 查看日志
tail -f /tmp/sandbox.log

# 等待启动完成（看到 "Started YuojCodeSandboxApplication" 即可）
```

**验证沙箱启动成功：**
```bash
# 方法1：检查端口
curl http://localhost:8090/health
# 应该返回：{"status":"ok"}

# 方法2：查看进程
jps -l | grep sandbox
# 应该看到：yuoj-code-sandbox-0.0.1-SNAPSHOT.jar
```

#### 2.3 启动后端微服务

**注意**：以下命令需要在 `yuoj-backend-microservice-master` 目录下执行，且每个服务在不同的终端窗口中启动，方便查看日志。

```bash
# 进入后端项目目录
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-backend-microservice-master
```

**2.3.1 启动 Gateway（网关，端口 8101）**

```bash
nohup java -jar yuoj-backend-gateway/target/yuoj-backend-gateway-0.0.1-SNAPSHOT.jar > /tmp/gateway.log 2>&1 &

# 等待启动（约 10-20 秒）
sleep 20

# 验证
curl http://localhost:8101/
# 应该返回 Gateway 的信息
```

**2.3.2 启动 User Service（用户服务，端口 8102）**

```bash
nohup java -jar yuoj-backend-user-service/target/yuoj-backend-user-service-0.0.1-SNAPSHOT.jar > /tmp/user.log 2>&1 &

# 等待启动
sleep 20

# 验证（通过 Gateway）
curl http://localhost:8101/api/user/get/login
# 应该返回登录信息或错误信息（表示服务已注册）
```

**2.3.3 启动 Question Service（题目服务，端口 8103）**

```bash
nohup java -jar yuoj-backend-question-service/target/yuoj-backend-question-service-0.0.1-SNAPSHOT.jar > /tmp/question.log 2>&1 &

# 等待启动
sleep 20

# 验证
curl http://localhost:8101/api/question/list/page
```

**2.3.4 启动 Judge Service（判题服务，端口 8104）**

```bash
nohup java -jar yuoj-backend-judge-service/target/yuoj-backend-judge-service-0.0.1-SNAPSHOT.jar > /tmp/judge.log 2>&1 &

# 等待启动
sleep 20

# 验证（这个服务没有直接暴露的 HTTP 接口，通过 Nacos 查看）
```

#### 2.4 验证所有服务已注册到 Nacos

```bash
# 查看 Nacos 中注册的服务数量
curl "http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10"

# 应该看到返回的数据中包含 4 个服务：
# - yuoj-backend-gateway
# - yuoj-backend-user-service
# - yuoj-backend-question-service
# - yuoj-backend-judge-service
```

**或者在浏览器中查看：**
```
http://localhost:8848/nacos
账号：nacos
密码：nacos
```

---

### 🌐 第三步：访问系统

#### 3.1 API 文档（Knife4j）

```
http://localhost:8101/doc.html
```

这个页面可以看到所有 API 接口，并且可以在线测试。

#### 3.2 服务端口对照表

| 服务 | 端口 | 访问地址 | 说明 |
|------|------|----------|------|
| Gateway | 8101 | http://localhost:8101 | API 统一入口 |
| 代码沙箱 | 8090 | http://localhost:8090 | 独立服务 |
| User Service | 8102 | http://localhost:8102 | 直接访问（不推荐） |
| Question Service | 8103 | http://localhost:8103 | 直接访问（不推荐） |
| Judge Service | 8104 | http://localhost:8104 | 直接访问（不推荐） |
| Nacos | 8848 | http://localhost:8848/nacos | 服务注册中心 |
| RabbitMQ | 5672/15672 | http://localhost:15672 | 消息队列管理 |
| MySQL | 3306 | localhost:3306 | 数据库 |
| Redis | 6379 | localhost:6379 | 缓存 |

#### 3.3 测试接口示例

```bash
# 1. 测试网关是否工作
curl http://localhost:8101/

# 2. 测试用户注册（需要先获取验证码，略）
# curl -X POST http://localhost:8101/api/user/register \
#   -H "Content-Type: application/json" \
#   -d '{"userAccount":"test","userPassword":"12345678","checkPassword":"12345678"}'

# 3. 测试题目列表
curl "http://localhost:8101/api/question/list/page?pageSize=10"

# 4. 测试代码沙箱健康检查
curl http://localhost:8090/health
```

---

### 🔍 第四步：排查问题

#### 4.1 查看日志

```bash
# 代码沙箱日志
tail -f /tmp/sandbox.log

# 网关日志
tail -f /tmp/gateway.log

# 用户服务日志
tail -f /tmp/user.log

# 题目服务日志
tail -f /tmp/question.log

# 判题服务日志
tail -f /tmp/judge.log
```

#### 4.2 常见问题

**问题1：Java 服务启动失败，提示连接 Nacos 失败**
```
解决方案：确保 Nacos 已启动并可以访问
curl http://localhost:8848/nacos/v1/ns/operator/metrics
```

**问题2：Java 服务提示连接 MySQL 失败**
```
解决方案：
1. 检查 MySQL 容器是否运行：docker ps | grep mysql
2. 检查数据库是否存在：docker exec guyu_mysql mysql -uroot -p123456 -e "SHOW DATABASES;"
3. 检查表是否存在：docker exec guyu_mysql mysql -uroot -p123456 -e "USE yuoj; SHOW TABLES;"
```

**问题3：Java 服务提示连接 Redis 失败**
```
解决方案：
1. 检查 Redis 容器是否运行：docker ps | grep redis
2. 检查 Redis 版本：我们使用 5.0.14，latest 版本需要密码
3. 测试连接：docker exec guyu_redis redis-cli ping
```

**问题4：判题服务无法连接代码沙箱**
```
解决方案：
1. 确保代码沙箱已启动：curl http://localhost:8090/health
2. 检查沙箱日志：tail -f /tmp/sandbox.log
```

**问题5：Nacos 注册的服务数量为 0**
```
解决方案：
1. 等待 Java 服务完全启动（每个服务需要 20-30 秒）
2. 检查 Java 服务日志是否有错误
3. 检查 Nacos 是否正常运行：curl http://localhost:8848/nacos/v1/ns/operator/metrics
```

#### 4.3 一键检查脚本

```bash
#!/bin/bash
echo "=== 检查 Docker 容器 ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo ""
echo "=== 检查端口监听 ==="
ss -tlnp | grep -E "(3306|6379|5672|15672|8848|8090|8101|8102|8103|8104)"

echo ""
echo "=== 检查 Java 进程 ==="
jps -l | grep yuoj

echo ""
echo "=== 检查 Nacos 服务注册 ==="
curl -s "http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10" | head -1
```

---

### ⏹️ 第五步：停止服务

#### 5.1 停止 Java 服务

```bash
# 停止所有 Java 服务
jps -l | grep yuoj | awk '{print $1}' | xargs kill -9

# 或者分别停止
# 查看进程 ID
jps -l | grep yuoj
# 然后 kill <PID>
```

#### 5.2 停止 Docker 容器

```bash
# 停止所有基础设施容器
docker stop guyu_mysql guyu_redis rabbitmq nacos-quick

# 或者全部停止
docker stop $(docker ps -q)
```

---

### 📊 完整启动顺序总结

```
【必须严格按照这个顺序启动】

第1步：启动基础设施（Docker）
├─ 1.1 MySQL (3306) - docker start guyu_mysql
├─ 1.2 Redis (6379) - docker start guyu_redis
├─ 1.3 RabbitMQ (5672/15672) - docker start rabbitmq
└─ 1.4 Nacos (8848) - docker start nacos-quick
   └─ 等待 30 秒让 Nacos 完全启动

第2步：初始化数据库（如果还没创建）
└─ docker exec -i guyu_mysql mysql -uroot -p123456 < mysql-init/create_table.sql

第3步：启动 Java 服务（按顺序，每个等待 20 秒）
├─ 3.1 Code Sandbox (8090) - 代码沙箱
├─ 3.2 Gateway (8101) - 网关
├─ 3.3 User Service (8102) - 用户服务
├─ 3.4 Question Service (8103) - 题目服务
└─ 3.5 Judge Service (8104) - 判题服务

第4步：验证服务
├─ 检查 Nacos 注册：curl http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10
├─ 检查沙箱健康：curl http://localhost:8090/health
└─ 访问 API 文档：http://localhost:8101/doc.html

【所有密码】
- MySQL root: 123456
- Redis: 无密码（使用 5.0.14 版本）
- Nacos: nacos / nacos
- RabbitMQ: guest / guest
```

---

### 🎯 快速启动（熟练后使用）

如果你已经创建好了所有容器，只需要执行：

```bash
# 1. 一键启动所有基础设施
docker start guyu_mysql guyu_redis rabbitmq nacos-quick

# 等待 30 秒
echo "等待基础设施启动..."
sleep 30

# 2. 一键启动所有 Java 服务
cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-code-sandbox-master
nohup java -jar target/yuoj-code-sandbox-0.0.1-SNAPSHOT.jar > /tmp/sandbox.log 2>&1 &

cd /home/guyu/Ever/Project/oj-backend-and-codebox/yuoj-backend-microservice-master
nohup java -jar yuoj-backend-gateway/target/yuoj-backend-gateway-0.0.1-SNAPSHOT.jar > /tmp/gateway.log 2>&1 &
nohup java -jar yuoj-backend-user-service/target/yuoj-backend-user-service-0.0.1-SNAPSHOT.jar > /tmp/user.log 2>&1 &
nohup java -jar yuoj-backend-question-service/target/yuoj-backend-question-service-0.0.1-SNAPSHOT.jar > /tmp/question.log 2>&1 &
nohup java -jar yuoj-backend-judge-service/target/yuoj-backend-judge-service-0.0.1-SNAPSHOT.jar > /tmp/judge.log 2>&1 &

# 等待 30 秒
echo "等待 Java 服务启动..."
sleep 30

# 3. 验证
echo "=== 检查服务状态 ==="
curl http://localhost:8090/health
curl "http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10"
echo ""
echo "API 文档：http://localhost:8101/doc.html"
echo "Nacos 控制台：http://localhost:8848/nacos (nacos/nacos)"
```

---

## 📝 附录：配置信息汇总

### 数据库连接信息

```yaml
# MySQL
host: localhost
port: 3306
database: yuoj
username: root
password: 123456

# Redis
host: localhost
port: 6379
database: 1
password: (空，使用 Redis 5.0.14)
```

### 中间件信息

```yaml
# Nacos
host: localhost
port: 8848
username: nacos
password: nacos

# RabbitMQ
host: localhost
port: 5672 (服务端口) / 15672 (管理界面)
username: guest
password: guest
vhost: /

exchange: code_exchange
queue: code_queue
routingKey: my_routingKey
```

### 服务配置

```yaml
# Code Sandbox
port: 8090
authHeader: auth
authKey: secretKey

# Gateway
port: 8101

# User Service
port: 8102

# Question Service
port: 8103

# Judge Service
port: 8104
codeSandbox.type: remote  # 使用远程沙箱
```

---

<p align="center">
  <i>Last Updated: 2026-03-11</i>
</p>
