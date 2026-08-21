---
name: crud
description: |
  当需要基于已存在的数据库表快速生成 CRUD 代码时自动使用此 Skill。

  触发场景：
  - 数据库表已存在，需要生成后端代码
  - 需要标准的增删改查功能
  - 快速原型开发
  - 需要生成树形结构的 CRUD

  触发词：生成CRUD、crud、代码生成、快速生成、生成代码、CRUD生成、表生成代码
---

# /crud - 快速生成 CRUD 代码

作为 CRUD 代码生成助手，基于已存在的数据库表快速生成标准后端 CRUD 代码。

## 🎯 适用场景

### ✅ 适合使用 `/crud` 的情况

- ✅ **数据库表已存在** - 表结构已设计完毕
- ✅ **只需标准 CRUD** - 增删改查、导入导出等标准功能
- ✅ **无复杂业务逻辑** - 没有特殊的业务规则
- ✅ **快速原型开发** - 需要快速搭建基础功能
- ✅ **树形结构数据** - 部门、分类等层级数据（tree 模板）

### ❌ 不适合使用 `/crud` 的情况

- ❌ **表结构尚未设计** → 请使用 `/dev` 命令
- ❌ **需要复杂业务逻辑** → 请使用 `/dev` 命令后手动增强
- ❌ **需要特殊的查询条件** → 请使用 `/dev` 命令后手动修改

### 📋 支持的模板类型

| 模板类型 | 适用场景 | 特点 |
|---------|---------|------|
| **crud** | 普通表 | 标准增删改查、分页列表 |
| **tree** | 树形表 | 父子层级、展开折叠、无分页 |

---

## 📋 执行流程

### 第一步：连接数据库并查看表结构

#### 1.1 询问用户

```
请提供表名：
（如：sys_notice, demo_xxx）

💡 数据库连接信息将从 application-dev.yml 自动读取
```

#### 1.2 读取数据库配置（强制执行）⭐⭐⭐⭐⭐

```bash
# 读取开发环境配置文件
Read ruoyi-admin/src/main/resources/application-dev.yml
```

从配置文件中提取以下信息（注意环境变量和默认值）：

**配置文件格式示例**（动态多数据源）：
```yaml
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    dynamic:
      primary: master
      datasource:
        master:
          type: ${spring.datasource.type}
          driverClassName: com.mysql.cj.jdbc.Driver
          url: jdbc:mysql://localhost:3306/ry-vue?useUnicode=true&characterEncoding=utf8&...
          username: root
          password: root
```

**解析规则**：
- 配置路径：`spring.datasource.dynamic.datasource.master`
- 主机和端口：从 `url` 中的 `jdbc:mysql://[主机]:[端口]/[数据库名]` 解析
- 用户名：`username` 字段
- 密码：`password` 字段
- 如果使用环境变量格式 `${ENV:默认值}`，取默认值部分

**⚠️ 注意**：不要输出数据库连接信息给用户确认，直接使用读取到的配置连接数据库

#### 1.3 连接数据库并查看表结构

```bash
# 使用解析出的配置连接数据库
mysql -h [主机] -P [端口] -u [用户名] -p[密码] [数据库名]

# 查看表结构
SHOW CREATE TABLE [表名];

# 查看字段详情
DESC [表名];

# 查询最大菜单ID
SELECT MAX(menu_id) FROM sys_menu;

# 查询最大表ID（用于生成配置）
SELECT MAX(table_id) FROM gen_table;

# 查询现有字典类型（避免重复创建）
SELECT dict_type FROM sys_dict_type WHERE del_flag = '0';
```

#### 1.4 字段类型映射规则

| 数据库类型 | Java类型 | 说明 |
|-----------|---------|------|
| BIGINT(20), BIGINT | Long | 长整数 |
| INT(11), INT | Integer | 整数 |
| VARCHAR(n), CHAR(n) | String | 字符串 |
| TEXT, LONGTEXT | String | 长文本 |
| DATETIME, TIMESTAMP | Date | 日期时间 |
| DECIMAL(m,n) | BigDecimal | 高精度数值 |
| TINYINT(1), CHAR(1) | String | 状态字段（0/1）|

#### 1.5 输出表结构分析

