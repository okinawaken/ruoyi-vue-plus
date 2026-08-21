---
name: crud-development
description: |
  后端 CRUD 开发规范。基于 RuoYi-Vue-Plus 三层架构（Controller → Service → Mapper），无独立 DAO 层。

  触发场景：
  - 新建业务模块的 CRUD 功能
  - 创建 Entity、BO、VO、Service、Mapper、Controller
  - 分页查询、新增、修改、删除、导出
  - 查询条件构建（buildQueryWrapper）

  触发词：CRUD、增删改查、新建模块、Entity、BO、VO、Service、Mapper、Controller、分页查询、buildQueryWrapper、@AutoMapper、BaseMapperPlus、TenantEntity

  注意：
  - 本项目是三层架构，Service 直接注入 Mapper，无 DAO 层。
  - 查询条件在 Service 层构建（buildQueryWrapper）。
  - 使用 @AutoMapper（单数）而非 @AutoMappers。
  - API 路径使用标准 RESTful 格式（/list、/{id}）。
---

# CRUD 全栈开发规范（RuoYi-Vue-Plus 三层架构版）

> **⚠️ 重要声明**: 本项目后端采用 **三层架构**（Controller → Service → Mapper），**无独立 DAO 层**，Service 直接调用 Mapper。
> **前端检测**：如果 `plus-ui/` 目录存在，则包含 PC 管理端前端代码（Vue 3 + Element Plus），CRUD 开发时应同时生成前端代码。

## 核心架构特征

| 对比项 | 本项目 (RuoYi-Vue-Plus) |
|--------|----------------------|
| **包名前缀** | `org.dromara.*` |
| **架构** | 三层：Controller → Service → Mapper |
| **DAO 层** | ❌ 不存在，Service 直接注入 Mapper |
| **查询构建** | Service 层 `buildQueryWrapper()` |
| **Mapper 继承** | `BaseMapperPlus<Entity, VO>` |
| **对象转换** | `MapstructUtils.convert()` |
| **Entity 基类** | `TenantEntity`（多租户） |
| **BO 映射** | `@AutoMapper` 注解（单数） |
| **API 路径** | 标准 RESTful：`/list`、`/{id}` |

---

## 1. Entity 实体类（继承 TenantEntity）

```java
package org.dromara.demo.domain;

import org.dromara.common.tenant.core.TenantEntity;
import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import lombok.EqualsAndHashCode;
import java.io.Serial;

/**
 * XXX 对象
 *
 * @author Lion Li
 */
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("test_xxx")
public class Xxx extends TenantEntity {  // ✅ 继承 TenantEntity（多租户）

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键 ID
     */
    @TableId(value = "id")
    private Long id;

    /**
     * 名称
     */
    private String xxxName;

    /**
     * 状态（0正常 1停用）
     */
    private String status;

    /**
     * 删除标志
     */
    @TableLogic
    private Long delFlag;
}
```

---

## 2. BO 业务对象（@AutoMapper 映射）

```java
package org.dromara.demo.domain.bo;

import io.github.linpeilie.annotations.AutoMapper;
import org.dromara.demo.domain.Xxx;
import org.dromara.demo.domain.vo.XxxVo;
import org.dromara.common.core.validate.AddGroup;
import org.dromara.common.core.validate.EditGroup;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.dromara.common.mybatis.core.domain.BaseEntity;
import jakarta.validation.constraints.*;

/**
 * XXX 业务对象
 */
@Data
@EqualsAndHashCode(callSuper = true)
@AutoMapper(target = Xxx.class, reverseConvertGenerate = false)  // ✅ 映射到 Entity
public class XxxBo extends BaseEntity {

    /**
     * 主键 ID
     */
    @NotNull(message = "主键 ID 不能为空", groups = {EditGroup.class})
    private Long id;

    /**
     * 名称
     */
    @NotBlank(message = "名称不能为空", groups = {AddGroup.class, EditGroup.class})
    private String xxxName;

    /**
     * 状态
     */
    private String status;
}
```

---

## 3. VO 视图对象（@AutoMapper 映射）

