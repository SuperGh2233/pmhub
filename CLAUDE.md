# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

PmHub 是一套基于 **SpringCloud Alibaba & LLM** 的微服务智能项目管理系统（若依 RuoYi-Cloud 二次开发）。包含认证、流程（Flowable/BPM）、项目管理、代码生成、定时任务、网关、监控等微服务，前端为 Vue 2 + Element UI。

本目录是**微服务版本**。仓库根目录同时包含单体架构版本 `pmhub-boot/`（独立 Maven 工程，不要混淆）：单体版启动 `pmhub-admin` 中的 `PmhubApplication`，微服务版按下面每条说明分别启动各服务。

## 常用命令

### 后端（Maven，Java 8，阿里巴巴 Maven 镜像已在根 pom 配置）

```bash
# 全量构建并跳过测试（服务模块多，建议跳过测试）
mvn clean package -Dmaven.test.skip=true

# 构建单个服务（如 pmhub-project）
mvn clean package -Dmaven.test.skip=true -pl pmhub-modules/pmhub-project -am

# 安装公共模块到本地仓库（新增/改动 pmhub-base、pmhub-api 后必须先执行）
mvn install -Dmaven.test.skip=true
```

项目**没有单元测试**（无 test 目录），`-Dmaven.test.skip=true` 是常规操作。

### 前端（pmhub-ui，Vue 2 / vue-cli 4）

```bash
npm install --registry=https://registry.npmmirror.com   # 不要用 cnpm
npm run dev      # 开发模式，端口 1024（Node 版本过新时改用 npm run serve）
npm run serve    # 若无 NODE_OPTIONS 环境变量则用这个
npm run build:prod      # 生产构建
npm run build:stage     # 测试环境构建
set NODE_OPTIONS=--openssl-legacy-provider   # Windows 遇 digital envelope 错误时先执行
```

## 微服务启动顺序与端口

所有服务从 Nacos 拉取配置（bootstrap.yml），**必须先启动 MySQL、Redis、Nacos**，并将各服务的 `pmhub-xxx-dev.yml` 配置文件中的数据库连接改为本机配置（配置文件在 Nacos，非项目内）。`sql/` 目录下每个服务对应一个数据库文件。

1. **pmhub-gateway** — `PmHubGatewayApplication`，端口 **6880**。聚合 Swagger、统一鉴权（`filter/` 下 AuthFilter、BlackListUrlFilter、ValidateCodeFilter、XssFilter、CacheRequestFilter）。
2. **pmhub-auth** — `PmHubAuthApplication`，端口 **6800**。OAuth2/登录认证，产 token。
3. **pmhub-system** — `PmHubSystemApplication`，端口 **6801**。系统管理（用户/角色/菜单/日志/字典）。
4. **pmhub-project** — `PmHubProjectApplication`，端口 **6806**。核心业务：项目、任务、阶段、成员、文件、日志。
5. **pmhub-workflow** — `PmHubWorkflowApplication`，端口 **6808**。Flowable 审批流、流程设计、表单设计。
6. **pmhub-gen** — `PmHubGenApplication`，端口 **6802**。代码生成（Velocity 模板）。
7. **pmhub-job** — `PmHubJobApplication`，端口 **6803**。Quartz 定时任务（含 AI 相关 job，见下）。
8. **pmhub-monitor** — `PmHubMonitorApplication`，端口 **6888**。Spring Boot Admin 监控面板，访问 `/wallboard`。

各 Application 类位置：`pmhub-gateway|pmhub-auth|pmhub-monitor/src/main/java/com/laigeoffer/pmhub/…/`、`pmhub-modules/pmhub-{system,project,workflow,gen,job}/...`。

## 模块架构

```
pmhub-ui              # 前端 Vue2 + Element UI + vuex + vue-router
pmhub-gateway [6880]  # 网关：路由、鉴权过滤链、Xss、验证码
pmhub-auth   [6800]   # 认证中心
pmhub-api             # OpenFeign 接口定义与 DTO（pmhub-api-system、pmhub-api-workflow）
pmhub-base            # 通用组件（被所有服务依赖，改动需 mvn install）
│   ├── pmhub-base-core        # 基座：返回体 R、异常、注解、工具类
│   ├── pmhub-base-datasource  # 数据源/MyBatis 配置
│   ├── pmhub-base-security    # Spring Security 安全组件
│   ├── pmhub-base-seata       # 分布式事务
│   ├── pmhub-base-notice      # RocketMQ 消息
│   ├── pmhub-base-sensitive   # 数据脱敏
│   └── pmhub-base-swagger     # Knife4j/Swagger
pmhub-modules         # 业务服务
│   ├── pmhub-system [6801]
│   ├── pmhub-gen     [6802]
│   ├── pmhub-job     [6803]
│   ├── pmhub-project [6806]   # 项目/任务/日志（职责链工厂模式，见下）
│   └── pmhub-workflow [6808]  # Flowable 流程
pmhub-monitor [6888]  # 监控中心
docker/               # docker-compose + 各基础组件（mysql/redis/nacos/seata）配置
sql/                  # 每个微服务一个 SQL
```

## 关键业务结构

**pmhub-project（核心业务模块）** 按传统分层：`controller → service(接口+impl) → domain → mapper`。其中 `service/task/` 目录是典型的**策略+工厂模式**：`QueryLogAbstractExecutor` 抽象执行器 + `QueryLogEnum` 枚举 + `QueryLogFactory` 工厂，各 `QueryXxxLogExecutor` 按类型查询项目任务操作日志。新增日志类型 = 加一个枚举 + 一个 Executor。

**pmhub-api** 存放跨服务的 OpenFeign 接口（如 system/workflow RPC），被各业务服务依赖，改动后要 `mvn install` 才生效。

**pmhub-base-core** 的 `core/` 包含统一返回体、全局异常处理、注解（如 `@Log` 操作日志切面）、XSS 过滤、工具类。各服务均依赖它。

## 数据库

- `sql/pmhub-system.sql`、`pmhub-project.sql`、`pmhub-workflow.sql`、`pmhub-gen.sql`、`pmhub_nacos.sql`、`pmhub_seata.sql` —— Nacos、Seata 各一库，其余每服务各一库。
- 服务通过 Nacos 配置中心管理数据源（Druid 连接池），不在服务本地配置。

## 基础设施（非必须但被大量核心功能使用）

- **Nacos**：服务注册发现 + 配置中心（必须）。默认账号密码 `nacos/nacos`。
- **RocketMQ**：通知模块（pmhub-base-notice）走消息队列。
- **Seata**：`pmhub-base-seata` 提供分布式事务。
- **Sentinel**：熔断限流。
- **SkyWalking**：链路追踪。
- ES、MongoDB、Ali-OSS、FastDFS：部分功能可选依赖。

## 注意事项

- Java 8，Spring Boot 2.7.18，Spring Cloud 2021.0.8 / Alibaba 2021.0.5.0。
- 前后端通过反向代理解决跨域；前端 dev 代理配置在 `pmhub-ui/vue.config.js`。
- 修改 `pmhub-base`、`pmhub-api` 下任意模块后，记得先 `mvn install` 再构建依赖它的服务。
- git commit 规范见 `pmhub-ui/README.md`：`feat/模块名`、`bugfix/说明`、`refactor/说明` 等。