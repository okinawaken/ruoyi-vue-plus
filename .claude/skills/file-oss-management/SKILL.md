---
name: file-oss-management
description: |
  当需要进行文件上传、下载、存储管理时自动使用此 Skill。支持本地存储、阿里云OSS、腾讯云COS、七牛云、MinIO等。

  触发场景：
  - 文件上传下载
  - 云存储配置
  - 预签名URL生成
  - 文件元数据管理
  - 图片处理

  触发词：文件上传、OSS、云存储、MinIO、阿里云、腾讯云、七牛、图片上传、文件下载、预签名、presigned
---

# 文件与云存储指南

> ⚠️ **本项目规范**：本文档基于项目实际代码编写，所有API和字段名均已验证。标记 `🔴 本项目规范` 的部分必须严格遵守。

---

## 支持的存储类型

> **统一协议**：基于 AWS S3 SDK v2，支持所有兼容 S3 协议的云服务

| 类型 | 配置键示例 | 说明 |
|------|-----------|------|
| 本地存储 | `local` | 存储到服务器本地目录 |
| 阿里云OSS | `aliyun` | 阿里云对象存储 |
| 腾讯云COS | `qcloud` | 腾讯云对象存储 |
| 七牛云 | `qiniu` | 七牛云存储 |
| MinIO | `minio` | 开源对象存储 |
| 华为云OBS | `obs` | 华为云对象存储 |

---

## 核心类（已验证）

| 类 | 位置 | 说明 |
|----|------|------|
| `OssFactory` | `ruoyi-common-oss/.../factory/OssFactory.java` | 获取 OssClient 实例（只有2个方法） |
| `OssClient` | `ruoyi-common-oss/.../core/OssClient.java` | 统一操作入口（基于 AWS S3 SDK v2） |
| `UploadResult` | `ruoyi-common-oss/.../entity/UploadResult.java` | 上传结果（url, filename, eTag） |
| `ISysOssService` | `ruoyi-system/.../service/ISysOssService.java` | OSS 文件管理服务接口 |
| `SysOssServiceImpl` | `ruoyi-system/.../service/impl/SysOssServiceImpl.java` | OSS 文件管理服务实现 |
| `SysOssController` | `ruoyi-system/.../controller/system/SysOssController.java` | OSS 文件上传下载接口 |

---

## 🔴 基础使用（本项目规范）

### 获取 OssClient

> **实际代码位置**：`ruoyi-common-oss/src/main/java/org/dromara/common/oss/factory/OssFactory.java`

> **重要**：OssFactory 只有 2 个方法，配置键是字符串（如 "aliyun", "minio"），不是枚举。

```java
import org.dromara.common.oss.factory.OssFactory;
import org.dromara.common.oss.core.OssClient;

// ✅ 获取默认配置的客户端（从 Redis 读取默认 configKey）
OssClient client = OssFactory.instance();

// ✅ 根据配置键获取（配置键是字符串）
OssClient client = OssFactory.instance("aliyun");
OssClient client = OssFactory.instance("minio");

// ❌ 错误：不存在 OssType 枚举参数
OssClient client = OssFactory.instance(OssType.ALIYUN);  // 编译错误！
```

**工作原理**：
- 使用 `ConcurrentHashMap` 缓存 OssClient 实例
- 使用 `ReentrantLock` 双检锁模式确保线程安全
- 支持多租户隔离（对每个租户的 configKey 分别缓存）
- 配置信息从 Redis 中读取（`CacheNames.SYS_OSS_CONFIG`）

---

### 文件上传

> **实际代码位置**：`ruoyi-common-oss/src/main/java/org/dromara/common/oss/core/OssClient.java:140-227`

