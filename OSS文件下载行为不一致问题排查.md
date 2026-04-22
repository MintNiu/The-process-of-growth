# OSS 文件下载行为不一致问题排查记录

## 📋 问题描述

**现象**：同一份代码，上传同一份 PDF 文件到两个 OSS Bucket：
- **测试环境**（`xxx-test-bucket`）：点击文件链接 → **自动下载** ✅
- **正式环境**（`xxx-bucket`）：点击文件链接 → **浏览器直接预览** ❌

**影响**：正式环境用户无法直接下载文件，需要先预览再手动保存，体验不佳。

---

## 🔍 排查过程

### 1. 初步怀疑：CDN 缓存问题

**发现**：
- 正式环境响应头包含 `Server: cloudflare` 和 `Cf-Cache-Status: HIT`
- 怀疑 Cloudflare CDN 缓存了不带 `Content-Disposition` 的旧版本

**排查**：
- 按 `Ctrl+F5` 强制刷新，清除 CDN 缓存
- 后续确认两个环境都直连 OSS（`Server: AliyunOSS`）

**结论**：❌ 问题不在 CDN

---

### 2. 怀疑：OSS Bucket 配置差异

**方法**：通过 OSS SDK 编写对比接口，逐项对比两个 Bucket 的配置

**对比结果**：

| 配置项 | 测试环境 | 正式环境 | 是否一致 |
|--------|---------|---------|---------|
| `storageClass` | Standard | Standard | ✅ |
| `versioning` | Enabled | Enabled | ✅ |
| `dataRedundancyType` | LRS | LRS | ✅ |
| `location` | oss-ap-southeast-1 | oss-ap-southeast-1 | ✅ |
| `acl` | public-read | public-read | ✅ |
| `Owner ID` | xxx | xxx | ✅ |

**唯一差异**：生命周期规则（测试有，正式无）

**结论**：❌ Bucket 配置层面无任何导致下载行为差异的设置

---

### 3. 怀疑：文件元数据差异

**方法**：在 `AliOssUtil.putFile()` 中增加上传后立即检查元数据的日志

**对比结果**：

```
测试环境：Content-Type: application/pdf, Content-Disposition: null
正式环境：Content-Type: application/pdf, Content-Disposition: null
```

**结论**：❌ 两个 Bucket 的文件元数据**完全一致**，都没有设置 `Content-Disposition`

---

### 4. 怀疑：代码版本不一致

**排查**：
- 确认测试和正式环境使用同一份代码
- 确认都是新上传的文件（排除历史元数据影响）

**结论**：❌ 代码完全一致

---

## 🎯 根本原因

**浏览器对 `Content-Type: application/pdf` 且无 `Content-Disposition` 的文件，在不同域名下有默认行为的不确定性。**

| 环境 | 域名                                                | 浏览器默认行为 |
|------|---------------------------------------------------|--------------|
| 测试 | `xxx-test-bucket.oss-ap-southeast-1.aliyuncs.com` | 下载 ✅ |
| 正式 | `oss.xxx.com`                                     | 预览 ❌ |

**这不是 OSS 的 bug，也不是代码的问题，而是浏览器行为的不确定性。**

- 不同浏览器版本、不同域名、不同文件大小等因素都可能影响默认行为
- 无法通过配置 OSS 来保证一致性
- 唯一可控的方式是在代码中明确告知浏览器行为

---

## ✅ 解决方案

### 修改点 1：上传时设置强制下载元数据

**文件**：`AliOssUtil.java`

```java
ObjectMetadata meta = new ObjectMetadata();
// 上传时主动设置 Content-Disposition，明确告诉浏览器"我要下载"
meta.setContentDisposition("attachment; filename=\"" + new File(filePath).getName() + "\"");
uploadFileRequest.setObjectMetadata(meta);
```

**效果**：所有新上传的文件都会自动携带 `Content-Disposition: attachment`，浏览器强制下载。

---

### 修改点 2：下载接口增加 `forceDownload` 参数

**文件**：`DownFileUtil.java`

- 添加 `forceDownload` 参数，允许调用方控制是否强制下载
- 当 `forceDownload=true` 时，在返回 URL 前同步更新 OSS 元数据
- 确保第一次访问即生效，无需等待缓存刷新

