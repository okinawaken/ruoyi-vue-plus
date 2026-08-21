# /check - 代码规范检查（后端 + 前端）

作为代码规范检查助手，自动检测项目代码是否符合 RuoYi-Vue-Plus 规范。如果 `plus-ui/` 目录存在，同时检查前端代码。

## 检查范围

支持三种检查模式：

1. **全量检查**：`/check` - 检查所有代码（后端 + 前端）
2. **模块检查**：`/check system` - 检查指定模块
3. **文件检查**：`/check XxxServiceImpl.java` - 检查指定文件

---

## 检查清单总览

| 检查项 | 级别 | 说明 |
|--------|------|------|
| 包名规范 | 🔴 严重 | 必须是 `org.dromara.*` |
| 完整类型引用 | 🔴 严重 | 禁止内联全限定名，必须 import |
| API 路径规范 | 🔴 严重 | Controller 方法路径必须规范 |
| 权限注解检查 | 🟡 警告 | 必须使用 @SaCheckPermission |
| Service 接口 | 🟡 警告 | Service 接口必须有标准方法声明 |
| Entity 基类 | 🟡 警告 | 业务实体应继承 TenantEntity |
| BO 映射注解 | 🟡 警告 | 必须使用 @AutoMapper |
| 对象转换 | 🟡 警告 | 必须用 MapstructUtils，禁止 BeanUtil |
| Mapper 继承 | 🟢 建议 | 必须继承 BaseMapperPlus |
| Map 传递数据 | 🟢 建议 | 禁止用 Map 封装业务数据 |

---

## 检查详情

### 1. 包名规范 [🔴 严重]

```bash
# 检查错误包名
Grep pattern: "package com\.ruoyi\." path: ruoyi-modules/ output_mode: files_with_matches
Grep pattern: "import com\.ruoyi\." path: ruoyi-modules/ output_mode: files_with_matches
```

```java
// ❌ 错误
package com.ruoyi.system.service;
import com.ruoyi.common.core.domain.R;

// ✅ 正确
package org.dromara.system.service;
import org.dromara.common.core.domain.R;
```

### 2. 完整类型引用 [🔴 严重]

```bash
# 检查完整类型引用
Grep pattern: "org\.dromara\.[a-z]+\.[A-Z][a-zA-Z]+\s" path: ruoyi-modules/ glob: "*.java" output_mode: content
```

```java
// ❌ 错误
public org.dromara.common.core.domain.R<XxxVo> getXxx(Long id) { ... }

// ✅ 正确
import org.dromara.common.core.domain.R;
public R<XxxVo> getXxx(Long id) { ... }
```

### 3. API 路径规范 [🔴 严重]

```bash
# 检查 Controller 方法的路径规范
Grep pattern: "@(GetMapping|PostMapping|PutMapping|DeleteMapping)\(" path: ruoyi-modules/ glob: "*Controller.java" output_mode: content -C 1
```

**规范要求**：

| 操作 | HTTP 方法 | 路径格式 | 示例 |
|------|---------|--------|------|
| 分页查询 | GET | `/list` 或 `/` 空路径 | `@GetMapping("/list")` |
| 获取详情 | GET | `/{id}` | `@GetMapping("/{id}")` |
| 新增 | POST | `/` 空路径 | `@PostMapping` |
| 修改 | PUT | `/` 空路径 | `@PutMapping` |
| 删除 | DELETE | `/{ids}` | `@DeleteMapping("/{ids}")` |
| 导出 | POST | `/export` | `@PostMapping("/export")` |

```java
// ❌ 错误（路径不规范）
@GetMapping("/queryList")  // 应该用 /list
@PostMapping("/add")       // POST 应该不加路径
@PutMapping("/update")     // PUT 应该不加路径

// ✅ 正确
@GetMapping("/list")
@PostMapping
@PutMapping
@DeleteMapping("/{ids}")
```

### 4. 权限注解检查 [🟡 警告]

```bash
# 检查 Controller 是否使用权限注解
Grep pattern: "@SaCheckPermission" path: ruoyi-modules/ glob: "*Controller.java" output_mode: files_with_matches
Grep pattern: "public.*\(.*\)\s*\{" path: ruoyi-modules/ glob: "*Controller.java" output_mode: content -B 1
```

**规范要求**：Controller 的所有公开接口都必须添加 `@SaCheckPermission` 注解

```java
// ❌ 错误（缺少权限注解）
@GetMapping("/list")
public TableDataInfo<XxxVo> list(XxxBo bo, PageQuery pageQuery) {
    return service.queryPageList(bo, pageQuery);
}

// ✅ 正确
@SaCheckPermission("xxx:list")
@GetMapping("/list")
public TableDataInfo<XxxVo> list(XxxBo bo, PageQuery pageQuery) {
    return service.queryPageList(bo, pageQuery);
}
```