```java
package org.dromara.demo.domain.vo;

import io.github.linpeilie.annotations.AutoMapper;
import org.dromara.demo.domain.Xxx;
import org.dromara.demo.domain.bo.XxxBo;
import cn.idev.excel.annotation.ExcelIgnoreUnannotated;
import cn.idev.excel.annotation.ExcelProperty;
import lombok.Data;
import java.io.Serial;
import java.io.Serializable;
import java.util.Date;

/**
 * XXX 视图对象
 */
@Data
@ExcelIgnoreUnannotated
@AutoMapper(target = Xxx.class)  // ✅ VO 也使用 @AutoMapper
public class XxxVo implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    /**
     * 主键 ID
     */
    @ExcelProperty(value = "主键 ID")
    private Long id;

    /**
     * 名称
     */
    @ExcelProperty(value = "名称")
    private String xxxName;

    /**
     * 状态
     */
    @ExcelProperty(value = "状态")
    private String status;

    /**
     * 创建时间
     */
    @ExcelProperty(value = "创建时间")
    private Date createTime;
}
```

---

## 4. Service 接口

```java
package org.dromara.demo.service;

import org.dromara.demo.domain.bo.XxxBo;
import org.dromara.demo.domain.vo.XxxVo;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import java.util.Collection;
import java.util.List;

/**
 * XXX 服务接口
 */
public interface IXxxService {

    /**
     * 根据 ID 查询
     */
    XxxVo queryById(Long id);

    /**
     * 查询列表
     */
    List<XxxVo> queryList(XxxBo bo);

    /**
     * 分页查询
     */
    TableDataInfo<XxxVo> queryPageList(XxxBo bo, PageQuery pageQuery);

    /**
     * 新增
     */
    Boolean insertByBo(XxxBo bo);

    /**
     * 修改
     */
    Boolean updateByBo(XxxBo bo);

    /**
     * 删除
     */
    Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid);
}
```

---

## 5. Service 实现类（⭐ 核心：三层架构，NO DAO 层）

```java
package org.dromara.demo.service.impl;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.dromara.common.core.exception.ServiceException;
import org.dromara.common.core.utils.MapstructUtils;
import org.dromara.common.core.utils.StringUtils;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.demo.domain.Xxx;
import org.dromara.demo.domain.bo.XxxBo;
import org.dromara.demo.domain.vo.XxxVo;
import org.dromara.demo.mapper.XxxMapper;
import org.dromara.demo.service.IXxxService;
import java.util.Collection;
import java.util.List;
import java.util.Map;

/**
 * XXX 服务实现
 *
 * @author Lion Li
 */
@Service
@RequiredArgsConstructor
public class XxxServiceImpl implements IXxxService {

    private final XxxMapper baseMapper;  // ✅ 直接注入 Mapper（NO DAO!）

    @Override
    public XxxVo queryById(Long id) {
        return baseMapper.selectVoById(id);
    }

    @Override
    public List<XxxVo> queryList(XxxBo bo) {
        return baseMapper.selectVoList(buildQueryWrapper(bo));
    }

    @Override
    public TableDataInfo<XxxVo> queryPageList(XxxBo bo, PageQuery pageQuery) {
        LambdaQueryWrapper<Xxx> lqw = buildQueryWrapper(bo);  // ✅ Service 层构建查询
        Page<XxxVo> result = baseMapper.selectVoPage(pageQuery.build(), lqw);
        return TableDataInfo.build(result);
    }

    @Override
    public Boolean insertByBo(XxxBo bo) {
        Xxx add = MapstructUtils.convert(bo, Xxx.class);  // ✅ MapstructUtils 转换
        validEntityBeforeSave(add);
        return baseMapper.insert(add) > 0;
    }

    @Override
    public Boolean updateByBo(XxxBo bo) {
        Xxx update = MapstructUtils.convert(bo, Xxx.class);
        validEntityBeforeSave(update);
        return baseMapper.updateById(update) > 0;
    }

    @Override
    public Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid) {
        if (isValid) {
            List<Xxx> list = baseMapper.selectByIds(ids);
            if (list.size() != ids.size()) {
                throw new ServiceException("您没有删除权限!");
            }
        }
        return baseMapper.deleteByIds(ids) > 0;
    }

    /**
     * 构建查询条件
     * ✅ Service 层直接构建（不是 DAO 层）
     */
    private LambdaQueryWrapper<Xxx> buildQueryWrapper(XxxBo bo) {
        Map<String, Object> params = bo.getParams();
        LambdaQueryWrapper<Xxx> lqw = Wrappers.lambdaQuery();

        // ✅ 精确匹配
        lqw.eq(bo.getId() != null, Xxx::getId, bo.getId());
        lqw.eq(StringUtils.isNotBlank(bo.getStatus()), Xxx::getStatus, bo.getStatus());

        // ✅ 模糊匹配
        lqw.like(StringUtils.isNotBlank(bo.getXxxName()), Xxx::getXxxName, bo.getXxxName());

        // ✅ 时间范围
        lqw.between(params.get("beginCreateTime") != null && params.get("endCreateTime") != null,
            Xxx::getCreateTime, params.get("beginCreateTime"), params.get("endCreateTime"));

        // ✅ 排序
        lqw.orderByAsc(Xxx::getId);
        return lqw;
    }

    /**
     * 保存前验证
     */
    private void validEntityBeforeSave(Xxx entity) {
        // TODO 做一些数据校验，如唯一约束
    }
}
```

