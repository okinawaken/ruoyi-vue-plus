# RuoYi-Vue-Plus 多租户管理系统

基于 Spring Boot 3 + Vue 3 的企业级多租户快速开发平台，采用前后端分离架构。后端以 Sa-Token 认证、MyBatis-Plus ORM、Warm-Flow 工作流为核心，内置代码生成器与丰富的公共组件开箱即用。

## 1. 功能特性

- **多租户支持**：行级数据隔离，TenantEntity 基类 + 租户过滤，轻松构建 SaaS 应用
- **权限认证**：Sa-Token + JWT，支持 RBAC 菜单权限、按钮权限、数据权限（部门/本人/自定义）
- **工作流引擎**：集成 Warm-Flow 国产工作流，支持审批、驳回、转办、委派、加签等操作
- **分布式任务调度**：SnailJob 可视化调度中心，支持失败重试、任务分片、广播任务
- **代码生成器**：根据数据库表一键生成前后端 CRUD 代码，支持 Velocity 模板自定义
- **对象存储**：S3 协议统一封装，支持 MinIO、阿里云 OSS、腾讯云 COS、七牛云等
- **消息能力**：SMS4j 多厂商短信、邮件发送、SSE/WebSocket 实时推送
- **第三方登录**：JustAuth 集成 Gitee、GitHub、钉钉、微信等多平台 OAuth 登录
- **安全加固**：数据脱敏（@Sensitive）、字段加密（@EncryptField）、接口限流（@RateLimiter）、防重复提交（@RepeatSubmit）

## 2. 技术栈

### 2.1 后端

| 技术 | 版本 | 说明 |
|------|------|------|
| Java | 17 | 运行环境 |
| Spring Boot | 3.5.15 | 核心框架 |
| MyBatis-Plus | 3.5.16 | ORM 框架 |
| Sa-Token | 1.45.0 | 权限认证 |
| Redisson | 3.52.0 | Redis 客户端 / 分布式锁 |
| Warm-Flow | 1.8.5 | 工作流引擎 |
| SnailJob | 1.10.0 | 分布式任务调度 |
| Mapstruct-Plus | 1.5.0 | 对象转换 |
| SpringDoc | 2.8.17 | 接口文档（OpenAPI 3） |

### 2.2 前端（plus-ui）

| 技术 | 版本 | 说明 |
|------|------|------|
| Vue | 3.5.30 | 渐进式框架 |
| Element Plus | 2.13.5 | UI 组件库 |
| TypeScript | 5.9.3 | 类型安全 |
| Vite | 7.3.2 | 构建工具 |
| Pinia | 3.0.4 | 状态管理 |
| vxe-table | 4.18.1 | 高性能表格 |
| UnoCSS | 66.6.6 | 原子化 CSS |

## 3. 快速开始

### 3.1 环境要求

| 依赖 | 要求 |
|------|------|
| JDK | ≥ 17 |
| Maven | ≥ 3.8 |
| Node.js | ≥ 18 |
| MySQL | ≥ 8.0 |
| Redis | ≥ 6.0 |

### 3.2 初始化数据库

创建数据库 `ry-vue`（utf8mb4），依次执行 `script/sql/` 下的脚本：

```sql
CREATE DATABASE `ry-vue` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

```bash
mysql -uroot -p ry-vue < script/sql/ry_vue_5.X.sql    # 必执行：系统核心表
mysql -uroot -p ry-vue < script/sql/ry_workflow.sql   # 使用工作流时执行
mysql -uroot -p ry-vue < script/sql/ry_job.sql        # 使用任务调度时执行
```

### 3.3 启动后端

默认连接 `localhost:3306/ry-vue`（root/root）与 `localhost:6379`（密码 ruoyi123），本地环境不一致时通过环境变量覆盖：

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `MYSQL_HOST` | localhost | MySQL 地址 |
| `MYSQL_PORT` | 3306 | MySQL 端口 |
| `MYSQL_ROOT_USER` | root | MySQL 用户名 |
| `MYSQL_ROOT_PASSWORD` | root | MySQL 密码 |
| `REDIS_HOST` | localhost | Redis 地址 |
| `REDIS_PORT` | 6379 | Redis 端口 |
| `REDIS_DATABASE` | 0 | Redis 数据库编号 |
| `REDIS_PASSWORD` | ruoyi123 | Redis 密码 |

```bash
# 方式一：Maven 命令启动
mvn clean package -pl ruoyi-admin -am
java -jar ruoyi-admin/target/ruoyi-admin.jar

# 方式二：IDEA 直接运行主类 org.dromara.DromaraApplication
# 推荐使用 .run/ 目录下的共享运行配置（已预置环境变量占位）
```

启动成功后访问接口文档：<http://localhost:8080/swagger-ui/index.html>

### 3.4 启动前端

```bash
cd plus-ui
npm install
npm run dev
```

浏览器访问 <http://localhost:9090>（Vite 自动代理后端接口）。

默认账号：`admin` / `admin123`
