---
name: dev
description: |
  当需要从零开始开发新功能、完整开发流程时自动使用此 Skill。

  触发场景：
  - 需要从零开始开发一个新功能
  - 需要设计数据库表并生成代码
  - 需要完整的开发流程引导
  - 需要配置代码生成器并生成后端代码

  触发词：开发功能、dev、新功能、功能开发、从零开发、完整开发、开发新模块
---

# /dev - 开发新功能（双模式代码生成）

作为新功能开发助手，支持两种代码生成模式：直接编写代码或生成配置后手动生成。专为 RuoYi-Vue-Plus 三层架构（Controller → Service → Mapper）设计。

## 核心优势

- **双模式选择**：直接编写代码（全自动）/ 生成配置后手动生成（可微调）
- **后端三层架构**：Controller → Service → Mapper，无 DAO 层（如 plus-ui 存在则同时生成前端）
- **包名适配**：`org.dromara.*`
- Token 消耗低（配置模式约 2000 tokens）
- 全自动执行，无需多次确认
- 智能推断模块、前缀、字典

## 两种模式对比

| 特性 | 模式A：直接编写代码 | 模式B：生成配置 |
|------|-------------------|----------------|
| **适用场景** | 标准 CRUD，无需微调 | 需要调整字段配置后再生成 |
| **自动化程度** | 全自动（建表+配置+AI直接编写代码+菜单导入） | 半自动（建表+配置，手动去代码生成器生成） |
| **可调整性** | AI 编写后可直接修改代码 | 生成前可在代码生成器界面微调列配置 |
| **执行速度** | 最快（一次性完成） | 需要额外手动操作 |
| **前端生成** | 如 plus-ui 存在则自动生成 | 代码生成器统一处理 |

---

## 执行流程

### 第一步：询问需求与模式选择

使用 AskUserQuestion 工具同时询问以下问题：

**问题1：功能信息**
```
请告诉我要开发的功能：
1. **功能名称**？（如：广告管理、反馈管理）
2. **所属模块**？（system/business/其他）
```

**问题2：生成模式选择**（使用 AskUserQuestion）

| 选项 | 说明 |
|------|------|
| **直接编写代码（推荐）** | 建表 → 配置 → AI 直接编写全部后端代码文件 + 菜单SQL，全程无需手动操作 |
| **生成配置，手动生成** | 建表 → 配置 → 在代码生成器界面微调后手动点击生成 |

**自动推断配置**（无需确认）：

| 模块 | 表前缀 | 包名 | 上级菜单 |
|------|--------|------|---------|
| system | `sys_` | `org.dromara.system` | 系统管理 |
| business | `b_` | `org.dromara.business` | 业务管理 |
| 其他（如 demo） | `demo_` | `org.dromara.demo` | [模块]管理 |

### 第一步续：检查活跃任务关联

**自动扫描** `docs/tasks/active/` 是否有与当前功能相关的活跃任务：

```bash
# 扫描活跃任务
ls docs/tasks/active/
```

**处理逻辑**：

| 情况 | 处理方式 |
|------|---------|
| 存在相关活跃任务 | 显示关联信息，开发完成后可更新 |
| 无相关任务或目录不存在 | 静默跳过，不输出 |

---

### 第二步：功能重复检查（强制执行）⭐⭐⭐⭐⭐

**自动检查以下内容**：

```bash
# 1. 检查后端代码
Grep pattern: "[功能名]Service" path: ruoyi-modules/ output_mode: files_with_matches
Grep pattern: "[功能名]Controller" path: ruoyi-modules/ output_mode: files_with_matches

# 2. 检查数据库表
mysql ... -e "SHOW TABLES LIKE '[表前缀]%';"

# 3. 检查菜单
mysql ... -e "SELECT menu_name FROM sys_menu WHERE menu_name LIKE '%[功能名]%';"
```

**处理结果**：

**如果功能已存在** → 停止执行，输出：
```markdown
功能已存在，避免重复开发！

**已有实现**：
- 后端：[Service/Controller位置]
- 数据库表：[表名]

建议：
- 增强功能：在现有代码中添加方法
- 修改功能：使用 Read 工具查看现有代码

是否仍要继续？（通常不建议）
```

**如果功能未存在** → 继续执行。

---

### 第三步：数据库现状分析（自动执行）

**数据库连接信息必须从 `application-dev.yml` 动态读取，不要硬编码！**

```bash
# 1. 读取数据库配置
Read ruoyi-admin/src/main/resources/application-dev.yml

# 2. 从配置文件中解析数据库连接信息
# 格式：${环境变量:默认值}
# 示例：${DB_HOST:127.0.0.1} → 127.0.0.1

# 3. 连接数据库查询（使用解析出的配置）
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
-- 查询最大ID（用于生成新ID）
SELECT MAX(menu_id) FROM sys_menu;
SELECT MAX(table_id) FROM gen_table;
SELECT MAX(dict_id) FROM sys_dict_type;
SELECT MAX(dict_code) FROM sys_dict_data;

-- 查询上级菜单（确定菜单归属）
-- ⚠️ 重要：记录查询结果中的 menu_id，后续步骤需要使用
SELECT menu_id, menu_name, order_num
FROM sys_menu
WHERE menu_type = 'M' AND parent_id = 0 AND del_flag = 0
ORDER BY order_num;

-- 查询各模块下的子菜单（用于计算菜单顺序）
SELECT parent_id, MAX(order_num) as max_order
FROM sys_menu
WHERE menu_type = 'C' AND del_flag = 0
GROUP BY parent_id;

-- 查询现有字典类型（避免重复创建）
SELECT dict_type, dict_name FROM sys_dict_type;
EOF
```