```markdown
## 📊 表结构分析

**表名**：[表名]
**注释**：[表注释]

**字段列表**：
| 字段名 | 类型 | 是否必填 | 默认值 | 注释 |
|--------|------|---------|--------|------|
| id | BIGINT(20) | 是 | - | 主键ID |
| tenant_id | VARCHAR(20) | 否 | '000000' | 租户ID |
| xxx_name | VARCHAR(100) | 是 | - | 名称 |
| status | CHAR(1) | 否 | '1' | 状态 |
| create_dept | BIGINT(20) | 否 | NULL | 创建部门 |
| create_by | BIGINT(20) | 否 | NULL | 创建人 |
| create_time | DATETIME | 否 | CURRENT_TIMESTAMP | 创建时间 |
| update_by | BIGINT(20) | 否 | NULL | 修改人 |
| update_time | DATETIME | 否 | CURRENT_TIMESTAMP | 更新时间 |
| del_flag | CHAR(1) | 否 | '0' | 删除标志 |

**审计字段**：✅ 完整（包含 create_dept, create_by, create_time, update_by, update_time）
**逻辑删除**：✅ 已配置（del_flag）
**租户支持**：✅ 已支持（tenant_id）

---

### 提取功能名称

根据表名 `sys_notice` 提取功能名称：
- 中文名：公告
- 英文名：Notice
- 类名前缀：Notice
- 接口路径：/system/notice

确认功能名称，或自定义修改？
```

---

### 第 1.6 步：选择模板类型（新增）⭐⭐⭐⭐⭐

根据表结构特征，询问用户选择模板类型：

**自动检测规则**：
- 表中包含 `parent_id` 字段且 `parent_id` 为 BIGINT → `tree` 模板
- 其他情况 → `crud` 模板

**用户选择** (如需覆盖自动检测)：
```
## 🎯 请选择模板类型

根据您的表结构，请选择合适的模板类型：

1. **crud** - 普通表（默认）
   适用于：标准增删改查，无层级关系
   示例：通知公告、用户反馈等

2. **tree** - 树表
   适用于：有父子层级关系的数据（如部门、分类、菜单）
   特征：表中包含 parent_id 或类似的父级字段

请输入模板类型（crud/tree）：
```

#### 1.6.1 树表自动检测

如果表结构包含以下字段，自动提示可能是树表：
- `parent_id` / `pid` / `parent` - 父级ID
- `order_num` / `sort` - 排序字段

```
💡 检测到您的表可能是树表结构：
- 发现父级字段：parent_id
- 发现排序字段：order_num

是否使用树表模板？(Y/n)
```

#### 1.6.2 树表配置

当用户选择 tree 模板时，询问以下配置：

```
## 🌳 树表配置

请确认或修改以下配置：

1. **树编码字段**（Java 字段名）
   用于构建树结构的主键字段
   默认检测：id

2. **树父编码字段**（Java 驼峰字段名）
   父节点的关联字段
   检测到：parentId

3. **树名称字段**（Java 驼峰字段名）
   在树节点上显示的名称字段
   检测到：deptName

确认配置？(Y/n)
```

---

### 第二步：生成菜单 SQL

#### 2.1 根据表名前缀确定模块名

| 表名前缀 | 模块名 | 权限标识符格式 |
|---------|--------|--------------|
| `sys_` | `system` | `system:[功能名]:[操作]` |
| `demo_` | `demo` | `demo:[功能名]:[操作]` |
| 其他 | 询问用户 | `[模块名]:[功能名]:[操作]` |

#### 2.2 询问菜单信息

```
请提供菜单配置信息：

1. **父菜单ID**：（默认：1 - 系统管理）
2. **排序值**：（默认：10）
3. **菜单图标**：（默认：'guide'）
4. **菜单ID起始值**：当前最大ID + 10
```

#### 2.3 生成菜单 SQL

生成完整的菜单 SQL（6个权限：查看、查询、新增、修改、删除、导出）。

---

### 第三步：生成后端代码

#### 3.1 学习现有代码（强制执行）

```bash
# 必须先阅读 demo 模块代码作为参考（标准 CRUD 写法）
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/controller/TestDemoController.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/service/impl/TestDemoServiceImpl.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/domain/bo/TestDemoBo.java
Read ruoyi-modules/ruoyi-demo/src/main/java/org/dromara/demo/domain/vo/TestDemoVo.java
```

#### 3.2 生成代码顺序（三层架构）

按照以下顺序生成：