### 5. Service 接口 [🟡 警告]

```bash
# 检查 Service 接口是否声明标准方法
Grep pattern: "public interface I.*Service" path: ruoyi-modules/ glob: "*Service.java" output_mode: files_with_matches
```

**规范要求**：Service 接口必须声明以下标准方法

```java
// ❌ 错误（缺少标准方法声明）
public interface IXxxService {
    // 仅有实现类，无接口声明
}

// ✅ 正确
public interface IXxxService {
    /**
     * 查询单条记录
     */
    XxxVo queryById(Long id);

    /**
     * 分页查询列表
     */
    TableDataInfo<XxxVo> queryPageList(XxxBo bo, PageQuery pageQuery);

    /**
     * 查询全量列表
     */
    List<XxxVo> queryList(XxxBo bo);

    /**
     * 新增记录
     */
    Boolean insertByBo(XxxBo bo);

    /**
     * 修改记录
     */
    Boolean updateByBo(XxxBo bo);

    /**
     * 删除记录（带校验）
     */
    Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid);
}
```

### 6. Service 查询条件规范 [🟡 警告]

```bash
# 检查 Service 层是否规范构建查询条件
Grep pattern: "buildQueryWrapper" path: ruoyi-modules/ glob: "*ServiceImpl.java" output_mode: files_with_matches
```

```java
// ✅ 正确（在 ServiceImpl 中封装查询条件）
private LambdaQueryWrapper<Xxx> buildQueryWrapper(XxxBo bo) {
    Map<String, Object> params = bo.getParams();
    LambdaQueryWrapper<Xxx> lqw = Wrappers.lambdaQuery();
    lqw.like(StringUtils.isNotBlank(bo.getName()), Xxx::getName, bo.getName());
    lqw.eq(StringUtils.isNotBlank(bo.getStatus()), Xxx::getStatus, bo.getStatus());
    return lqw;
}
```

### 7. Entity 基类 [🟡 警告]

```bash
# 检查业务实体是否继承 TenantEntity
Grep pattern: "extends TenantEntity" path: ruoyi-modules/ glob: "*.java" output_mode: files_with_matches
Grep pattern: "extends BaseEntity" path: ruoyi-modules/ glob: "*.java" output_mode: files_with_matches
```

```java
// ⚠️ 普通实体（非多租户）
public class SysNotice extends BaseEntity { }

// ✅ 业务实体（多租户）
public class XxxEntity extends TenantEntity { }
```

### 8. BO 映射注解 [🟡 警告]

```bash
# 检查 BO 是否使用 @AutoMapper
Grep pattern: "@AutoMapper" path: ruoyi-modules/ glob: "*Bo.java" output_mode: files_with_matches
```

```java
// ❌ 错误（缺少映射注解）
public class XxxBo extends BaseEntity { }

// ✅ 正确（单映射目标）
@AutoMapper(target = Xxx.class, reverseConvertGenerate = false)
public class XxxBo extends BaseEntity { }

// ✅ 正确（多映射目标，如 SysOperLogBo）
@AutoMappers({
    @AutoMapper(target = SysOperLog.class, reverseConvertGenerate = false),
    @AutoMapper(target = OperLogEvent.class)
})
public class SysOperLogBo { }
```

### 9. 对象转换 [🟡 警告]

```bash
# 检查是否使用 BeanUtil
Grep pattern: "BeanUtil\.copy" path: ruoyi-modules/ output_mode: files_with_matches
Grep pattern: "BeanUtils\.copy" path: ruoyi-modules/ output_mode: files_with_matches
```

```java
// ❌ 错误
BeanUtil.copyProperties(bo, entity);
BeanUtils.copyProperties(source, target);

// ✅ 正确
XxxVo vo = MapstructUtils.convert(entity, XxxVo.class);
List<XxxVo> voList = MapstructUtils.convert(entityList, XxxVo.class);
```

### 10. Mapper 继承 [🟢 建议]

```bash
Grep pattern: "extends BaseMapperPlus" path: ruoyi-modules/ output_mode: files_with_matches
```

```java
// ✅ 正确
public interface XxxMapper extends BaseMapperPlus<Xxx, XxxVo> { }
```

### 11. Map 传递数据 [🟢 建议]

```bash
Grep pattern: "Map<String,\s*Object>" path: ruoyi-modules/ glob: "*Service*.java" output_mode: files_with_matches
```

```java
// ❌ 错误
public Map<String, Object> getXxx(Long id) {
    Map<String, Object> result = new HashMap<>();
    result.put("id", entity.getId());
    return result;
}

// ✅ 正确（创建 VO 类）
public XxxVo getXxx(Long id) {
    return MapstructUtils.convert(entity, XxxVo.class);
}
```

---

## 输出格式