```java
import org.dromara.common.oss.entity.UploadResult;
import java.nio.file.Path;

// ✅ 1. 上传字节数组，自动生成路径（推荐）
byte[] data = multipartFile.getBytes();
UploadResult result = client.uploadSuffix(data, ".jpg", "image/jpeg");
// 结果：prefix/2024/12/01/uuid.jpg

// ✅ 2. 上传输入流，自动生成路径
InputStream is = multipartFile.getInputStream();
Long fileSize = multipartFile.getSize();
UploadResult result = client.uploadSuffix(is, ".jpg", fileSize, "image/jpeg");

// ✅ 3. 上传 File 对象，自动生成路径
File file = new File("/path/to/file.jpg");
UploadResult result = client.uploadSuffix(file, ".jpg");

// ✅ 4. 上传到指定路径（手动指定完整 key）
Path filePath = file.toPath();
UploadResult result = client.upload(filePath, "avatar/user123.jpg", null, "image/jpeg");

// ✅ 5. 上传流到指定路径
UploadResult result = client.upload(is, "images/photo.jpg", fileSize, "image/jpeg");
```

**方法签名（已验证）**：
```java
// 上传本地文件到S3
public UploadResult upload(Path filePath, String key, String md5Digest, String contentType)

// 上传输入流到S3
public UploadResult upload(InputStream inputStream, String key, Long length, String contentType)

// 上传字节数组（带后缀自动拼接路径）
public UploadResult uploadSuffix(byte[] data, String suffix, String contentType)

// 上传流（带后缀自动拼接路径）
public UploadResult uploadSuffix(InputStream inputStream, String suffix, Long length, String contentType)

// 上传文件对象（带后缀自动拼接路径）
public UploadResult uploadSuffix(File file, String suffix)
```

---

### UploadResult 字段（已验证）

> **实际代码位置**：`ruoyi-common-oss/src/main/java/org/dromara/common/oss/entity/UploadResult.java`

> **重要**：UploadResult 使用 Lombok `@Builder`，只有 3 个字段。

| 字段 | 类型 | 说明 |
|------|------|------|
| `url` | String | 文件访问URL（完整路径） |
| `filename` | String | 文件名/对象键（注意是小写 'n'） |
| `eTag` | String | 文件校验标记（用来校验文件） |

```java
// ✅ 正确
String url = result.getUrl();
String filename = result.getFilename();  // 小写 'n'
String eTag = result.getETag();

// ❌ 错误：不存在这些字段
String fileName = result.getFileName();  // 编译错误！
Long fileSize = result.getFileSize();    // 编译错误！
String contentType = result.getContentType();  // 编译错误！
```

---

## 文件下载

> **实际代码位置**：`OssClient.java:236-319`

```java
import java.nio.file.Path;
import java.io.OutputStream;

// ✅ 1. 下载到临时文件
Path tempFile = client.fileDownload("images/photo.jpg");
// 返回临时文件路径

// ✅ 2. 下载到输出流（推荐用于响应流）
OutputStream out = response.getOutputStream();
client.download("images/photo.jpg", out, contentLength -> {
    // 可选的文件大小回调（用于设置响应头）
    response.setContentLengthLong(contentLength);
});

// ✅ 3. 获取文件输入流
InputStream is = client.getObjectContent("images/photo.jpg");
// 注意：此方法会创建临时文件，使用后会自动删除
```

**方法签名（已验证）**：
```java
// 下载文件到临时目录
public Path fileDownload(String path)

// 下载文件到输出流（含文件大小回调）
public void download(String key, OutputStream out, Consumer<Long> consumer)

// 获取文件输入流
public InputStream getObjectContent(String path) throws IOException
```

---

## 文件操作

> **实际代码位置**：`OssClient.java`

```java
// ✅ 删除文件
client.delete("images/photo.jpg");

// ❌ 以下方法不存在于 OssClient
client.copyFile(...);        // 不存在
client.getFileMetadata(...); // 不存在
client.listFiles(...);       // 不存在
```

---

## 预签名URL

> **实际代码位置**：`OssClient.java:343-375`

