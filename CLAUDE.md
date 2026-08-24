## 1. 常用命令

### 1.1 后端（Maven 多模块）

```bash
# 打包后端（生成 ruoyi-admin/target/ruoyi-admin.jar）
mvn clean package -pl ruoyi-admin -am

# 启动后端（默认连接 localhost:3306/ry-vue、localhost:6379）
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

## 2. 项目文档与命令

### 2.1 文档索引

| 路径 | 内容 |
|------|------|
| `.claude/docs/` | 后端开发指南、框架说明、数据库设计规范、工作流开发指南等 |
| `.claude/skills/` | 各功能开发技能（CRUD、数据权限、安全、定时任务等） |
| `.claude/commands/` | 10 个自定义命令 |

### 2.2 常用命令速查

| 命令 | 用途 | 说明 |
|------|------|------|
| `/dev` | 开发新功能 | 双模式代码生成（后端+前端），适合完整业务 |
| `/crud` | 快速 CRUD | 基于已有表快速生成标准 CRUD，适合简单模块 |
| `/start` | 项目快速了解 | 新成员上手必看，自动扫描模块生成概览 |
| `/progress` | 项目进度 | 分析后端开发进度、识别完成度和待办 |
| `/check` | 代码规范检查 | 自动检测后端+前端代码是否符合项目规范 |
| `/next` | 下一步建议 | 根据当前状态推荐下一步开发方向 |
| `/sync` | 代码状态同步 | 同步后端代码状态，保持文档与代码一致 |
| `/update-status` | 状态增量更新 | 自动扫描最近变更，智能更新项目文档 |
| `/add-todo` | 添加待办事项 | 快速添加待办任务，支持联动项目状态 |
| `/init-docs` | 初始化业务文档 | 根据模式初始化项目文档 |