```markdown
# 🔍 代码规范检查报告

**检查时间**：YYYY-MM-DD HH:mm
**检查范围**：[全量 / 模块名 / 文件名]

---

## 📋 检查结果汇总

| 类别 | 通过 | 警告 | 错误 |
|------|------|------|------|
| 后端 Java | X | X | X |

---

## 🔴 严重问题（必须修复）

### 1. [问题类型]

**文件**：`path/to/file.java:42`
**问题**：包名使用了 com.ruoyi
**代码**：
\```java
package com.ruoyi.system.service;
\```
**修复**：
\```java
package org.dromara.system.service;
\```

---

## 🟡 警告问题（建议修复）

### 1. [问题类型]
...

---

## 🟢 建议优化

### 1. [优化建议]
...

---

## ✅ 检查通过项

- [x] 包名规范
- [x] 对象转换
- ...
```

---

## 检查优先级

### 开发完成后必查（阻塞提交）

1. 包名是否是 `org.dromara.*`
2. 是否有完整类型引用（内联全限定名）
3. API 路径是否规范
4. 对象转换是否使用 MapstructUtils
5. 权限注解是否完整

### 代码审查建议查

1. Service 接口是否有标准方法声明
2. BO 是否有 @AutoMapper
3. Entity 是否继承正确的基类
4. Mapper 是否继承 BaseMapperPlus
5. 是否有冗余的 Map 传递

---

## 快速修复指南

### 包名错误修复

```bash
# 查找所有错误包名
Grep pattern: "package com\.ruoyi\." path: ruoyi-modules/ output_mode: files_with_matches
```

批量替换：`com.ruoyi` → `org.dromara`

### API 路径修复

| 错误写法 | 正确写法 | 说明 |
|---------|--------|------|
| `@GetMapping("/queryList")` | `@GetMapping("/list")` | 列表查询统一用 /list |
| `@PostMapping("/add")` | `@PostMapping` | POST 新增不加路径 |
| `@PutMapping("/update")` | `@PutMapping` | PUT 修改不加路径 |
| `@GetMapping` | `@GetMapping("/{id}")` | 单条查询必须加 ID 参数 |

### 权限注解修复

```java
// 快速修复模板
@SaCheckPermission("模块:功能:操作")
@GetMapping("/list")
public TableDataInfo<XxxVo> list(...) { ... }

// 权限标识符标准格式（参考 SysNoticeController）
// 模块:功能:list     - 分页查询（注意：不是 query）
// 模块:功能:query    - 获取详情（单条查询）
// 模块:功能:add      - 新增
// 模块:功能:edit     - 修改
// 模块:功能:remove   - 删除
// 模块:功能:export   - 导出
// 模块:功能:import   - 导入
// 示例：system:notice:list, demo:demo:query
```

### 对象转换修复

| 替换前 | 替换后 |
|--------|--------|
| `BeanUtil.copyProperties(a, b)` | `MapstructUtils.convert(a, B.class)` |
| `BeanUtils.copyProperties(a, b)` | `MapstructUtils.convert(a, B.class)` |
| `new HashMap<>()` (传递业务数据) | 创建专属 VO 类 |

---

## 参考

- 正确后端代码：`ruoyi-modules/ruoyi-system/.../controller/system/SysNoticeController.java`
- 正确 Service 代码：`ruoyi-modules/ruoyi-system/.../service/impl/SysNoticeServiceImpl.java`

---

## 前端代码规范检查（plus-ui 存在时执行）

> **前提**：执行 `ls plus-ui/` 确认前端目录存在后，才执行以下检查。

### 前端检查清单总览

| 检查项 | 级别 | 说明 |
|--------|------|------|
| API URL 一致性 | 🔴 严重 | 前端 API URL 必须与后端 Controller 路径一致 |
| 权限标识一致性 | 🔴 严重 | v-hasPermi 必须与后端 @SaCheckPermission 一致 |
| 类型定义完整性 | 🟡 警告 | types.ts 中 VO/Query/Form 必须与后端字段对应 |
| 全局类型误导入 | 🟡 警告 | BaseEntity/PageQuery 等全局类型不应 import |
| 字典使用规范 | 🟡 警告 | 必须使用 useDict + dict-tag 展示 |
| Composition API | 🟢 建议 | 必须使用 `<script setup>` 语法 |
| 表单校验 | 🟢 建议 | 必填字段必须有 rules 校验 |

### F1. API URL 一致性 [🔴 严重]

```bash
# 检查前端 API 文件中的 URL
Grep pattern: "url: '/" path: plus-ui/src/api/ glob: "*.ts" output_mode: content
```

**规范**：前端 `url` 必须与后端 `@RequestMapping` + `@GetMapping/@PostMapping` 拼接路径完全一致。