### 下载预签名URL

```java
import java.time.Duration;

// ✅ 生成60分钟有效的预签名下载URL
String presignedUrl = client.createPresignedGetUrl("images/photo.jpg", Duration.ofMinutes(60));

// ❌ 以下方法不存在
String url = client.generatePresignedUrl(...);  // 不存在
String url = client.generatePublicUrl(...);     // 不存在
```

### 上传预签名URL（前端直传）

```java
import java.util.Map;

// ✅ 生成预签名上传URL
Map<String, String> metadata = Map.of("user-id", "123");
String uploadUrl = client.createPresignedPutUrl(
    "images/upload.jpg",           // 对象键
    Duration.ofHours(1),           // 有效期
    metadata                       // 元数据（可为 null）
);

// 前端使用此 URL 直接 PUT 上传
```

**方法签名（已验证）**：
```java
// 创建下载预签名URL（GET请求）
public String createPresignedGetUrl(String objectKey, Duration expiredTime)

// 创建上传预签名URL（PUT请求）
public String createPresignedPutUrl(String objectKey, Duration expiredTime, Map<String, String> metadata)
```

---

## OssClient 其他工具方法

> **实际代码位置**：`OssClient.java`

```java
// 获取云存储服务的基础URL
String baseUrl = client.getUrl();

// 获取终端点URL (http:// 或 https://)
String endpoint = client.getEndpoint();

// 获取自定义域名或终端点
String domain = client.getDomain();

// 获取服务商配置键
String configKey = client.getConfigKey();

// 获取桶权限类型（PUBLIC 或 PRIVATE）
AccessPolicyType accessPolicy = client.getAccessPolicy();

// 生成对象键路径（prefix + dateTime + uuid + suffix）
String path = client.getPath("", ".jpg");

// 移除基础URL获取相对路径
String relativePath = client.removeBaseUrl(fullUrl);

// 检查配置是否相同
boolean isSame = client.checkPropertiesSame(newProperties);
```

---

## Controller 示例（已验证）

> **实际代码位置**：`ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/controller/system/SysOssController.java`

### API 接口清单

| 操作 | HTTP方法 | 路径 | 权限 |
|------|---------|------|------|
| 查询列表 | GET | `/resource/oss/list` | `system:oss:list` |
| 批量查询 | GET | `/resource/oss/listByIds/{ossIds}` | `system:oss:query` |
| 上传文件 | POST | `/resource/oss/upload` | `system:oss:upload` |
| 下载文件 | GET | `/resource/oss/download/{ossId}` | `system:oss:download` |
| 删除文件 | DELETE | `/resource/oss/{ossIds}` | `system:oss:remove` |

### 🔴 文件上传接口（本项目规范）

```java
@RestController
@RequestMapping("/resource/oss")
@RequiredArgsConstructor
public class SysOssController extends BaseController {

    private final ISysOssService ossService;

    /**
     * 上传文件
     *
     * 🔴 注意：
     * 1. 必须使用 @RequestPart("file")
     * 2. 必须指定 consumes = MediaType.MULTIPART_FORM_DATA_VALUE
     * 3. 参数名必须是 "file"（前端约定）
     */
    @SaCheckPermission("system:oss:upload")
    @Log(title = "OSS对象存储", businessType = BusinessType.INSERT)
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public R<SysOssUploadVo> upload(@RequestPart("file") MultipartFile file) {
        // 调用 Service 上传（会保存到数据库）
        SysOssVo oss = ossService.upload(file);

        // 构建返回对象
        SysOssUploadVo uploadVo = new SysOssUploadVo();
        uploadVo.setUrl(oss.getUrl());
        uploadVo.setFileName(oss.getOriginalName());
        uploadVo.setOssId(oss.getOssId().toString());

        return R.ok(uploadVo);
    }

    /**
     * 下载文件
     */
    @SaCheckPermission("system:oss:download")
    @GetMapping("/download/{ossId}")
    public void download(@PathVariable Long ossId, HttpServletResponse response) throws IOException {
        ossService.download(ossId, response);
    }

    /**
     * 删除文件
     */
    @SaCheckPermission("system:oss:remove")
    @Log(title = "OSS对象存储", businessType = BusinessType.DELETE)
    @DeleteMapping("/{ossIds}")
    public R<Void> remove(@NotEmpty(message = "主键不能为空") @PathVariable Long[] ossIds) {
        return toAjax(ossService.deleteWithValidByIds(List.of(ossIds), true));
    }
}
```

