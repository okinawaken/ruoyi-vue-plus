## 1. 常用命令

### 1.1 后端（Maven 多模块）

```bash
# 完整启动：按顺序启动监控 → 调度 → 主应用（均为独立 Spring Boot 应用）

# 1. Spring Boot Admin 服务监控
mvn clean package -pl ruoyi-extend/ruoyi-monitor-admin -am
java -jar ruoyi-extend/ruoyi-monitor-admin/target/ruoyi-monitor-admin.jar

# 2. SnailJob 分布式任务调度
mvn clean package -pl ruoyi-extend/ruoyi-snailjob-server -am
java -jar ruoyi-extend/ruoyi-snailjob-server/target/ruoyi-snailjob-server.jar

# 3. 打包并启动主应用
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