```typescript
// ❌ 错误（多了 /api 前缀，或路径不对）
url: '/api/system/notice/list'   // 不需要 /api
url: '/system/notice/query'      // 后端是 /list 不是 /query

// ✅ 正确
url: '/system/notice/list'       // 与后端一致
```

### F2. 权限标识一致性 [🔴 严重]

```bash
# 检查前端权限指令
Grep pattern: "v-hasPermi" path: plus-ui/src/views/ glob: "*.vue" output_mode: content
```

**规范**：`v-hasPermi` 的权限字符串必须与后端 `@SaCheckPermission` 完全一致。

```html
<!-- ❌ 错误（权限标识与后端不一致） -->
<el-button v-hasPermi="['system:notice:update']">修改</el-button>

<!-- ✅ 正确（后端用的是 edit） -->
<el-button v-hasPermi="['system:notice:edit']">修改</el-button>
```

**标准权限对照**：

| 操作 | 前端 v-hasPermi | 后端 @SaCheckPermission |
|------|----------------|----------------------|
| 查询列表 | `模块:功能:list` | `模块:功能:list` |
| 查询详情 | `模块:功能:query` | `模块:功能:query` |
| 新增 | `模块:功能:add` | `模块:功能:add` |
| 修改 | `模块:功能:edit` | `模块:功能:edit` |
| 删除 | `模块:功能:remove` | `模块:功能:remove` |
| 导出 | `模块:功能:export` | `模块:功能:export` |

### F3. 类型定义完整性 [🟡 警告]

```bash
# 检查 types.ts 文件
Grep pattern: "export interface" path: plus-ui/src/api/ glob: "types.ts" output_mode: content
```

**规范**：每个模块的 `types.ts` 必须定义 3 个接口：

```typescript
// ✅ 正确：3 个接口完整
export interface XxxVO extends BaseEntity { ... }   // 列表展示
export interface XxxQuery extends PageQuery { ... }  // 查询参数
export interface XxxForm { ... }                     // 表单对象
```

### F4. 全局类型误导入 [🟡 警告]

```bash
# 检查是否错误导入全局类型
Grep pattern: "import.*BaseEntity|import.*PageQuery|import.*DialogOption|import.*ComponentInternalInstance" path: plus-ui/src/ glob: "*.ts" output_mode: files_with_matches
Grep pattern: "import.*BaseEntity|import.*PageQuery|import.*DialogOption|import.*ComponentInternalInstance" path: plus-ui/src/ glob: "*.vue" output_mode: files_with_matches
```

**规范**：以下类型已在 `env.d.ts` 全局声明，不需要 import：

```typescript
// ❌ 错误（不需要导入全局类型）
import { BaseEntity, PageQuery } from '@/types';

// ✅ 正确（直接使用）
export interface XxxVO extends BaseEntity { }
export interface XxxQuery extends PageQuery { }
```

### F5. 字典使用规范 [🟡 警告]

```bash
# 检查字典使用
Grep pattern: "useDict" path: plus-ui/src/views/ glob: "*.vue" output_mode: content
Grep pattern: "dict-tag" path: plus-ui/src/views/ glob: "*.vue" output_mode: files_with_matches
```

**规范**：状态/类型等字典字段必须使用 `useDict` + `<dict-tag>` 展示：

```typescript
// ✅ 正确
const { sys_normal_disable } = toRefs<any>(proxy?.useDict('sys_normal_disable'));
```

```html
<!-- ✅ 正确：表格列使用 dict-tag -->
<el-table-column label="状态" prop="status">
  <template #default="scope">
    <dict-tag :options="sys_normal_disable" :value="scope.row.status" />
  </template>
</el-table-column>
```

### F6. Composition API [🟢 建议]

```bash
# 检查是否使用 Options API
Grep pattern: "export default {" path: plus-ui/src/views/ glob: "*.vue" output_mode: files_with_matches
```

**规范**：必须使用 `<script setup>` 语法，不使用 Options API。

### F7. 表单校验 [🟢 建议]

```bash
# 检查表单是否有 rules
Grep pattern: ":rules=" path: plus-ui/src/views/ glob: "*.vue" output_mode: files_with_matches
```

**规范**：弹窗表单必须配置 `:rules` 校验，必填字段必须有 required 规则。

---

### 前端输出格式

在检查报告的"检查结果汇总"表中增加前端行：

```markdown
| 类别 | 通过 | 警告 | 错误 |
|------|------|------|------|
| 后端 Java | X | X | X |
| 前端 Vue/TS | X | X | X |
```

### 前端检查优先级

**开发完成后必查（阻塞提交）**：
1. API URL 与后端路径一致
2. 权限标识与后端一致

**代码审查建议查**：
3. 类型定义 VO/Query/Form 完整
4. 无错误导入全局类型
5. 字典字段使用 dict-tag 展示
6. 使用 Composition API
7. 表单有校验规则