**返回对象 SysOssUploadVo**：
```java
@Data
public class SysOssUploadVo {
    private String url;        // 文件URL
    private String fileName;   // 原始文件名
    private String ossId;      // OSS对象ID（String类型）
}
```

---

## Service 层示例（已验证）

> **实际代码位置**：`ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/service/impl/SysOssServiceImpl.java`

### 服务接口方法（ISysOssService）

```java
// 分页查询OSS对象
TableDataInfo<SysOssVo> queryPageList(SysOssBo sysOss, PageQuery pageQuery);

// 根据ossId列表查询
List<SysOssVo> listByIds(Collection<Long> ossIds);

// 单个ID查询（带缓存）
@Cacheable(cacheNames = CacheNames.SYS_OSS, key = "#ossId")
SysOssVo getById(Long ossId);

// MultipartFile上传
SysOssVo upload(MultipartFile file);

// File对象上传
SysOssVo upload(File file);

// 文件下载
void download(Long ossId, HttpServletResponse response) throws IOException;

// 删除文件
Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid);
```

### 文件上传完整流程

```java
@RequiredArgsConstructor
@Service
public class SysOssServiceImpl implements ISysOssService {

    private final SysOssMapper baseMapper;

    /**
     * 上传文件（保存到 OSS 并记录到数据库）
     *
     * 🔴 重要：
     * 1. 使用 new ServiceException() 抛出异常
     * 2. UploadResult.getFilename() 是小写 'n'
     * 3. 扩展信息存储在 ext1 字段（JSON格式）
     */
    @Override
    public SysOssVo upload(MultipartFile file) {
        if (ObjectUtil.isNull(file) || file.isEmpty()) {
            throw new ServiceException("上传文件不能为空");
        }

        // 1. 提取文件后缀
        String originalfileName = file.getOriginalFilename();
        String suffix = StringUtils.substring(originalfileName,
                                             originalfileName.lastIndexOf("."),
                                             originalfileName.length());

        // 2. 获取 OSS 客户端（默认配置）
        OssClient storage = OssFactory.instance();

        // 3. 上传文件
        UploadResult uploadResult;
        try {
            uploadResult = storage.uploadSuffix(file.getBytes(), suffix, file.getContentType());
        } catch (IOException e) {
            throw new ServiceException(e.getMessage());
        }

        // 4. 构建扩展信息
        SysOssExt ext1 = new SysOssExt();
        ext1.setFileSize(file.getSize());
        ext1.setContentType(file.getContentType());

        // 5. 保存到数据库
        return buildResultEntity(originalfileName, suffix,
            storage.getConfigKey(), uploadResult, ext1);
    }

    /**
     * 构建结果实体并保存
     */
    private SysOssVo buildResultEntity(String originalfileName, String suffix,
        String configKey, UploadResult uploadResult, SysOssExt ext1) {

        SysOss oss = new SysOss();
        oss.setUrl(uploadResult.getUrl());
        oss.setFileSuffix(suffix);
        oss.setFileName(uploadResult.getFilename());       // 对象键（小写 'n'）
        oss.setOriginalName(originalfileName);              // 原始文件名
        oss.setService(configKey);                          // 服务商标识
        oss.setExt1(JsonUtils.toJsonString(ext1));         // 扩展信息（JSON）

        baseMapper.insert(oss);
        SysOssVo sysOssVo = MapstructUtils.convert(oss, SysOssVo.class);
        return this.matchingUrl(sysOssVo);
    }

    /**
     * 文件下载
     */
    @Override
    public void download(Long ossId, HttpServletResponse response) throws IOException {
        SysOssVo sysOss = SpringUtils.getAopProxy(this).getById(ossId);
        if (ObjectUtil.isNull(sysOss)) {
            throw new ServiceException("文件数据不存在!");
        }

        // 设置响应头
        FileUtils.setAttachmentResponseHeader(response, sysOss.getOriginalName());
        response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE + "; charset=UTF-8");

        // 下载文件
        OssClient storage = OssFactory.instance(sysOss.getService());
        storage.download(sysOss.getFileName(),
            response.getOutputStream(),
            response::setContentLengthLong);  // 设置响应头Content-Length
    }

    /**
     * 删除文件
     */
    @Override
    public Boolean deleteWithValidByIds(Collection<Long> ids, Boolean isValid) {
        if (isValid) {
            // 做一些业务上的校验，判断是否需要校验
        }

        List<SysOss> list = baseMapper.selectByIds(ids);
        for (SysOss sysOss : list) {
            OssClient storage = OssFactory.instance(sysOss.getService());
            storage.delete(sysOss.getUrl());  // 删除OSS中的文件
        }

        return baseMapper.deleteByIds(ids) > 0;  // 删除数据库记录
    }

    /**
     * 为私有文件生成预签名 URL
     *
     * 🔴 重要：私有桶的文件需要生成临时访问 URL（120秒有效）
     */
    private SysOssVo matchingUrl(SysOssVo oss) {
        OssClient storage = OssFactory.instance(oss.getService());
        // 仅修改桶类型为 private 的URL，临时URL时长为120s
        if (AccessPolicyType.PRIVATE == storage.getAccessPolicy()) {
            oss.setUrl(storage.createPresignedGetUrl(oss.getFileName(), Duration.ofSeconds(120)));
        }
        return oss;
    }
}
```