---

### 第四步：智能表结构设计

#### 4.1 数据库规范学习

```bash
Read CLAUDE.md
```

#### 4.2 智能字段命名和推断

根据字段名后缀自动推断控件和查询方式：

| 字段后缀 | 推断结果 | 控件类型 | 查询方式 | 字典类型 |
|---------|---------|---------|---------|---------|
| `xxx_name` | 名称 | input | LIKE | - |
| `xxx_title` | 标题 | input | LIKE | - |
| `xxx_content` | 内容 | editor | - | - |
| `remark` | 备注 | textarea | - | - |
| `xxx_num` / `xxx_number` | 数量 | input | EQ | - |
| `xxx_amount` / `xxx_price` | 金额 | input | EQ | - |
| `xxx_time` / `xxx_date` | 时间 | datetime | BETWEEN | - |
| `status` | 状态 | radio | EQ | sys_normal_disable |
| `xxx_status` | 状态 | select | EQ | 自定义字典 |
| `xxx_type` | 分类 | select | EQ | 自定义字典 |
| `is_xxx` | 是否 | radio | EQ | sys_yes_no |
| `xxx_img` / `xxx_cover` | 图片 | imageUpload | - | - |
| `xxx_file` | 文件 | fileUpload | - | - |

#### 4.3 标准表结构模板

```sql
CREATE TABLE [表前缀]_[功能名] (
    id              BIGINT(20)   NOT NULL COMMENT '主键ID',  -- 雪花ID，不用 AUTO_INCREMENT
    tenant_id       VARCHAR(20)  DEFAULT '000000' COMMENT '租户ID',

    -- 业务字段（遵循命名规则）
    xxx_name        VARCHAR(100) NOT NULL COMMENT '名称',
    xxx_type        CHAR(1)      DEFAULT '1' COMMENT '类型',
    status          CHAR(1)      DEFAULT '0' COMMENT '状态(0正常 1停用)',

    -- 审计字段
    create_dept     BIGINT(20)   DEFAULT NULL COMMENT '创建部门',
    create_by       BIGINT(20)   DEFAULT NULL COMMENT '创建人',
    create_time     DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_by       BIGINT(20)   DEFAULT NULL COMMENT '更新人',
    update_time     DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    remark          VARCHAR(500) DEFAULT NULL COMMENT '备注',
    del_flag        CHAR(1)      DEFAULT '0' COMMENT '删除标志(0正常 1已删除)',

    PRIMARY KEY (id),
    KEY idx_tenant_id (tenant_id),
    KEY idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='xxx表';
```

**重要默认值**：
- `tenant_id`: 必须默认 `'000000'`
- `status`: 必须默认 `'0'` (正常)，原框架约定 **0=正常 1=停用**
- `del_flag`: 必须默认 `'0'` (未删除)

---

### 第五步：生成方案并确认（仅此一次确认）⭐⭐⭐⭐⭐

**输出完整方案**：

```markdown
## 代码生成方案

### 基本配置
- **功能名称**：广告管理
- **生成模式**：模式A - 直接编写代码 / 模式B - 生成配置
- **模块**：business
- **表名**：b_ad
- **Java类名**：Ad
- **包名**：org.dromara.business
- **接口路径**：/business/ad

### 菜单配置
- **上级菜单**：业务管理 (menu_id: [从第三步查询获取])
  - 如不存在会自动创建
- **菜单顺序**：[根据现有子菜单自动递增]

### 字典类型检查
| 字典类型 | 状态 | 说明 |
|---------|------|------|
| sys_normal_disable | 已存在 | 系统内置（状态） |
| b_ad_type | 需创建 | 广告分类（如：图片、文字、视频） |

### 表结构设计
\```sql
CREATE TABLE b_ad (
    ...
);
\```

### 字段配置详情
| 字段 | 类型 | 控件 | 查询方式 | 字典类型 |
|------|------|------|---------|---------|
| id | Long | input | EQ | - |
| tenant_id | String | - | - | - (框架自动处理) |
| ad_name | String | input | LIKE | - |
| ad_type | String | select | EQ | b_ad_type |
| status | String | radio | EQ | sys_normal_disable |
| create_time | Date | datetime | BETWEEN | - |

### 执行步骤
1. （如需要）创建上级菜单
2. （如需要）创建字典类型和字典数据
3. 创建数据库表
4. 插入代码生成配置（gen_table + gen_table_column）
5. 【模式A】直接编写后端代码文件（7个）+ 插入菜单SQL
6. 【模式B】输出配置报告，引导手动生成

**确认开始生成？**（回复"确认"或"开始"）
```