1. **Entity** - 继承 TenantEntity，字段从表结构映射
2. **BO** - 使用 @AutoMapper 注解
3. **VO** - 含 Excel 导出注解
4. **Mapper** - 继承 BaseMapperPlus
5. **Service 接口** - 标准 CRUD 方法声明
6. **ServiceImpl** - 业务逻辑实现，包含 buildQueryWrapper
7. **Controller** - 标准接口 + 导入导出

#### 3.3 字段类型映射规则

| 数据库类型 | Java类型 |
|-----------|---------|
| BIGINT(20) | Long |
| VARCHAR/CHAR | String |
| TEXT | String |
| DATETIME | Date |
| DECIMAL | BigDecimal |
| INT | Integer |

#### 3.4 查询条件生成规则⭐⭐⭐⭐⭐

| 字段名后缀 | 查询方式 | Java类型 | 说明 |
|-----------|---------|---------|------|
| `xxx_name`、`xxx_title` | LIKE | String | 名称/标题字段（模糊搜索） |
| `xxx_content`、`remark` | LIKE | String | 长文本/备注字段（模糊搜索） |
| `id`、`code` | EQ | Long/String | ID/编码字段（精确匹配） |
| `status` | EQ | String | 标准状态字段，绑定 `sys_enable_status` |
| `xxx_status` | EQ | String | 业务状态字段，绑定自定义字典 `xxx_status` |
| `xxx_type` | EQ | String | 分类/类型字段，绑定自定义字典 `xxx_type` |
| `is_xxx` | EQ | String | 布尔标志字段，绑定 `sys_boolean_flag` |
| `xxx_num`、`xxx_count`、`xxx_quantity` | EQ | Integer | 数量/计数字段 |
| `xxx_amount`、`xxx_price`、`xxx_total` | EQ | BigDecimal | 金额/价格/合计字段 |
| `xxx_rate`、`xxx_percentage` | EQ | BigDecimal | 比率/百分比字段 |
| `xxx_time`、`xxx_date` | BETWEEN | Date | 日期/时间范围查询 |
| `begin_xxx_time`、`end_xxx_time` | BETWEEN | Date | 时间段查询（分离字段） |

#### 3.5 字典类型配置（可选）

如果字段需要使用字典下拉，需要检查并配置字典类型：

```
## 🏷️ 字典类型检查

检测到以下字段可能需要配置字典：

| 字段 | 推荐字典类型 | 是否已存在 |
|------|------------|----------|
| status | sys_enable_status | ✅ 存在 |
| notice_type | （需自定义） | ❌ 不存在 |

对于不存在的字典，是否需要创建？(Y/n)
```

---

### 第四步：编写前端代码（如果 plus-ui 目录存在）

```bash
# 检查 plus-ui 是否存在
ls plus-ui/
```

**如果 plus-ui 存在**：
1. 在 `plus-ui/src/views/` 下找一个已有的 CRUD 页面作为模板（推荐 `system/notice/index.vue`）
2. 读取参考代码（至少读取一个完整的 .vue 页面 + 对应的 API 文件 + 类型定义文件）
3. 按照相同风格编写前端文件

**需要生成的前端文件（3个）**：

| 文件 | 路径 | 参考 |
|------|------|------|
| API 请求 | `plus-ui/src/api/[模块]/[功能]/index.ts` | `plus-ui/src/api/system/notice/index.ts` |
| 类型定义 | `plus-ui/src/api/[模块]/[功能]/types.ts` | `plus-ui/src/api/system/notice/types.ts` |
| 页面组件 | `plus-ui/src/views/[模块]/[功能]/index.vue` | `plus-ui/src/views/system/notice/index.vue` |

**API 文件规范**（`index.ts`）：
```typescript
import request from '@/utils/request';
import { XxxVO, XxxQuery, XxxForm } from './types';
import { AxiosPromise } from 'axios';

// 查询列表
export function listXxx(query: XxxQuery): AxiosPromise<XxxVO[]> {
  return request({ url: '/[模块]/[功能]/list', method: 'get', params: query });
}
// 查询详细
export function getXxx(id: string | number): AxiosPromise<XxxVO> {
  return request({ url: '/[模块]/[功能]/' + id, method: 'get' });
}
// 新增
export function addXxx(data: XxxForm) {
  return request({ url: '/[模块]/[功能]', method: 'post', data: data });
}
// 修改
export function updateXxx(data: XxxForm) {
  return request({ url: '/[模块]/[功能]', method: 'put', data: data });
}
// 删除
export function delXxx(id: string | number | Array<string | number>) {
  return request({ url: '/[模块]/[功能]/' + id, method: 'delete' });
}
```