---

## 数据库实体类（已验证）

### SysOss 实体

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("sys_oss")
public class SysOss extends TenantEntity {

    @TableId(value = "oss_id")
    private Long ossId;              // 对象存储主键

    private String fileName;         // 文件名（OSS对象键）
    private String originalName;     // 原始文件名
    private String fileSuffix;       // 文件后缀名（如 .jpg）
    private String url;              // URL地址
    private String ext1;             // 扩展字段（JSON格式）
    private String service;          // 服务商标识

    // 继承自 TenantEntity 的字段
    // tenantId, createTime, createBy, updateTime, updateBy, delFlag 等
}
```

### SysOssVo 视图对象

```java
@Data
@AutoMapper(target = SysOss.class)
public class SysOssVo implements Serializable {

    private Long ossId;
    private String fileName;
    private String originalName;
    private String fileSuffix;
    private String url;
    private String ext1;
    private Date createTime;
    private Long createBy;

    @Translation(type = TransConstant.USER_ID_TO_NAME, mapper = "createBy")
    private String createByName;  // 上传人名称（自动翻译）

    private String service;
}
```

### SysOssBo 业务对象

```java
@Data
@EqualsAndHashCode(callSuper = true)
@AutoMapper(target = SysOss.class, reverseConvertGenerate = false)
public class SysOssBo extends BaseEntity {

    private Long ossId;
    private String fileName;
    private String originalName;
    private String fileSuffix;
    private String url;
    private String ext1;
    private String service;

    // 继承自 BaseEntity 的 params 字段（时间范围查询用）
}
```

### SysOssExt 扩展信息对象

> **存储在 ext1 字段中**：以 JSON 字符串形式存储

```java
@Data
public class SysOssExt implements Serializable {