---

### 第六步：自动执行生成（无需确认）

用户确认后，AI 自动执行以下步骤：

#### 6.1 智能判断和创建上级菜单

**判断逻辑**：
```
system 模块   → 归属 "系统管理" (menu_id: 从第三步查询结果获取)
business 模块 → 归属 "业务管理" (从查询结果获取，如不存在则创建)
其他模块      → 归属 "[模块名]管理" (从查询结果获取，如不存在则创建)
```

**上级菜单不存在时，自动创建**：

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
-- 插入上级菜单（M 类型目录）
-- 字段顺序：menu_id, menu_name, parent_id, order_num, path, component, query_param,
--           is_frame, is_cache, menu_type, visible, status, perms, icon,
--           create_dept, create_by, create_time, update_by, update_time, remark
INSERT INTO sys_menu VALUES (
    [新menu_id], '[模块名]管理', 0, [order_num], '[module]Manage',
    NULL, '', 1, 0, 'M', '0', '0',
    '', '[icon]', 103, 1, sysdate(), NULL, NULL, '[模块名]管理目录'
);
EOF
```

#### 6.2 计算菜单顺序

```sql
-- 查询上级菜单下最大的 order_num
SELECT MAX(order_num) as max_order
FROM sys_menu
WHERE parent_id = [上级菜单ID] AND menu_type = 'C' AND del_flag = 0;

-- 新菜单顺序 = MAX(order_num) + 10
-- 如果 MAX(order_num) 为 NULL，则从 1 开始
```

#### 6.3 执行建表 SQL

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
[建表SQL]
EOF
```

输出：`表创建成功：b_ad`

#### 6.4 检查并生成字典类型（如需要）

**AI 必须检查字段使用的字典是否已存在**：

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
SELECT dict_type FROM sys_dict_type WHERE dict_type IN ('需要的字典类型列表');
EOF
```

**常见系统内置字典**（无需创建）：
```
sys_normal_disable    - 系统开关（0正常 1停用）
sys_yes_no            - 系统是否（Y是 N否）
sys_user_sex          - 用户性别
```

**字典不存在时，自动创建**：

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
-- 插入字典类型
-- 字段顺序：dict_id, tenant_id, dict_name, dict_type, create_dept, create_by, create_time, update_by, update_time, remark
INSERT INTO sys_dict_type VALUES (
    [新dict_id], '000000', '广告分类', 'b_ad_type',
    103, 1, sysdate(), NULL, NULL, '业务字典：广告分类'
);

-- 插入字典数据
-- 字段顺序：dict_code, tenant_id, dict_sort, dict_label, dict_value, dict_type,
--           css_class, list_class, is_default, create_dept, create_by, create_time, update_by, update_time, remark
INSERT INTO sys_dict_data VALUES
([新dict_code],   '000000', 1, '图片广告', '1', 'b_ad_type', '', 'primary', 'N', 103, 1, sysdate(), NULL, NULL, ''),
([新dict_code+1], '000000', 2, '文字广告', '2', 'b_ad_type', '', 'success', 'N', 103, 1, sysdate(), NULL, NULL, ''),
([新dict_code+2], '000000', 3, '视频广告', '3', 'b_ad_type', '', 'info',    'N', 103, 1, sysdate(), NULL, NULL, '');
EOF
```

输出：
```markdown
字典类型检查完成：
- sys_normal_disable: 已存在（系统内置）
- b_ad_type: 已创建（3个字典项）
```

#### 6.5 生成代码生成器配置 SQL

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
-- 表配置（gen_table，共22个字段）
INSERT INTO gen_table (
    table_id, data_name, table_name, table_comment, sub_table_name, sub_table_fk_name,
    class_name, tpl_category, package_name, module_name, business_name,
    function_name, function_author, gen_type, gen_path, options,
    create_dept, create_by, create_time, update_by, update_time, remark
) VALUES (
    [新table_id], 'master', 'b_ad', '广告表', NULL, NULL,
    'Ad', 'crud', 'org.dromara.business', 'business', 'ad',
    '广告', '系统生成', '1', '/',
    '{"parentMenuId":"[查询到的上级菜单ID]","parentMenuName":"业务管理"}',
    103, 1, sysdate(), NULL, sysdate(), '广告管理'
);