---

## 6. Mapper 接口（继承 BaseMapperPlus）

```java
package org.dromara.demo.mapper;

import org.dromara.demo.domain.Xxx;
import org.dromara.demo.domain.vo.XxxVo;
import org.dromara.common.mybatis.core.mapper.BaseMapperPlus;

/**
 * XXX Mapper 接口
 */
public interface XxxMapper extends BaseMapperPlus<Xxx, XxxVo> {
    // ✅ 继承 BaseMapperPlus，已提供 selectVoById、selectVoPage、selectVoList 等方法
}
```

---

## 7. Controller 控制器（标准 RESTful 路径）

```java
package org.dromara.demo.controller;

import java.util.Arrays;
import java.util.List;
import lombok.RequiredArgsConstructor;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.validation.constraints.*;
import cn.dev33.satoken.annotation.SaCheckPermission;
import org.springframework.web.bind.annotation.*;
import org.springframework.validation.annotation.Validated;
import org.dromara.common.idempotent.annotation.RepeatSubmit;
import org.dromara.common.log.annotation.Log;
import org.dromara.common.log.enums.BusinessType;
import org.dromara.common.mybatis.core.page.PageQuery;
import org.dromara.common.mybatis.core.page.TableDataInfo;
import org.dromara.common.web.core.BaseController;
import org.dromara.common.core.domain.R;
import org.dromara.common.core.validate.AddGroup;
import org.dromara.common.core.validate.EditGroup;
import org.dromara.common.excel.utils.ExcelUtil;
import org.dromara.demo.domain.vo.XxxVo;
import org.dromara.demo.domain.bo.XxxBo;
import org.dromara.demo.service.IXxxService;

/**
 * XXX 管理控制器
 */
@Validated
@RequiredArgsConstructor
@RestController
@RequestMapping("/demo/xxx")
public class XxxController extends BaseController {  // ✅ 继承 BaseController

    private final IXxxService xxxService;

    /**
     * 查询列表
     * ✅ RESTful 路径：/list（不是 /pageXxxs）
     */
    @SaCheckPermission("demo:xxx:list")
    @GetMapping("/list")
    public TableDataInfo<XxxVo> list(XxxBo bo, PageQuery pageQuery) {
        return xxxService.queryPageList(bo, pageQuery);
    }

    /**
     * 获取详情
     * ✅ RESTful 路径：/{id}（不是 /getXxx/{id}）
     */
    @SaCheckPermission("demo:xxx:query")
    @GetMapping("/{id}")
    public R<XxxVo> getInfo(@NotNull(message = "主键不能为空") @PathVariable Long id) {
        return R.ok(xxxService.queryById(id));
    }

    /**
     * 新增
     * ✅ POST 空路径
     */
    @SaCheckPermission("demo:xxx:add")
    @Log(title = "XXX管理", businessType = BusinessType.INSERT)
    @RepeatSubmit()
    @PostMapping()
    public R<Void> add(@Validated(AddGroup.class) @RequestBody XxxBo bo) {
        return toAjax(xxxService.insertByBo(bo));
    }

    /**
     * 修改
     * ✅ PUT 空路径
     */
    @SaCheckPermission("demo:xxx:edit")
    @Log(title = "XXX管理", businessType = BusinessType.UPDATE)
    @RepeatSubmit()
    @PutMapping()
    public R<Void> edit(@Validated(EditGroup.class) @RequestBody XxxBo bo) {
        return toAjax(xxxService.updateByBo(bo));
    }

    /**
     * 删除
     * ✅ DELETE /{ids}
     */
    @SaCheckPermission("demo:xxx:remove")
    @Log(title = "XXX管理", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ids}")
    public R<Void> remove(@NotEmpty(message = "主键不能为空") @PathVariable Long[] ids) {
        return toAjax(xxxService.deleteWithValidByIds(Arrays.asList(ids), true));
    }

    /**
     * 导出
     */
    @SaCheckPermission("demo:xxx:export")
    @Log(title = "XXX管理", businessType = BusinessType.EXPORT)
    @PostMapping("/export")
    public void export(@Validated XxxBo bo, HttpServletResponse response) {
        List<XxxVo> list = xxxService.queryList(bo);
        ExcelUtil.exportExcel(list, "XXX数据", XxxVo.class, response);
    }
}
```