    private String bizType;         // 业务类型（avatar、report等）
    private Long fileSize;          // 文件大小（字节）
    private String contentType;     // MIME类型
    private String source;          // 来源标识
    private String uploadIp;        // 上传IP
    private String remark;          // 备注
    private List<String> tags;      // 标签
    private String refId;           // 绑定业务ID
    private String refType;         // 绑定业务类型
    private Boolean isTemp;         // 是否临时文件
    private String md5;             // 文件MD5值
}
```

---

## 配置说明

### 数据库配置表（sys_oss_config）

> **配置存储位置**：Redis（`CacheNames.SYS_OSS_CONFIG`）

| 字段 | 说明 |
|------|------|
| `config_key` | 配置标识（如 aliyun、minio） |
| `access_key` | Access Key |
| `secret_key` | Secret Key |
| `bucket_name` | 存储桶名称 |
| `prefix` | 路径前缀 |
| `endpoint` | 服务端点 |
| `domain` | 自定义域名 |
| `is_https` | 是否HTTPS（Y/N） |
| `region` | 区域 |
| `access_policy` | 访问策略（0-private, 1-public, 2-custom） |
| `status` | 启用状态 |

### OssProperties 配置类

```java
@Data
public class OssProperties {

    private String tenantId;       // 租户ID
    private String endpoint;       // 访问站点（endpoint）
    private String domain;         // 自定义域名
    private String prefix;         // 文件前缀（对象键前缀）
    private String accessKey;      // ACCESS_KEY
    private String secretKey;      // SECRET_KEY
    private String bucketName;     // 存储空间名
    private String region;         // 存储区域
    private String isHttps;        // 是否HTTPS（Y/N）
    private String accessPolicy;   // 桶权限类型（0-private, 1-public, 2-custom）
}
```

### 访问策略

| 策略 | 说明 |
|------|------|
| `PRIVATE` | 私有访问（"0"），需要预签名URL（matchingUrl 自动处理） |
| `PUBLIC` | 公开访问（"1"），URL 直接可访问 |
| `CUSTOM` | 自定义访问（"2"），公开读取 |

---

## 最佳实践

### 1. 文件路径规范

```java
// ✅ 推荐：使用 uploadSuffix 自动生成路径
OssClient client = OssFactory.instance();
UploadResult result = client.uploadSuffix(file.getBytes(), ".jpg", contentType);
// 结果：prefix/2024/12/01/550e8400-e29b.jpg

// ❌ 避免：手动拼接路径容易出错
UploadResult result = client.upload(file.toPath(), "avatar/" + fileName, null, contentType);
```

### 2. 文件上传后保存数据库

```java
// ✅ 推荐：使用 SysOssService 上传（自动保存到数据库）
@Autowired
private ISysOssService ossService;

SysOssVo ossVo = ossService.upload(multipartFile);
Long ossId = ossVo.getOssId();  // 保存 ossId 到业务表

// ❌ 不推荐：直接使用 OssClient（无数据库记录，难以管理）
OssClient client = OssFactory.instance();
UploadResult result = client.uploadSuffix(file.getBytes(), ".jpg", contentType);
```

### 3. 文件类型校验

```java
// 限制允许的文件类型
private static final Set<String> ALLOWED_TYPES = Set.of(
    "image/jpeg", "image/png", "image/gif", "application/pdf"
);

public SysOssVo upload(MultipartFile file) {
    if (!ALLOWED_TYPES.contains(file.getContentType())) {
        throw new ServiceException("不支持的文件类型");
    }
    // 继续上传...
}
```

### 4. 文件大小限制

```yaml
# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 20MB
```

### 5. 异常处理方式

```java
// ✅ 正确：使用 new ServiceException()
if (file.isEmpty()) {
    throw new ServiceException("上传文件不能为空");
}

