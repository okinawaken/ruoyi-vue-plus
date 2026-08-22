# RuoYi-Vue-Plus 多租户管理系统

基于 Spring Boot 3 + Vue 3 的企业级多租户快速开发平台，采用前后端分离架构。后端以 Sa-Token 认证、MyBatis-Plus ORM、Warm-Flow 工作流为核心，内置代码生成器与丰富的公共组件开箱即用。

## 功能特性

- **多租户支持**：行级数据隔离，TenantEntity 基类 + 租户过滤，轻松构建 SaaS 应用
- **权限认证**：Sa-Token + JWT，支持 RBAC 菜单权限、按钮权限、数据权限（部门/本人/自定义）
- **工作流引擎**：集成 Warm-Flow 国产工作流，支持审批、驳回、转办、委派、加签等操作
- **分布式任务调度**：SnailJob 可视化调度中心，支持失败重试、任务分片、广播任务
- **代码生成器**：根据数据库表一键生成前后端 CRUD 代码，支持 Velocity 模板自定义
- **对象存储**：S3 协议统一封装，支持 MinIO、阿里云 OSS、腾讯云 COS、七牛云等
- **消息能力**：SMS4j 多厂商短信、邮件发送、SSE/WebSocket 实时推送
- **第三方登录**：JustAuth 集成 Gitee、GitHub、钉钉、微信等多平台 OAuth 登录
- **安全加固**：数据脱敏（@Sensitive）、字段加密（@EncryptField）、接口限流（@RateLimiter）、防重复提交（@RepeatSubmit）

## 技术栈

### 后端

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

### 前端（plus-ui）

| 技术 | 说明 |
|------|------|
| Vue 3.5 | 渐进式框架 |
| Element Plus | UI 组件库 |
| TypeScript | 类型安全 |
| Vite | 构建工具 |
| Pinia | 状态管理 |
| vxe-table | 高性能表格 |
| UnoCSS | 原子化 CSS |

## 项目结构

```
RuoYi-Vue-Plus/
├── ruoyi-admin/                 # 后端启动入口（端口 8080）
│   └── src/main/resources/
│       ├── application.yml      # 主配置（数据库/Redis 支持环境变量注入）
│       └── logback-plus.xml     # 日志配置
├── ruoyi-common/                # 公共模块（24 个子模块）
│   ├── ruoyi-common-core/       # 核心工具（StringUtils、MapstructUtils 等）
│   ├── ruoyi-common-mybatis/    # MyBatis 扩展（BaseMapperPlus、分页）
│   ├── ruoyi-common-tenant/     # 多租户
│   ├── ruoyi-common-satoken/    # 权限认证
│   ├── ruoyi-common-redis/      # Redis 缓存
│   ├── ruoyi-common-web/        # Web 通用配置
│   ├── ruoyi-common-doc/        # 接口文档
│   └── ...                      # excel、oss、sms、mail、log、encrypt、sensitive 等
├── ruoyi-modules/               # 业务模块
│   ├── ruoyi-system/            # 系统管理（用户、角色、菜单、部门等）
│   ├── ruoyi-generator/         # 代码生成器
│   ├── ruoyi-job/               # 定时任务客户端
│   ├── ruoyi-workflow/          # 工作流模块
│   └── ruoyi-demo/              # 功能演示
├── ruoyi-extend/                # 扩展服务（独立部署，按需启动）
│   ├── ruoyi-monitor-admin/     # Spring Boot Admin 服务监控
│   └── ruoyi-snailjob-server/   # SnailJob 调度服务端
├── plus-ui/                     # PC 管理端前端
├── script/sql/                  # 数据库脚本
│   ├── ry_vue_5.X.sql           # 系统核心表
│   ├── ry_workflow.sql          # 工作流表
│   └── ry_job.sql               # 任务调度表
├── .run/                        # IDEA 共享运行配置
└── pom.xml                      # Maven 父工程
```

## 快速开始

### 环境要求

| 依赖 | 要求 |
|------|------|
| JDK | ≥ 17 |
| Maven | ≥ 3.8 |
| Node.js | ≥ 18 |
| MySQL | ≥ 8.0 |
| Redis | ≥ 6.0 |

### 1. 初始化数据库

创建数据库 `ry-vue`（utf8mb4），依次执行 `script/sql/` 下的脚本：

```sql
CREATE DATABASE `ry-vue` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

```bash
mysql -uroot -p ry-vue < script/sql/ry_vue_5.X.sql    # 必执行：系统核心表
mysql -uroot -p ry-vue < script/sql/ry_workflow.sql   # 使用工作流时执行
mysql -uroot -p ry-vue < script/sql/ry_job.sql        # 使用任务调度时执行
```

> 如需使用 SnailJob 调度中心，还需启动 `ruoyi-extend/ruoyi-snailjob-server`。

### 2. 启动后端

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

### 3. 启动前端

```bash
cd plus-ui
npm install
npm run dev
```

浏览器访问 <http://localhost:80>（Vite 自动代理后端接口）。

默认账号：`admin` / `admin123`

## 内置功能

<details>
<summary><strong>系统管理</strong></summary>

- 用户管理：用户的增删改查、角色分配、部门分配、重置密码
- 角色管理：角色权限分配、数据权限分配（全部/自定义/本部门/本部门及以下/仅本人）
- 菜单管理：菜单与按钮权限配置、路由动态加载
- 部门管理：树形组织架构管理
- 岗位管理：岗位编码与排序维护
- 字典管理：业务枚举数据的统一维护
- 参数设置：系统运行参数动态配置
- 通知公告：公告的发布与管理
- 操作日志 / 登录日志：全量审计追踪
- 文件管理：OSS 文件上传下载与存储配置切换
- 租户管理：租户套餐、租户企业信息维护（多租户版特有）

</details>

<details>
<summary><strong>系统监控</strong></summary>

- 在线用户：查看并强退在线会话
- 缓存监控：Redis 运行状态与命中率
- 缓存列表：键值查询与批量清理
- 服务监控：JVM、磁盘、CPU、内存监控
- 定时任务：SnailJob 调度中心可视化管理

</details>

<details>
<summary><strong>系统工具</strong></summary>

- 表单构建：拖拽式表单设计
- 代码生成：导入表结构，生成前后端完整 CRUD 代码
- 系统接口：SpringDoc 在线 API 文档

</details>

## 开发规范

本项目约定三层架构（Controller → Service → Mapper），核心规范摘要：

- 包名统一使用 `org.dromara.*`
- 对象转换必须使用 `MapstructUtils.convert()`，禁止 BeanUtil
- 实体类继承 `TenantEntity`，主键使用雪花 ID，禁止自增
- BO 使用 `@AutoMapper` 注解，Mapper 继承 `BaseMapperPlus`
- 业务数据传递使用 VO，禁止 `Map<String, Object>`

详细规范见 [CLAUDE.md](CLAUDE.md) 与 `.claude/skills/` 技能库。

## 许可证

[MIT](LICENSE)