---

## 8. 数据库建表（SQL）

```sql
-- 表前缀：demo_（根据模块选择：sys_/demo_/workflow_ 等）
CREATE TABLE demo_xxx (
    id           BIGINT(20)   NOT NULL COMMENT '主键 ID',  -- ✅ 雪花 ID，不用 AUTO_INCREMENT
    tenant_id    VARCHAR(20)  DEFAULT '000000' COMMENT '租户 ID',

    -- 业务字段
    xxx_name     VARCHAR(100) NOT NULL COMMENT '名称',
    status       CHAR(1)      DEFAULT '0' COMMENT '状态(0正常 1停用)',

    -- 审计字段（必须）
    create_dept  BIGINT(20)   DEFAULT NULL COMMENT '创建部门',
    create_by    BIGINT(20)   DEFAULT NULL COMMENT '创建人',
    create_time  DATETIME     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_by    BIGINT(20)   DEFAULT NULL COMMENT '更新人',
    update_time  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    remark       VARCHAR(255) DEFAULT NULL COMMENT '备注',
    del_flag     BIGINT(20)   DEFAULT 0 COMMENT '删除标志(0正常 1已删除)',

    PRIMARY KEY (id)
) ENGINE=InnoDB COMMENT='XXX表';
```

---

## 架构对比

### 三层架构流程图

```
请求到达
   ↓
Controller （路由转发 + 权限检查 + 参数校验）
   ↓
Service （业务逻辑 + 查询构建 + 对象转换）
   ↓
Mapper （数据持久化）
   ↓
数据库
```

### 关键差异

| 环节 | 操作 | 位置 |
|------|------|------|
| **查询构建** | `buildQueryWrapper()` | **Service 层** ✅ |
| **Mapper 注入** | 在 Service 中注入 | ✅ 直接注入 baseMapper |
| **DAO 层** | 是否存在 | ❌ **不存在** |
| **对象转换** | `MapstructUtils.convert()` | Service 层 |
| **权限注解** | `@DataPermission` | Mapper 接口方法 |

---

## 常见错误速查

### ❌ 不要做