// ❌ 错误：不存在 of() 静态方法
throw ServiceException.of("上传文件不能为空");  // 编译错误！
```

---

## 常见问题

### ⚠️ UploadResult.getFilename() vs getFileName()

```java
// ✅ 正确：字段名是 filename（小写 'n'）
UploadResult result = client.uploadSuffix(file.getBytes(), ".jpg", contentType);
String filename = result.getFilename();  // 正确

// ❌ 错误：不是 fileName（驼峰）
String fileName = result.getFileName();  // 编译错误！
```

### ⚠️ ServiceException 构造方式

```java
// ✅ 正确：使用 new ServiceException()
if (file.isEmpty()) {
    throw new ServiceException("上传文件不能为空");
}

// ❌ 错误：不存在 of() 静态方法
throw ServiceException.of("上传文件不能为空");  // 编译错误！
```

### 上传失败：Access Denied

- 检查 Access Key 和 Secret Key
- 检查存储桶权限配置
- 检查 IP 白名单

### 预签名URL无效

- 检查服务器时间是否同步
- 检查 URL 有效期是否过期
- 私有桶必须使用预签名URL（matchingUrl 自动处理）

### 跨域问题

配置存储桶 CORS 规则：
- AllowedOrigins: `*` 或具体域名
- AllowedMethods: `GET, PUT, POST, DELETE`
- AllowedHeaders: `*`

---

## 快速对照表

| ❌ 错误 | ✅ 正确 |
|--------|--------|
| `OssFactory.instance(OssType.ALIYUN)` | `OssFactory.instance("aliyun")` |
| `client.uploadFile(file, key, type)` | `client.upload(file.toPath(), key, null, type)` |
| `client.uploadStream(is, key, size, type)` | `client.upload(is, key, size, type)` |
| `result.getFileName()` | `result.getFilename()` |
| `result.getFileSize()` | 不存在此字段 |
| `client.downloadToTempFile(path)` | `client.fileDownload(path)` |
| `client.generatePresignedUrl(key, duration)` | `client.createPresignedGetUrl(key, duration)` |
| `throw ServiceException.of("msg")` | `throw new ServiceException("msg")` |
| `client.copyFile(...)` | 不存在此方法 |
| `client.listFiles(...)` | 不存在此方法 |

---

## 核心工作流程

### 上传流程

```
1. Controller 接收 MultipartFile
2. Service 验证文件非空
3. 从文件名提取后缀
4. 获取默认 OssClient（从 Redis 读取配置）
5. 上传到 OSS（自动生成对象键：prefix/date/uuid + suffix）
6. 保存文件信息和扩展信息到数据库（ext1字段存JSON）
7. 若是私有桶，生成预签名URL（120秒有效）
8. 返回上传结果
```

### 下载流程

```
1. Controller 接收 ossId
2. Service 从数据库查询文件信息（带缓存）
3. 检查私有桶，若是则生成预签名URL
4. 获取对应的 OssClient
5. 下载文件到输出流
6. 设置响应头（Content-Disposition、Content-Type、Content-Length）
```

### 删除流程

```
1. 查询数据库获取文件信息
2. 获取对应的 OssClient
3. 删除 OSS 中的文件
4. 删除数据库记录
```

---

## 架构特性

1. **客户端缓存**：OssFactory 使用 ConcurrentHashMap 缓存 OssClient，支持多租户隔离

2. **配置来源**：从 Redis 中读取（`CacheNames.SYS_OSS_CONFIG`），配置为 JSON 格式

3. **默认配置键**：从 Redis key `sys_oss:default_config` 获取

4. **S3 协议支持**：支持所有兼容 S3 协议的云服务（阿里云、腾讯云、七牛云、MinIO等）

5. **异步传输**：基于 AWS SDK v2 的异步客户端和 S3TransferManager

6. **路径风格**：MinIO 使用路径风格访问，云服务商使用虚拟主机风格

7. **私有桶处理**：私有桶的URL自动生成预签名URL（120秒过期）

8. **扩展字段**：使用 JSON 存储在 ext1 字段中，支持灵活扩展

---

## 相关文件位置

| 类型 | 位置 |
|------|------|
| OssFactory | `ruoyi-common/ruoyi-common-oss/src/main/java/org/dromara/common/oss/factory/OssFactory.java` |
| OssClient | `ruoyi-common/ruoyi-common-oss/src/main/java/org/dromara/common/oss/core/OssClient.java` |
| UploadResult | `ruoyi-common/ruoyi-common-oss/src/main/java/org/dromara/common/oss/entity/UploadResult.java` |
| SysOssController | `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/controller/system/SysOssController.java` |
| SysOssServiceImpl | `ruoyi-modules/ruoyi-system/src/main/java/org/dromara/system/service/impl/SysOssServiceImpl.java` |

---

## 前端文件上传与管理（plus-ui 存在时适用）

> 当 `plus-ui/` 目录存在时，文件上传不仅是后端 API，还需要前端组件配合实现完整的上传体验。

### 前端文件相关结构

```
plus-ui/src/
├── components/
│   ├── FileUpload/index.vue         # 文件上传组件（通用）
│   ├── ImageUpload/index.vue        # 图片上传组件（带预览）
│   └── ImagePreview/index.vue       # 图片预览组件
├── api/system/
│   ├── oss/
│   │   ├── index.ts                 # OSS 文件 API（上传/下载/删除）
│   │   └── types.ts                 # OSS 类型定义
│   └── ossConfig/
│       ├── index.ts                 # OSS 配置 API
│       └── types.ts                 # 配置类型定义
├── views/system/oss/
│   ├── index.vue                    # OSS 文件管理页面
│   └── config.vue                   # OSS 配置页面
└── plugins/
    └── download.ts                  # 文件下载插件