**类型定义规范**（`types.ts`）：
```typescript
export interface XxxVO extends BaseEntity {
  id: number;
  // 业务字段...
  status: string;
}
export interface XxxQuery extends PageQuery {
  // 查询条件字段...
  status: string;
}
export interface XxxForm {
  id: number | string | undefined;
  // 表单字段...
  status: string;
}
```

**页面组件规范**（`index.vue`）：

页面结构必须包含以下 4 个区域：

```
搜索区域（el-card + el-form :inline）
  ↓
工具栏（el-row + right-toolbar）+ 表格（el-table）+ 分页（pagination）
  ↓
弹窗（el-dialog + el-form 表单校验）
```

关键写法规范：
- `<script setup name="Xxx" lang="ts">` — Composition API
- 字典：`const { sys_normal_disable } = toRefs<any>(proxy?.useDict('sys_normal_disable'));`
- 权限：`v-hasPermi="['模块:功能:add']"` — 与后端 `@SaCheckPermission` 一致
- 列表数据：`const xxxList = ref<XxxVO[]>([]);`
- 表单数据：`const data = reactive<PageData<XxxForm, XxxQuery>>({...})`
- 日期格式：`proxy.parseTime(date, '{y}-{m}-{d}')`
- 字典列：`<dict-tag :options="字典变量" :value="scope.row.字段" />`
- 全局类型无需 import：`BaseEntity`、`PageQuery`、`PageData`、`DialogOption`、`ComponentInternalInstance`、`ElFormInstance`

> 详细的完整页面模板请参考 `crud-development` 技能中的"9.3 CRUD 页面模板"章节。

**如果 plus-ui 不存在**：跳过前端代码生成，仅生成后端。

---

### 第五步：输出代码清单

```markdown
✅ CRUD 代码生成完成！

## 已生成文件清单

### 菜单 SQL
- ✅ 菜单 SQL（需手动执行）

### 后端代码 (7个文件)
- ✅ domain/Xxx.java (Entity)
- ✅ domain/bo/XxxBo.java (BO)
- ✅ domain/vo/XxxVo.java (VO)
- ✅ mapper/XxxMapper.java (Mapper)
- ✅ service/IXxxService.java (Service接口)
- ✅ service/impl/XxxServiceImpl.java (Service实现)
- ✅ controller/XxxController.java (Controller)

### 前端代码 (3个文件，仅 plus-ui 存在时)
- ✅ plus-ui/src/api/[模块]/[功能]/index.ts (API请求)
- ✅ plus-ui/src/api/[模块]/[功能]/types.ts (类型定义)
- ✅ plus-ui/src/views/[模块]/[功能]/index.vue (CRUD页面)

---

## 下一步操作

1. 执行菜单 SQL
2. 重启后端服务
3. 测试接口（Swagger: http://localhost:8080/doc.html）
4. 配置菜单权限
5. 启动前端查看页面效果（如有 plus-ui）
```

---

## 代码模板

### Entity 模板

```java
package org.dromara.[模块].domain;

import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import org.dromara.common.tenant.core.TenantEntity;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
@TableName("[表名]")
public class [实体名] extends TenantEntity {

    @TableId(value = "id")
    private Long id;

    // 业务字段...
}
```

### BO 模板

```java
package org.dromara.[模块].domain.bo;

import org.dromara.[模块].domain.[实体名];
import org.dromara.common.core.validate.AddGroup;
import org.dromara.common.core.validate.EditGroup;
import org.dromara.common.mybatis.core.domain.BaseEntity;
import io.github.linpeilie.annotations.AutoMapper;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
@AutoMapper(target = [实体名].class, reverseConvertGenerate = false)
public class [实体名]Bo extends BaseEntity {

    /**
     * 主键（编辑时必填）
     */
    @NotNull(message = "主键不能为空", groups = {EditGroup.class})
    private Long id;

    // 业务字段示例:
    // @NotBlank(message = "名称不能为空", groups = {AddGroup.class, EditGroup.class})
    // private String xxxName;
}
```