```java
// 错误 1: 在 Service 层注入 DAO
@Service
public class XxxServiceImpl {
    private final IXxxDao xxxDao;  // ❌ 本项目没有 DAO 层！
}

// 错误 2: 使用 BeanUtil
BeanUtil.copyProperties(bo, entity);  // ❌ 必须用 MapstructUtils.convert()

// 错误 3: Service 继承基类
public class XxxServiceImpl extends ServiceImpl<XxxMapper, Xxx> {  // ❌ 不继承！
}

// 错误 4: 使用 @AutoMappers（复数）
@AutoMappers({  // ❌ 本项目用单数 @AutoMapper
    @AutoMapper(target = Xxx.class)
})
public class XxxBo { }

// 错误 5: 包名错误
package org.dromara.xxx;  // ❌ 必须是 org.dromara.xxx

// 错误 6: 使用错误的路径格式
@GetMapping("/pageXxxs")  // ❌ 应该是 /list
@GetMapping("/getXxx/{id}")  // ❌ 应该是 /{id}
```

### ✅ 正确做法

```java
// 正确 1: 直接在 Service 中注入 Mapper
@Service
@RequiredArgsConstructor
public class XxxServiceImpl implements IXxxService {
    private final XxxMapper baseMapper;  // ✅ 直接注入 Mapper
}

// 正确 2: 使用 MapstructUtils
Xxx entity = MapstructUtils.convert(bo, Xxx.class);  // ✅

// 正确 3: Service 只实现接口
public class XxxServiceImpl implements IXxxService {  // ✅

// 正确 4: 使用 @AutoMapper（单数）
@AutoMapper(target = Xxx.class)  // ✅
public class XxxBo { }

// 正确 5: 使用 org.dromara 包名
package org.dromara.demo.service;  // ✅

// 正确 6: 使用标准 RESTful 路径
@GetMapping("/list")  // ✅
@GetMapping("/{id}")  // ✅
@PostMapping
@PutMapping
@DeleteMapping("/{ids}")
```

---

## 检查清单

生成代码前必须检查：

- [ ] **包名是否是 `org.dromara.*`**？
- [ ] **Service 是否只实现接口，不继承任何基类**？
- [ ] **Service 是否直接注入 Mapper（无 DAO 层）**？
- [ ] **buildQueryWrapper() 是否在 Service 层实现**？
- [ ] **Entity 是否继承 `TenantEntity`**？
- [ ] **BO 是否使用 `@AutoMapper`（单数）映射到 Entity**？
- [ ] **VO 是否使用 `@AutoMapper` 映射**？
- [ ] **是否使用 `MapstructUtils.convert()` 转换对象**？
- [ ] **是否所有类型都先 import 再使用短类名**？
- [ ] **Mapper 是否继承 `BaseMapperPlus<Entity, VO>`**？
- [ ] **Controller 是否使用标准 RESTful 路径（/list、/{id} 等）**？
- [ ] **是否使用了 `@DataPermission` 进行行级权限控制**？
- [ ] **SQL 是否使用了 `del_flag`（非 `is_deleted`）**？
- [ ] **主键是否使用雪花 ID（无 AUTO_INCREMENT）**？
- [ ] **所有代码注释是否使用中文**？（Javadoc、行内注释、SQL 注释）
- [ ] **SQL COMMENT 是否使用中文**？（禁止英文 COMMENT）

---

## 参考实现

查看已有的完整实现：

- **Entity 参考**: `org.dromara.demo.domain.TestDemo`
- **BO 参考**: `org.dromara.demo.domain.bo.TestDemoBo`
- **VO 参考**: `org.dromara.demo.domain.vo.TestDemoVo`
- **Service 参考**: `org.dromara.demo.service.impl.TestDemoServiceImpl`
- **Mapper 参考**: `org.dromara.demo.mapper.TestDemoMapper`
- **Controller 参考**: `org.dromara.demo.controller.TestDemoController`

**特别注意**：上述参考代码是本项目的标准实现，严格遵循三层架构（Service 直接调用 Mapper，无 DAO 层）。

---

## 9. 前端代码（plus-ui 存在时必须生成）