```

### 文件上传组件用法

#### FileUpload - 通用文件上传
```vue
<template>
  <!-- 基础用法 -->
  <file-upload v-model="form.fileOssId" />

  <!-- 自定义限制 -->
  <file-upload
    v-model="form.fileOssId"
    :limit="5"
    :file-size="10"
    :file-type="['doc', 'xls', 'pdf']"
  />
</template>
```

#### ImageUpload - 图片上传（带预览）
```vue
<template>
  <!-- 单图上传 -->
  <image-upload v-model="form.avatar" :limit="1" />

  <!-- 多图上传 -->
  <image-upload v-model="form.images" :limit="5" />
</template>
```

#### ImagePreview - 图片预览
```vue
<template>
  <!-- 图片预览（支持 ossId 或 URL） -->
  <image-preview :src="row.imageOssId" :width="60" :height="60" />
</template>
```

### 文件下载

```typescript
// 使用 download 插件
import { getCurrentInstance } from 'vue';

const { proxy } = getCurrentInstance() as ComponentInternalInstance;

// 通过 ossId 下载
proxy?.download.oss(ossId);

// 通过 URL 下载
proxy?.download.zip('/api/xxx/export', '导出文件.zip');
```

### 前端 OSS 参考代码

| 类型 | 位置 | 说明 |
|------|------|------|
| 文件上传组件 | `plus-ui/src/components/FileUpload/index.vue` | 通用文件上传 |
| 图片上传组件 | `plus-ui/src/components/ImageUpload/index.vue` | 图片上传+预览 |
| 图片预览组件 | `plus-ui/src/components/ImagePreview/index.vue` | 图片展示 |
| OSS API | `plus-ui/src/api/system/oss/index.ts` | 上传/下载/删除 |
| OSS 管理页面 | `plus-ui/src/views/system/oss/index.vue` | 文件列表管理 |
| OSS 配置页面 | `plus-ui/src/views/system/oss/config.vue` | 存储配置管理 |
| 下载插件 | `plus-ui/src/plugins/download.ts` | download.oss() 等 |