### VO 模板

```java
package org.dromara.[模块].domain.vo;

import org.dromara.[模块].domain.[实体名];
import cn.idev.excel.annotation.ExcelIgnoreUnannotated;
import cn.idev.excel.annotation.ExcelProperty;
import io.github.linpeilie.annotations.AutoMapper;
import lombok.Data;

import java.io.Serial;
import java.io.Serializable;

@Data
@ExcelIgnoreUnannotated
@AutoMapper(target = [实体名].class)
public class [实体名]Vo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    @ExcelProperty(value = "主键")
    private Long id;

    // 业务字段...
}
```

### Mapper 模板

```java
package org.dromara.[模块].mapper;

import org.dromara.[模块].domain.[实体名];
import org.dromara.[模块].domain.vo.[实体名]Vo;
import org.dromara.common.mybatis.core.mapper.BaseMapperPlus;

public interface [实体名]Mapper extends BaseMapperPlus<[实体名], [实体名]Vo> {
}
```

### Service 接口模板

```java
package org.dromara.[模块].service;

import org.dromara.[模块].domain.bo.[实体名]Bo;
import org.dromara.[模块].domain.vo.[实体名]Vo;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import java.util.Collection;
import java.util.List;

public interface I[实体名]Service {

    /**
     * 查询
     */
    [实体名]Vo queryById(Long id);

    /**
     * 分页查询
     */
    TableDataInfo<[实体名]Vo> queryPageList([实体名]Bo bo, PageQuery pageQuery);

    /**
     * 查询列表
     */
    List<[实体名]Vo> queryList([实体名]Bo bo);

    /**
     * 新增
     */
    Boolean insertByBo([实体名]Bo bo);

    /**
     * 修改
     */
    Boolean updateByBo([实体名]Bo bo);

    /**
     * 校验并批量删除
     */
    Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid);
}
```

### ServiceImpl 模板

```java
package org.dromara.[模块].service.impl;

import cn.hutool.core.util.ObjectUtil;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import lombok.RequiredArgsConstructor;
import org.dromara.[模块].domain.[实体名];
import org.dromara.[模块].domain.bo.[实体名]Bo;
import org.dromara.[模块].domain.vo.[实体名]Vo;
import org.dromara.[模块].mapper.[实体名]Mapper;
import org.dromara.[模块].service.I[实体名]Service;
import org.dromara.common.core.utils.MapstructUtils;
import org.dromara.common.core.utils.StringUtils;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.springframework.stereotype.Service;
import java.util.Collection;
import java.util.List;
import java.util.Map;

@RequiredArgsConstructor
@Service
public class [实体名]ServiceImpl implements I[实体名]Service {

    private final [实体名]Mapper baseMapper;

    @Override
    public [实体名]Vo queryById(Long id) {
        return baseMapper.selectVoById(id);
    }

    @Override
    public TableDataInfo<[实体名]Vo> queryPageList([实体名]Bo bo, PageQuery pageQuery) {
        LambdaQueryWrapper<[实体名]> lqw = buildQueryWrapper(bo);
        Page<[实体名]Vo> result = baseMapper.selectVoPage(pageQuery.build(), lqw);
        return TableDataInfo.build(result);
    }

    @Override
    public List<[实体名]Vo> queryList([实体名]Bo bo) {
        LambdaQueryWrapper<[实体名]> lqw = buildQueryWrapper(bo);
        return baseMapper.selectVoList(lqw);
    }

    private LambdaQueryWrapper<[实体名]> buildQueryWrapper([实体名]Bo bo) {
        Map<String, Object> params = bo.getParams();
        LambdaQueryWrapper<[实体名]> lqw = Wrappers.lambdaQuery();
        // 根据字段类型添加查询条件
        // lqw.like(StringUtils.isNotBlank(bo.getXxxName()), [实体名]::getXxxName, bo.getXxxName());
        // lqw.eq(StringUtils.isNotBlank(bo.getStatus()), [实体名]::getStatus, bo.getStatus());
        return lqw;
    }

    @Override
    public Boolean insertByBo([实体名]Bo bo) {
        [实体名] add = MapstructUtils.convert(bo, [实体名].class);
        validEntityBeforeSave(add);
        boolean flag = baseMapper.insert(add) > 0;
        if (flag) {
            bo.setId(add.getId());
        }
        return flag;
    }

    @Override
    public Boolean updateByBo([实体名]Bo bo) {
        [实体名] update = MapstructUtils.convert(bo, [实体名].class);
        validEntityBeforeSave(update);
        return baseMapper.updateById(update) > 0;
    }

    /**
     * 保存前的数据校验
     */
    private void validEntityBeforeSave([实体名] entity) {
        //TODO 做一些数据校验,如唯一约束
    }

    @Override
    public Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid) {
        if (isValid) {
            //TODO 做一些业务上的校验,判断是否需要校验
        }
        return baseMapper.deleteByIds(ids) > 0;
    }
}
```