> **前提条件**：`plus-ui/` 目录存在时，CRUD 开发必须同时生成前端 3 个文件。
> **前端参考代码**：`plus-ui/src/api/system/notice/` 和 `plus-ui/src/views/system/notice/index.vue`

### 前端文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| API 请求 | `plus-ui/src/api/[模块]/[功能]/index.ts` | 5 个标准请求方法 |
| 类型定义 | `plus-ui/src/api/[模块]/[功能]/types.ts` | VO/Query/Form 接口 |
| 页面组件 | `plus-ui/src/views/[模块]/[功能]/index.vue` | CRUD 页面 |

### 9.1 API 请求文件模板（index.ts）

```typescript
import request from '@/utils/request';
import { XxxForm, XxxQuery, XxxVO } from './types';
import { AxiosPromise } from 'axios';

// 查询列表
export function listXxx(query: XxxQuery): AxiosPromise<XxxVO[]> {
  return request({
    url: '/[模块]/[功能]/list',
    method: 'get',
    params: query
  });
}

// 查询详细
export function getXxx(id: string | number): AxiosPromise<XxxVO> {
  return request({
    url: '/[模块]/[功能]/' + id,
    method: 'get'
  });
}

// 新增
export function addXxx(data: XxxForm) {
  return request({
    url: '/[模块]/[功能]',
    method: 'post',
    data: data
  });
}

// 修改
export function updateXxx(data: XxxForm) {
  return request({
    url: '/[模块]/[功能]',
    method: 'put',
    data: data
  });
}

// 删除
export function delXxx(id: string | number | Array<string | number>) {
  return request({
    url: '/[模块]/[功能]/' + id,
    method: 'delete'
  });
}
```

**API URL 规则**：必须与后端 Controller 的 `@RequestMapping` 路径一致。

### 9.2 类型定义文件模板（types.ts）

```typescript
// VO 视图对象（对应后端 XxxVo，用于列表展示）
export interface XxxVO extends BaseEntity {
  id: number;
  xxxName: string;        // 业务字段（驼峰命名）
  status: string;         // 状态
  // ... 根据后端 VO 字段对应
}

// Query 查询参数（对应后端 XxxBo 中作为查询条件的字段 + 分页参数）
export interface XxxQuery extends PageQuery {
  xxxName: string;        // 模糊查询字段
  status: string;         // 精确查询字段
  // ... 只包含查询条件字段
}

// Form 表单对象（对应后端 XxxBo 中的可编辑字段）
export interface XxxForm {
  id: number | string | undefined;  // 主键（新增时 undefined，修改时有值）
  xxxName: string;
  status: string;
  remark: string;
  // ... 包含表单可编辑字段
}
```

**类型映射规则**：

| 后端 Java 类型 | 前端 TypeScript 类型 |
|---------------|---------------------|
| Long / Integer | number |
| String | string |
| Date | string（JSON 序列化后） |
| BigDecimal | number 或 string |

**字段命名**：后端数据库字段 `xxx_name` → 后端 Java `xxxName` → 前端 TypeScript `xxxName`（驼峰一致）

### 9.3 CRUD 页面模板（index.vue）

