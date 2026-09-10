## 1. 常用命令

### 1.1 后端（Maven 多模块）

三个服务均为独立 Spring Boot 应用，需分别在独立终端按 `监控 → 调度 → 主应用` 顺序启动。

**1. 环境变量**（连数据库/Redis 的终端都要先执行）

```bash
# MySQL
export MYSQL_HOST=nicomoe.cn
export MYSQL_PORT=30000
export MYSQL_ROOT_USER=root
export MYSQL_ROOT_PASSWORD='wang13587'

# Redis
export REDIS_HOST=nicomoe.cn
export REDIS_PORT=30001
export REDIS_DATABASE=0
export REDIS_PASSWORD='wang13587'
```

**2. Spring Boot Admin 服务监控** — 不连数据库，无需环境变量

```bash
mvn clean package -pl ruoyi-extend/ruoyi-monitor-admin -am
java -jar ruoyi-extend/ruoyi-monitor-admin/target/ruoyi-monitor-admin.jar
```

**3. SnailJob 分布式任务调度** — 连 MySQL

```bash
mvn clean package -pl ruoyi-extend/ruoyi-snailjob-server -am
java -jar ruoyi-extend/ruoyi-snailjob-server/target/ruoyi-snailjob-server.jar
```

**4. 主应用** — 连 MySQL + Redis

```bash
mvn clean package -pl ruoyi-admin -am
java -jar ruoyi-admin/target/ruoyi-admin.jar
```

### 1.2 前端（plus-ui）

```bash
cd plus-ui
npm install        # 安装依赖
npm run dev        # 开发启动（Vite 代理后端，端口 9090）
npm run build:prod # 生产构建
npm run lint:eslint        # ESLint 检查
npm run lint:eslint:fix    # ESLint 自动修复
npm run prettier           # 格式化全部文件
```

前端访问 `http://localhost:9090`，默认账号 `admin` / `admin123`。

## 2. 开发命令

| 命令 | 用途 | 说明 |
|------|------|------|
| `/dev` | 开发新功能 | 双模式代码生成（后端+前端），适合完整业务 |
| `/crud` | 快速 CRUD | 基于已有表快速生成标准 CRUD，适合简单模块 |
| `/check` | 代码规范检查 | 自动检测后端+前端代码是否符合项目规范 |

开发指南与技能参考 `.claude/docs/` 与 `.claude/skills/`。