### Controller 模板

```java
package org.dromara.[模块].controller;

import cn.dev33.satoken.annotation.SaCheckPermission;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import lombok.RequiredArgsConstructor;
import org.dromara.common.core.domain.R;
import org.dromara.common.core.validate.AddGroup;
import org.dromara.common.core.validate.EditGroup;
import org.dromara.common.excel.utils.ExcelUtil;
import org.dromara.common.log.annotation.Log;
import org.dromara.common.log.enums.BusinessType;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.web.core.BaseController;
import org.dromara.[模块].domain.bo.[实体名]Bo;
import org.dromara.[模块].domain.vo.[实体名]Vo;
import org.dromara.[模块].service.I[实体名]Service;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import java.util.Arrays;
import java.util.List;

@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/[模块路径]/[功能路径]")
public class [实体名]Controller extends BaseController {

    private final I[实体名]Service service;

    /**
     * 查询列表
     */
    @SaCheckPermission("[权限前缀]:list")
    @GetMapping("/list")
    public TableDataInfo<[实体名]Vo> list([实体名]Bo bo, PageQuery pageQuery) {
        return service.queryPageList(bo, pageQuery);
    }

    /**
     * 导出列表
     */
    @SaCheckPermission("[权限前缀]:export")
    @Log(title = "[功能名称]", businessType = BusinessType.EXPORT)
    @PostMapping("/export")
    public void export([实体名]Bo bo, HttpServletResponse response) {
        List<[实体名]Vo> list = service.queryList(bo);
        ExcelUtil.exportExcel(list, "[功能名称]", [实体名]Vo.class, response);
    }

    /**
     * 获取详细信息
     */
    @SaCheckPermission("[权限前缀]:query")
    @GetMapping("/{id}")
    public R<[实体名]Vo> getInfo(@NotNull(message = "主键不能为空") @PathVariable Long id) {
        return R.ok(service.queryById(id));
    }

    /**
     * 新增
     */
    @SaCheckPermission("[权限前缀]:add")
    @Log(title = "[功能名称]", businessType = BusinessType.INSERT)
    @PostMapping
    public R<Void> add(@Validated(AddGroup.class) @RequestBody [实体名]Bo bo) {
        return toAjax(service.insertByBo(bo));
    }

    /**
     * 修改
     */
    @SaCheckPermission("[权限前缀]:edit")
    @Log(title = "[功能名称]", businessType = BusinessType.UPDATE)
    @PutMapping
    public R<Void> edit(@Validated(EditGroup.class) @RequestBody [实体名]Bo bo) {
        return toAjax(service.updateByBo(bo));
    }

    /**
     * 删除
     */
    @SaCheckPermission("[权限前缀]:remove")
    @Log(title = "[功能名称]", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ids}")
    public R<Void> remove(@NotEmpty(message = "主键不能为空") @PathVariable Long[] ids) {
        return toAjax(service.deleteWithValidByIds(Arrays.asList(ids), true));
    }
}
```

---

## 与 `/dev` 命令的区别

| 对比项 | `/crud` | `/dev` |
|--------|---------|--------|
| **适用场景** | 表已存在 | 从零开始 |
| **表结构设计** | ❌ 跳过 | ✅ 引导设计 |
| **菜单 SQL** | ✅ 生成 | ✅ 生成 |
| **代码生成** | ✅ 自动 | ✅ 自动 |
| **树表支持** | ✅ 支持 | ✅ 支持 |
| **执行速度** | ⚡ 快速 | 🐢 较慢 |

**建议**：
- 快速原型：使用 `/crud`
- 正式开发：使用 `/dev`
- 表已存在：使用 `/crud`

<!-- 抓蛙师 -->