```vue
<template>
  <div class="p-2">
    <!-- 搜索区域 -->
    <transition :enter-active-class="proxy?.animate.searchAnimate.enter" :leave-active-class="proxy?.animate.searchAnimate.leave">
      <div v-show="showSearch" class="mb-[10px]">
        <el-card shadow="hover">
          <el-form ref="queryFormRef" :model="queryParams" :inline="true">
            <!-- 文本搜索示例 -->
            <el-form-item label="名称" prop="xxxName">
              <el-input v-model="queryParams.xxxName" placeholder="请输入名称" clearable @keyup.enter="handleQuery" />
            </el-form-item>
            <!-- 下拉字典搜索示例 -->
            <el-form-item label="状态" prop="status">
              <el-select v-model="queryParams.status" placeholder="请选择状态" clearable>
                <el-option v-for="dict in sys_normal_disable" :key="dict.value" :label="dict.label" :value="dict.value" />
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
              <el-button icon="Refresh" @click="resetQuery">重置</el-button>
            </el-form-item>
          </el-form>
        </el-card>
      </div>
    </transition>

    <!-- 工具栏 + 表格 -->
    <el-card shadow="hover">
      <template #header>
        <el-row :gutter="10" class="mb8">
          <el-col :span="1.5">
            <el-button v-hasPermi="['[模块]:[功能]:add']" type="primary" plain icon="Plus" @click="handleAdd">新增</el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['[模块]:[功能]:edit']" type="success" plain icon="Edit" :disabled="single" @click="handleUpdate()">修改</el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['[模块]:[功能]:remove']" type="danger" plain icon="Delete" :disabled="multiple" @click="handleDelete()">删除</el-button>
          </el-col>
          <right-toolbar v-model:show-search="showSearch" @query-table="getList"></right-toolbar>
        </el-row>
      </template>

      <el-table v-loading="loading" border :data="xxxList" @selection-change="handleSelectionChange">
        <el-table-column type="selection" width="55" align="center" />
        <el-table-column label="名称" align="center" prop="xxxName" :show-overflow-tooltip="true" />
        <!-- 字典列示例 -->
        <el-table-column label="状态" align="center" prop="status" width="100">
          <template #default="scope">
            <dict-tag :options="sys_normal_disable" :value="scope.row.status" />
          </template>
        </el-table-column>
        <el-table-column label="创建时间" align="center" prop="createTime" width="160">
          <template #default="scope">
            <span>{{ proxy.parseTime(scope.row.createTime, '{y}-{m}-{d}') }}</span>
          </template>
        </el-table-column>
        <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
          <template #default="scope">
            <el-tooltip content="修改" placement="top">
              <el-button v-hasPermi="['[模块]:[功能]:edit']" link type="primary" icon="Edit" @click="handleUpdate(scope.row)"></el-button>
            </el-tooltip>
            <el-tooltip content="删除" placement="top">
              <el-button v-hasPermi="['[模块]:[功能]:remove']" link type="primary" icon="Delete" @click="handleDelete(scope.row)"></el-button>
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>

      <pagination v-show="total > 0" v-model:page="queryParams.pageNum" v-model:limit="queryParams.pageSize" :total="total" @pagination="getList" />
    </el-card>

    <!-- 添加或修改对话框 -->
    <el-dialog v-model="dialog.visible" :title="dialog.title" width="580px" append-to-body>
      <el-form ref="xxxFormRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="名称" prop="xxxName">
          <el-input v-model="form.xxxName" placeholder="请输入名称" />
        </el-form-item>
        <el-form-item label="状态">
          <el-radio-group v-model="form.status">
            <el-radio v-for="dict in sys_normal_disable" :key="dict.value" :value="dict.value">{{ dict.label }}</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="form.remark" type="textarea" placeholder="请输入备注" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Xxx" lang="ts">
import { listXxx, getXxx, delXxx, addXxx, updateXxx } from '@/api/[模块]/[功能]';
import { XxxForm, XxxQuery, XxxVO } from '@/api/[模块]/[功能]/types';

const { proxy } = getCurrentInstance() as ComponentInternalInstance;
// 字典引用：根据实际使用的字典类型替换
const { sys_normal_disable } = toRefs<any>(proxy?.useDict('sys_normal_disable'));

const xxxList = ref<XxxVO[]>([]);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref<Array<string | number>>([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);

const queryFormRef = ref<ElFormInstance>();
const xxxFormRef = ref<ElFormInstance>();

const dialog = reactive<DialogOption>({
  visible: false,
  title: ''
});

// 表单初始值（与 XxxForm 对应）
const initFormData: XxxForm = {
  id: undefined,
  xxxName: '',
  status: '0',
  remark: ''
};
const data = reactive<PageData<XxxForm, XxxQuery>>({
  form: { ...initFormData },
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    xxxName: '',
    status: ''
  },
  rules: {
    xxxName: [{ required: true, message: '名称不能为空', trigger: 'blur' }]
  }
});

const { queryParams, form, rules } = toRefs(data);

/** 查询列表 */
const getList = async () => {
  loading.value = true;
  const res = await listXxx(queryParams.value);
  xxxList.value = res.rows;
  total.value = res.total;
  loading.value = false;
};
/** 取消按钮 */
const cancel = () => {
  reset();
  dialog.visible = false;
};
/** 表单重置 */
const reset = () => {
  form.value = { ...initFormData };
  xxxFormRef.value?.resetFields();
};
/** 搜索按钮操作 */
const handleQuery = () => {
  queryParams.value.pageNum = 1;
  getList();
};
/** 重置按钮操作 */
const resetQuery = () => {
  queryFormRef.value?.resetFields();
  handleQuery();
};
/** 多选框选中数据 */
const handleSelectionChange = (selection: XxxVO[]) => {
  ids.value = selection.map((item) => item.id);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
};
/** 新增按钮操作 */
const handleAdd = () => {
  reset();
  dialog.visible = true;
  dialog.title = '添加';
};
/** 修改按钮操作 */
const handleUpdate = async (row?: XxxVO) => {
  reset();
  const xxxId = row?.id || ids.value[0];
  const { data } = await getXxx(xxxId);
  Object.assign(form.value, data);
  dialog.visible = true;
  dialog.title = '修改';
};
/** 提交按钮 */
const submitForm = () => {
  xxxFormRef.value?.validate(async (valid: boolean) => {
    if (valid) {
      form.value.id ? await updateXxx(form.value) : await addXxx(form.value);
      proxy?.$modal.msgSuccess('操作成功');
      dialog.visible = false;
      await getList();
    }
  });
};
/** 删除按钮操作 */
const handleDelete = async (row?: XxxVO) => {
  const xxxIds = row?.id || ids.value;
  await proxy?.$modal.confirm('是否确认删除选中的数据项？');
  await delXxx(xxxIds);
  await getList();
  proxy?.$modal.msgSuccess('删除成功');
};

onMounted(() => {
  getList();
});
</script>
```