-- 列配置（gen_table_column，共23个字段）
-- ⚠️ 本框架 gen_table_column 没有 column_default 和 column_label 字段！
INSERT INTO gen_table_column (
    column_id, table_id, column_name, column_comment,
    column_type, java_type, java_field, is_pk, is_increment, is_required,
    is_insert, is_edit, is_list, is_query, query_type, html_type, dict_type,
    sort, create_dept, create_by, create_time, update_by, update_time
) VALUES
-- id 主键（雪花ID，is_increment='0'）
([新id],   [table_id], 'id',          '广告ID',   'bigint(20)',   'Long',       'id',         '1', '0', '1', NULL, '1', '1', '1', 'EQ',      'input',    '',                  1,  103, 1, sysdate(), NULL, sysdate()),
-- tenant_id（框架自动处理，配置全为0）
([新id+1], [table_id], 'tenant_id',   '租户ID',   'varchar(20)',  'String',     'tenantId',   '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'input',    '',                  2,  103, 1, sysdate(), NULL, sysdate()),
-- 业务字段
([新id+2], [table_id], 'ad_name',     '广告名称', 'varchar(100)', 'String',     'adName',     '0', '0', '1', '1',  '1', '1', '1', 'LIKE',    'input',    '',                  3,  103, 1, sysdate(), NULL, sysdate()),
([新id+3], [table_id], 'ad_type',     '广告类型', 'char(1)',      'String',     'adType',     '0', '0', '0', '1',  '1', '1', '1', 'EQ',      'select',   'b_ad_type',         4,  103, 1, sysdate(), NULL, sysdate()),
([新id+4], [table_id], 'status',      '状态',     'char(1)',      'String',     'status',     '0', '0', '0', '1',  '1', '1', '1', 'EQ',      'radio',    'sys_normal_disable', 5,  103, 1, sysdate(), NULL, sysdate()),
-- 审计字段
([新id+5], [table_id], 'create_dept', '创建部门', 'bigint(20)',   'Long',       'createDept', '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'input',    '',                  6,  103, 1, sysdate(), NULL, sysdate()),
([新id+6], [table_id], 'create_by',   '创建人',   'bigint(20)',   'Long',       'createBy',   '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'input',    '',                  7,  103, 1, sysdate(), NULL, sysdate()),
([新id+7], [table_id], 'create_time', '创建时间', 'datetime',     'Date',       'createTime', '0', '0', '0', '0',  '0', '1', '1', 'BETWEEN', 'datetime', '',                  8,  103, 1, sysdate(), NULL, sysdate()),
([新id+8], [table_id], 'update_by',   '更新人',   'bigint(20)',   'Long',       'updateBy',   '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'input',    '',                  9,  103, 1, sysdate(), NULL, sysdate()),
([新id+9], [table_id], 'update_time', '更新时间', 'datetime',     'Date',       'updateTime', '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'datetime', '',                  10, 103, 1, sysdate(), NULL, sysdate()),
([新id+10],[table_id], 'remark',      '备注',     'varchar(500)', 'String',     'remark',     '0', '0', '0', '1',  '1', '0', '0', 'EQ',      'textarea', '',                  11, 103, 1, sysdate(), NULL, sysdate()),
([新id+11],[table_id], 'del_flag',    '删除标志', 'char(1)',      'String',     'delFlag',    '0', '0', '0', '0',  '0', '0', '0', 'EQ',      'input',    '',                  12, 103, 1, sysdate(), NULL, sysdate())
;
EOF
```

输出：
```markdown
配置生成成功！
- gen_table: 1 条
- gen_table_column: 12 条
```

---

### 第七步：根据模式执行（模式分支）⭐⭐⭐⭐⭐

#### 模式A：直接编写代码（用户选择了"直接编写代码"）

**AI 直接编写所有后端代码文件，无需用户手动操作。**

##### 7A.1 读取参考代码（必须！）

> 优先查找目标模块下已有的业务代码作为参考。如果没有，使用 TestDemo 模块作为标准模板。

```bash
# 读取后端参考代码（TestDemo 模块作为标准模板）
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/domain/TestDemo.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/domain/bo/TestDemoBo.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/domain/vo/TestDemoVo.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/controller/TestDemoController.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/service/ITestDemoService.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/service/impl/TestDemoServiceImpl.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/mapper/TestDemoMapper.java
```

> 如果 TestDemo 模块不存在（demo 模块被排除），则使用 SysNotice 作为参考：
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/domain/SysNotice.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/domain/bo/SysNoticeBo.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/domain/vo/SysNoticeVo.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/controller/system/SysNoticeController.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/service/ISysNoticeService.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/service/impl/SysNoticeServiceImpl.java`
> - `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/mapper/SysNoticeMapper.java`

##### 7A.2 直接编写后端代码（7个文件）

按照参考代码风格，AI 使用 Write 工具直接创建以下文件：

**文件创建顺序**（有依赖关系，必须按顺序）：

| 序号 | 文件 | 路径 | 说明 |
|------|------|------|------|
| 1 | Entity | `domain/[Class].java` | extends TenantEntity |
| 2 | BO | `domain/bo/[Class]Bo.java` | @AutoMapper(target = [Class].class) |
| 3 | VO | `domain/vo/[Class]Vo.java` | @AutoMapper(target = [Class].class)，含 @ExcelProperty |
| 4 | Mapper | `mapper/[Class]Mapper.java` | extends BaseMapperPlus<[Class], [Class]Vo> |
| 5 | IService | `service/I[Class]Service.java` | Service 接口 |
| 6 | ServiceImpl | `service/impl/[Class]ServiceImpl.java` | buildQueryWrapper()，直接注入 Mapper |
| 7 | Controller | `controller/[Class]Controller.java` | extends BaseController |

**基路径**：`ruoyi-modules/ruoyi-[module]/src/main/java/org/dromara/[module]/`

**关键编码规范**（必须严格遵守）：

```
包名：org.dromara.[module]
Entity 继承 TenantEntity（含 tenant_id, create_dept, create_by, create_time, update_by, update_time, del_flag）
Mapper 继承 BaseMapperPlus<[Class], [Class]Vo>（2个泛型参数）
Service 不继承基类，implements I[Class]Service
ServiceImpl 使用 @RequiredArgsConstructor + private final [Class]Mapper baseMapper
查询条件在 ServiceImpl 的 buildQueryWrapper() 中构建
对象转换用 MapstructUtils.convert()
Controller 继承 BaseController
Controller 使用 @RequiredArgsConstructor + private final I[Class]Service
BO 使用 @AutoMapper(target = [Class].class, reverseConvertGenerate = false)
VO 使用 @AutoMapper(target = [Class].class)
禁止使用 @Autowired
```

**API 路径规范**：

| 操作 | HTTP方法 | 路径 | 权限标识 |
|------|---------|------|---------|
| 分页查询 | GET | `/list` | `[module]:[business]:list` |
| 获取详情 | GET | `/{id}` | `[module]:[business]:query` |
| 新增 | POST | `/` | `[module]:[business]:add` |
| 修改 | PUT | `/` | `[module]:[business]:edit` |
| 删除 | DELETE | `/{ids}` | `[module]:[business]:remove` |
| 导出 | POST | `/export` | `[module]:[business]:export` |

##### 7A.3 编写前端代码（如果 plus-ui 目录存在）

```bash
# 检查 plus-ui 是否存在
ls plus-ui/
```

**如果 plus-ui 不存在**：跳过前端代码生成，仅生成后端。

**如果 plus-ui 存在，必须生成前端 3 个文件**：

###### 7A.3.1 读取前端参考代码（必须！）

```bash
# 读取前端参考代码（notice 模块作为标准模板）
Read plus-ui/src/api/system/notice/index.ts
Read plus-ui/src/api/system/notice/types.ts
Read plus-ui/src/views/system/notice/index.vue
```

###### 7A.3.2 生成前端文件（3个）

| 序号 | 文件 | 路径 | 说明 |
|------|------|------|------|
| 1 | API 请求 | `plus-ui/src/api/[module]/[business]/index.ts` | 5 个标准 CRUD 方法 |
| 2 | 类型定义 | `plus-ui/src/api/[module]/[business]/types.ts` | VO/Query/Form 接口 |
| 3 | 页面组件 | `plus-ui/src/views/[module]/[business]/index.vue` | CRUD 页面 |

###### 7A.3.3 前端代码编写规范

**API 文件（index.ts）**：
- 导入 `request` 从 `@/utils/request`
- 导入类型从 `./types`
- 5 个方法：`listXxx`（GET /list）、`getXxx`（GET /{id}）、`addXxx`（POST）、`updateXxx`（PUT）、`delXxx`（DELETE /{id}）
- URL 必须与后端 Controller `@RequestMapping` 路径一致

**类型文件（types.ts）**：
- `XxxVO extends BaseEntity`：对应后端 VO 字段（列表展示用）
- `XxxQuery extends PageQuery`：对应后端 BO 中的查询条件字段
- `XxxForm`：对应后端 BO 中的可编辑字段，id 类型为 `number | string | undefined`
- `BaseEntity`、`PageQuery` 是全局类型，无需 import

**页面文件（index.vue）**：
- `<script setup name="Xxx" lang="ts">` 使用 Composition API
- 字典使用 `proxy?.useDict('字典类型')` + `toRefs`
- 权限指令 `v-hasPermi="['module:business:add']"`，权限标识必须与后端一致
- 搜索区域：`<el-card>` + `<el-form :inline="true">`，带动画过渡
- 工具栏：`<right-toolbar>` 组件
- 表格：`<el-table>` + `<el-table-column>`，字典列用 `<dict-tag>`
- 分页：`<pagination>` 组件绑定 pageNum/pageSize/total
- 弹窗：`<el-dialog>` + `<el-form>` 表单校验
- 日期格式化：`proxy.parseTime(date, '{y}-{m}-{d}')`

**全局已声明类型**（无需 import）：
`BaseEntity`、`PageQuery`、`PageData`、`DialogOption`、`ComponentInternalInstance`、`ElFormInstance`

##### 7A.4 插入菜单 SQL

```bash
mysql -h[host] -P[port] -u[user] -p[pass] [db] <<EOF
-- 插入 C 类型菜单（业务菜单）
-- 字段顺序：menu_id, menu_name, parent_id, order_num, path, component, query_param,
--           is_frame, is_cache, menu_type, visible, status, perms, icon,
--           create_dept, create_by, create_time, update_by, update_time, remark
INSERT INTO sys_menu VALUES (
    [新menu_id], '[功能名]管理', [上级菜单ID], [order_num], '[business]',
    '[module]/[business]/index', '', 1, 0, 'C', '0', '0',
    '[module]:[business]:list', '#', 103, 1, sysdate(), NULL, NULL, '[功能名]管理菜单'
);

-- 插入 F 类型按钮权限（查询、新增、修改、删除、导出）
INSERT INTO sys_menu VALUES
([新menu_id+1], '[功能名]查询', [C菜单ID], 1, '', '', '', 1, 0, 'F', '0', '0', '[module]:[business]:query',  '#', 103, 1, sysdate(), NULL, NULL, ''),
([新menu_id+2], '[功能名]新增', [C菜单ID], 2, '', '', '', 1, 0, 'F', '0', '0', '[module]:[business]:add',    '#', 103, 1, sysdate(), NULL, NULL, ''),
([新menu_id+3], '[功能名]修改', [C菜单ID], 3, '', '', '', 1, 0, 'F', '0', '0', '[module]:[business]:edit',   '#', 103, 1, sysdate(), NULL, NULL, ''),
([新menu_id+4], '[功能名]删除', [C菜单ID], 4, '', '', '', 1, 0, 'F', '0', '0', '[module]:[business]:remove', '#', 103, 1, sysdate(), NULL, NULL, ''),
([新menu_id+5], '[功能名]导出', [C菜单ID], 5, '', '', '', 1, 0, 'F', '0', '0', '[module]:[business]:export', '#', 103, 1, sysdate(), NULL, NULL, '');
EOF
```

**模式A 完成报告**：

```markdown
## 代码生成完成！（直接编写模式）

### 已完成
- 上级菜单已就绪：业务管理 (menu_id: [ID])
- 字典类型已就绪：b_ad_type (新建，含3个字典项)
- 数据库表创建：b_ad
- 代码生成配置保存完成（备用）
- 后端代码已直接编写（7个文件）
- 前端代码已直接编写（3个文件，仅 plus-ui 存在时）
- 菜单和权限已导入数据库

### 编写的文件
**后端**（7个）：
- Entity: ruoyi-modules/ruoyi-[module]/.../domain/[Class].java
- BO: .../domain/bo/[Class]Bo.java
- VO: .../domain/vo/[Class]Vo.java
- Mapper: .../mapper/[Class]Mapper.java
- Service: .../service/I[Class]Service.java
- ServiceImpl: .../service/impl/[Class]ServiceImpl.java
- Controller: .../controller/[Class]Controller.java

**前端**（3个，仅 plus-ui 存在时）：
- API: plus-ui/src/api/[module]/[business]/index.ts
- Types: plus-ui/src/api/[module]/[business]/types.ts
- Page: plus-ui/src/views/[module]/[business]/index.vue

### 后续操作
- **重启后端服务**使新代码生效
- 在【角色管理】中为角色分配权限
- 启动前端查看页面效果（如有 plus-ui）
- 推荐运行 `/check` 检查代码规范
```

---

#### 模式B：生成配置，手动生成（用户选择了"生成配置"）

**只输出配置完成报告，引导用户前往代码生成器界面手动生成。**

```markdown
## 配置生成完成！（配置模式）

### 已完成
- （如需要）上级菜单已就绪：业务管理 (menu_id: [查询到的上级菜单ID])
- （如需要）字典类型已就绪：b_ad_type (新建，含3个字典项)
- 数据库表创建：b_ad
- 代码生成配置保存完成
- 智能推断字段配置（12个字段）

### 字段配置详情
| 字段 | 类型 | 控件 | 查询 | 字典 |
|------|------|------|------|------|
| ad_name | String | input | LIKE | - |
| ad_type | String | select | EQ | b_ad_type |
| status | String | radio | EQ | sys_normal_disable |
| create_time | Date | datetime | BETWEEN | - |

---

## 下一步：前往代码生成器生成代码

1. **登录系统后台**
2. **导航**：系统工具 → 代码生成
3. **查找表**：找到 `b_ad` 表
4. **（可选）编辑**：微调字段配置
5. **生成代码**：点击【生成代码】按钮
6. **重启服务**：代码生成后需重启后端服务

### 生成后的文件结构

\```
[对应模块目录]/
├── controller/AdController.java
├── domain/Ad.java
├── domain/bo/AdBo.java
├── domain/vo/AdVo.java
├── mapper/AdMapper.java
├── service/IAdService.java
└── service/impl/AdServiceImpl.java
\```

### 说明
- 所有字段已智能推断配置，无需手动调整
- 上级菜单和顺序已自动配置完成
- 生成代码后需重启后端服务
- 首次使用记得在【角色管理】中分配权限
```

---

### 第八步：联动推荐（开发完成后）

#### 8.1 推荐代码规范检查

```markdown
推荐：运行 `/check` 检查生成的代码是否符合项目规范
```

#### 8.2 询问任务跟踪关联（必须询问用户，不能自动执行！）

输出提示：
```markdown
是否需要创建/更新任务跟踪文档？

1. **创建新任务** — 在 docs/tasks/active/ 创建此功能的任务跟踪文档
2. **更新现有任务** — 将此功能标记为已完成步骤（需要先有活跃任务）
3. **不需要** — 跳过任务关联
```

**处理逻辑**：

| 用户选择 | 操作 |
|---------|------|
| 创建新任务 | 调用 task-tracker 技能，创建任务文档 |
| 更新现有任务 | 读取活跃任务文档，标记相关步骤完成 |
| 不需要 | 结束，不做任何操作 |

**强制规则**：
- 禁止自动创建任务文档
- 禁止未经确认就修改任务文档
- 必须明确询问用户意愿
- 用户说"不需要"就立即结束

---

## 上级菜单映射规则

AI 必须根据模块自动判断归属的上级菜单：

| 模块名 | 上级菜单 | menu_id | 说明 |
|-------|---------|---------|------|
| **system** | 系统管理 | 从第三步查询获取 | 系统管理类功能 |
| **business** | 业务管理 | 查询获取（如不存在则自动创建） | 业务功能 |
| **其他** | [模块名]管理 | 自动创建 | 自定义模块 |

### 上级菜单创建模板

当上级菜单不存在时，使用以下模板创建：

```sql
-- sys_menu 字段顺序（共20个字段）：
-- menu_id, menu_name, parent_id, order_num, path, component, query_param,
-- is_frame, is_cache, menu_type, visible, status, perms, icon,
-- create_dept, create_by, create_time, update_by, update_time, remark
INSERT INTO sys_menu VALUES (
    [新menu_id],         -- 自增ID
    '[模块名]管理',       -- 如：业务管理
    0,                   -- parent_id = 0 (顶级菜单)
    [order_num],         -- 排序号
    '[module]Manage',    -- 如：businessManage
    NULL,                -- component 为空（M类型目录无组件）
    '',                  -- query_param 为空
    1,                   -- is_frame = 1 (非外链)
    0,                   -- is_cache = 0 (缓存)
    'M',                 -- menu_type = M (目录)
    '0',                 -- visible = '0' (显示)
    '0',                 -- status = '0' (正常)
    '',                  -- perms 为空
    '[icon]',            -- 图标（小写模块名）
    103,                 -- create_dept
    1,                   -- create_by
    sysdate(),           -- create_time
    NULL,                -- update_by
    NULL,                -- update_time
    '[模块名]管理目录'    -- remark
);
```

### 菜单顺序计算规则

```sql
-- 查询上级菜单下最大的 order_num
SELECT MAX(order_num) as max_order
FROM sys_menu
WHERE parent_id = [上级菜单ID] AND menu_type = 'C' AND del_flag = 0;

-- 新菜单顺序 = MAX(order_num) + 10
-- 如果 MAX(order_num) 为 NULL，则从 1 开始
```

---

## 字典类型命名规范

### 系统内置字典（无需创建）

| 字典类型 | 字典名称 | 用途 |
|---------|---------|------|
| `sys_normal_disable` | 系统开关 | status 字段（0正常 1停用） |
| `sys_yes_no` | 系统是否 | is_xxx 字段（Y是 N否） |
| `sys_user_sex` | 用户性别 | gender 字段 |

### 业务字典命名规则

**格式**: `[表前缀]_[业务对象]_[字段含义]`

**示例**：

| 模块 | 字段 | 字典类型 | 字典名称 | 选项示例 |
|------|------|---------|---------|---------|
| business | ad_type | `b_ad_type` | 广告分类 | 1-图片, 2-文字, 3-视频 |
| business | order_status | `b_order_status` | 订单状态 | 1-待付款, 2-已付款, 3-已完成 |

---

## 树表代码生成指南

### 树表场景识别

用户提到"分类"、"部门"、"层级"、"树形"等关键词时，使用树表模板。

### 树表 gen_table 配置

```sql
INSERT INTO gen_table (..., tpl_category, ..., options, ...)
VALUES (..., 'tree', ...,
    '{"treeCode":"id","treeParentCode":"parentId","treeName":"categoryName","parentMenuId":"[上级菜单ID]","parentMenuName":"[上级菜单名]"}',
...);
```

### 树表 options 必填字段

| 字段 | 说明 | 示例 | 重要提示 |
|------|------|------|---------|
| `treeCode` | 树节点ID字段 | `id` | 必须是 Java 驼峰命名 |
| `treeParentCode` | 父节点ID字段 | `parentId` | 必须是 Java 驼峰命名（不是 `parent_id`） |
| `treeName` | 节点显示名称字段 | `categoryName` | 必须是 Java 驼峰命名 |

### 树表必须有自关联的父ID字段

```sql
parent_id BIGINT(20) DEFAULT 0 COMMENT '父分类ID',
```

---

## 主子表代码生成指南

### 主子表场景识别

用户提到"订单明细"、"一个xxx包含多个xxx"等一对多关系时，使用主子表模板。

### 配置要点

| 配置项 | 主表 | 子表 |
|-------|------|------|
| `tpl_category` | `sub` | `crud` |
| `sub_table_name` | 子表名（如 `b_order_item`） | NULL |
| `sub_table_fk_name` | 外键字段（如 `order_id`） | NULL |

### 生成顺序

1. **先生成子表**（crud 模式）
2. **再生成主表**（sub 模式 + 关联配置）

---

## AI 强制执行规则

### 流程控制
1. **第一步必须询问生成模式（直接编写代码 / 生成配置）**
2. **仅在第五步确认一次，其他步骤自动执行**
3. **第二步必须检查功能是否存在**
4. 禁止多次询问用户确认
5. 禁止参考其他框架

### 模式分支规则
6. **模式A**：第七步读取参考代码 → 直接编写全部后端文件 → 插入菜单SQL
7. **模式B**：第七步只输出配置完成报告，引导用户前往代码生成器手动生成
8. **模式A 必须先读参考代码（TestDemo 或 SysNotice），严格按照相同风格编写**
9. 禁止在模式A中跳过读取参考代码直接编写（必须先 Read 再 Write）

### 代码规范
10. **包名必须是 `org.dromara.*`**（禁止 `com.ruoyi.*`、`plus.ruoyi.*`）
11. **三层架构**：Controller → Service → Mapper（禁止创建 DAO 层！）
12. **ServiceImpl 直接注入 Mapper**（`private final XxxMapper baseMapper`）
13. **buildQueryWrapper() 在 ServiceImpl 中构建**（不是 DAO 层）
14. **对象转换用 `MapstructUtils.convert()`**（禁止 BeanUtils）
15. **BO 用 `@AutoMapper`（单数）**（禁止 `@AutoMappers` 复数）
16. **依赖注入用 `@RequiredArgsConstructor` + `private final`**（禁止 @Autowired）
17. 必须遵循智能字段命名规则
18. 必须使用标准 RESTful API 路径

### 默认值设置
19. **tenant_id 默认值必须是 `'000000'`**
20. **status 默认值必须是 `'0'`（正常）**— 本框架 0=正常 1=停用
21. **del_flag 默认值必须是 `'0'`（未删除）**

### 菜单配置
22. **必须从第三步查询结果动态获取上级菜单ID**（禁止硬编码）
23. **必须自动计算菜单顺序**（MAX + 10）
24. **sys_menu 共20个字段**，包含 query_param 字段
25. **is_frame=1(非外链), is_cache=0(缓存), visible='0'(显示), status='0'(正常)**
26. **F 按钮权限命名**：query/add/edit/remove/export（不是 list/add/update/delete/export）

### 字典配置
27. **必须检查字典类型是否已存在**
28. **系统内置字典无需创建**（sys_normal_disable、sys_yes_no 等）
29. **业务字典不存在时自动创建**（字典类型 + 字典数据）
30. **字典数据必须包含合理的选项**（至少 2-3 个）
31. **sys_dict_data 主键字段是 `dict_code`**（不是 dict_data_id）

### 代码生成配置
32. **gen_type 设为 '1'（自定义路径）**
33. **gen_path 设为 '/'**
34. **gen_table_column 没有 column_default 和 column_label 字段**，禁止在 INSERT 中包含
35. **options 支持**：parentMenuId、parentMenuName（树表额外支持 treeCode、treeParentCode、treeName）

### 租户字段特殊规则
36. **tenant_id 的所有权限必须设为 '0'**：is_insert=0, is_edit=0, is_list=0, is_query=0
37. 原因：租户ID由 TenantEntity 自动注入，框架自动处理租户隔离

### 数据库连接
38. **数据库连接信息必须从 application-dev.yml 动态读取**
39. 禁止硬编码数据库名

### 执行规则
40. **自动执行所有 SQL，无需用户操作**
41. **雪花ID**：is_increment 必须为 '0'，禁止使用 AUTO_INCREMENT

### 任务跟踪关联
42. **开发前自动检查 docs/tasks/active/ 中是否有相关活跃任务**
43. 禁止自动创建/修改任务跟踪文档
44. **开发完成后必须询问用户是否需要创建/更新任务文档**
45. 用户拒绝则立即结束，不再追问

### 联动推荐
46. **开发完成后推荐运行 `/check` 检查代码规范**

---

## 参考代码位置

| 类型 | 位置 |
|------|------|
| Controller 示例 | `ruoyi-modules/ruoyi-demo/.../controller/TestDemoController.java` |
| Service 示例 | `ruoyi-modules/ruoyi-demo/.../service/impl/TestDemoServiceImpl.java` |
| Entity 示例 | `ruoyi-modules/ruoyi-demo/.../domain/TestDemo.java` |
| BO 示例 | `ruoyi-modules/ruoyi-demo/.../domain/bo/TestDemoBo.java` |
| VO 示例 | `ruoyi-modules/ruoyi-demo/.../domain/vo/TestDemoVo.java` |
| Mapper 示例 | `ruoyi-modules/ruoyi-demo/.../mapper/TestDemoMapper.java` |
| 备用参考 | `ruoyi-modules/ruoyi-system/.../controller/system/SysNoticeController.java` |
| SQL 建表参考 | `script/sql/ry_vue_5.X.sql` |