```java
public static Map<String, Object> getDownloadUrl(String filePath, boolean forceDownload) {
    // ... 原有逻辑 ...
    if (forceDownload && downloadUrl != null) {
        updateOssMetadataSync(downloadUrl, fileName);
    }
    // ...
}
```

---

### 修改点 3：Controller 层调用时传入 `forceDownload=true`

**文件**：`DownloadController.java`、`ViewController.java`

```java
@GetMapping("/download")
public ResponseEntity<Map<String, Object>> getDownloadUrl(@RequestParam("file") String file) {
    Map<String, Object> downloadUrl = DownFileUtil.getDownloadUrl(file, true);
    // ...
}
```

---

## 📦 涉及文件清单

| 文件路径 | 修改内容 |
|---------|---------|
| `plugins/aliyunOss/src/main/java/com/bit/aliyunoss/AliOssUtil.java` | 1. 上传时设置 `Content-Disposition: attachment`<br>2. 增加 `getBucketConfig()` 和 `compareBucketConfigs()` 方法<br>3. 增加上传后元数据检查日志 |
| `sys/src/main/java/com/bit/sys/common/utils/DownFileUtil.java` | 1. `getDownloadUrl()` 增加 `forceDownload` 参数<br>2. 增加 `updateOssMetadataSync()` 同步更新元数据<br>3. 移除基于 URL 的缓存逻辑，改为每次检查真实元数据 |
| `sys/src/main/java/com/bit/sys/controller/DownloadController.java` | 1. 调用 `getDownloadUrl()` 时传入 `true`<br>2. 增加 `/compareBuckets` 接口用于对比 Bucket 配置 |

---

## 🛠️ 测试验证

### 1. 新建文件验证

- 上传新文件到正式 OSS
- 通过 `/download` 接口获取链接
- 浏览器点击链接 → **自动下载** ✅

### 2. 历史文件验证

- 通过 `/download` 接口获取历史文件链接
- 接口会同步更新元数据，然后返回链接
- 浏览器点击链接 → **自动下载** ✅

### 3. Rebuild PDF 场景验证

- 通过 Rebuild PDF 接口生成的 PDF
- **不经过** `/download` 接口
- 浏览器点击链接 → **正常预览** ✅（符合业务需求）

---

## 💡 经验总结

### 1. 不要依赖浏览器的默认行为

即使 OSS 配置和文件元数据完全相同，浏览器对不同域名的默认行为可能不同。**对于需要确定下载/预览行为的场景，必须主动设置 `Content-Disposition` 响应头。**

### 2. 系统性排查方法

按照从外到内、从配置到代码的顺序逐层排查：

```
CDN/网关层 → OSS Bucket 配置 → 文件元数据 → 代码逻辑
```

避免盲目修改，每次排查都要有明确的假设和验证方法。

### 3. SDK 版本兼容性

阿里云 OSS SDK 3.17.4 的部分方法签名与文档不完全一致：

- `getBucketVersioning()` 返回 `BucketVersioningConfiguration` 而非 `String`
- 某些方法需要通过 IDE 提示确认正确的返回值类型
- 建议使用 `try-catch` 包裹 SDK 调用，避免运行时错误

### 4. 元数据更新的同步 vs 异步

- **异步更新**：不阻塞接口返回，但浏览器可能访问到旧元数据
- **同步更新**：确保元数据更新完成后再返回 URL，但接口响应时间略长
- **决策**：下载场景选择同步更新，确保第一次访问即生效

---

## 🔗 相关资源

- [阿里云 OSS 文档 - Object 元数据](https://help.aliyun.com/document_detail/31977.html)
- [RFC 6266 - Content-Disposition](https://datatracker.ietf.org/doc/html/rfc6266)
- [浏览器处理 PDF 的行为差异](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition)

---

## 📝 变更记录

| 日期 | 版本 | 变更内容 | 负责人     |
|------|------|---------|---------|
| 2026-04-22 | v1.0 | 初始版本，记录问题排查过程和解决方案 | MintNiu |

---

**问题状态**：✅ 已解决