### 前端关键规范

| 规范 | 说明 |
|------|------|
| **权限指令** | `v-hasPermi="['模块:功能:操作']"` 控制按钮显示 |
| **字典使用** | `proxy?.useDict('字典类型')` 获取，`<dict-tag>` 展示 |
| **分页组件** | `<pagination>` 组件，绑定 `pageNum`、`pageSize`、`total` |
| **工具栏** | `<right-toolbar>` 控制搜索区域显隐 |
| **日期格式** | `proxy.parseTime(date, '{y}-{m}-{d}')` |
| **弹窗组件** | `<el-dialog>` + `DialogOption` 类型 |
| **表单校验** | `rules` 配置 + `xxxFormRef.value?.validate()` |
| **全局类型** | `BaseEntity`、`PageQuery`、`PageData`、`DialogOption`、`ComponentInternalInstance` 已全局声明，无需 import |

### 前端常见错误

```typescript
// ❌ 错误 1: 导入全局类型（已在 env.d.ts 全局声明）
import { BaseEntity, PageQuery } from '@/types';  // 不需要！

// ✅ 正确: 直接使用，无需 import
export interface XxxVO extends BaseEntity { }
export interface XxxQuery extends PageQuery { }

// ❌ 错误 2: 权限字符串与后端不一致
v-hasPermi="['system:xxx:update']"  // 后端是 edit 不是 update

// ✅ 正确: 与后端 @SaCheckPermission 一致
v-hasPermi="['system:xxx:edit']"

// ❌ 错误 3: API URL 与后端路径不一致
url: '/api/system/xxx/list'  // 多了 /api 前缀

// ✅ 正确: 与后端 @RequestMapping 完全一致（request 工具会自动加前缀）
url: '/system/xxx/list'
```

### 前端参考代码

| 类型 | 位置 |
|------|------|
| API 参考 | `plus-ui/src/api/system/notice/index.ts` |
| 类型参考 | `plus-ui/src/api/system/notice/types.ts` |
| 页面参考 | `plus-ui/src/views/system/notice/index.vue` |

<!-- 抓蛙师 -->
