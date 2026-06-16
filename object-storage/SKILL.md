---
name: juicefs-object-storage
description: JuiceFS对象存储层，ObjectStorage接口与30+后端实现。当用户询问ObjectStorage接口、S3后端、工厂模式、Encrypt/Sharding装饰器、对象键命名规则时调用此技能。
---

# ObjectStorage 对象存储层

## 〇、为什么需要对象存储抽象层？

JuiceFS 的数据存哪里？答案是**任何支持 S3 兼容 API 的存储系统**。但不同后端（S3、MinIO、OSS、COS、GCS...）的 API 有差异。ObjectStorage 接口提供了统一的抽象，让 JuiceFS 核心代码不关心底层是哪个对象存储。

## 一、核心接口

### 1.1 ObjectStorage 接口

```go
// pkg/object/interface.go
type ObjectStorage interface {
    // 元信息
    String() string                         // 描述字符串（用于日志）
    Limits() Limits                         // 能力限制（最大对象大小、是否支持 multipart...）
    Create() error                          // 初始化（如创建 Bucket）
    
    // 基本操作
    Head(key string) (Object, error)        // 获取对象元信息
    Get(key string, off, limit int64) (io.ReadCloser, error)  // 读取对象（支持范围读取）
    Put(key string, in io.Reader) error     // 上传对象
    Copy(dst, src string) error             // 复制对象
    Delete(key string) error                // 删除对象
    
    // 列表操作
    List(prefix, marker, delimiter string, limit int64, followLink bool) ([]Object, error)
    ListAll(prefix, marker string) (<-chan Object, error)  // 列出所有对象（channel 异步）
    
    // 分片上传（大对象）
    CreateMultipartUpload(key string) (*MultipartUpload, error)
    UploadPart(key string, uploadID string, num int, data []byte) (*Part, error)
    UploadPartCopy(key string, uploadID string, num int, srcKey string, off, size int64) (*Part, error)
    AbortUpload(key string, uploadID string)
    CompleteUpload(key string, uploadID string, parts []*Part) error
    ListUploads(marker string) ([]*PendingPart, string, error)
    
    // 恢复（支持版本管理的存储，如 S3 版本控制）
    Restore(key string, days int64) error   // 从 Glacier/Deep Archive 恢复
}
```

### 1.2 Limits — 存储能力

```go
type Limits struct {
    IsSupportMultipartUpload bool    // 是否支持分片上传
    IsSupportUploadPartCopy  bool    // 是否支持服务端复制分片
    MinPartSize              int64   // 最小分片大小
    MaxPartSize              int64   // 最大分片大小
    MaxPartCount             int     // 最大分片数量
}
```

### 1.3 Object — 对象元信息

```go
type Object struct {
    Key          string    // 对象键
    Size         int64     // 对象大小（字节）
    Mtime        time.Time // 修改时间
    IsDir        bool      // 是否为"目录"（对象存储中的伪目录）
}
```

## 二、工厂模式 — CreateStorage

### 2.1 注册表

```go
// pkg/object/object_storage.go
var storages = make(map[string]ObjectStorage)

// 每个后端在 init() 中注册自己
func Register(name string, s ObjectStorage) {
    storages[name] = s
}
```

### 2.2 后端注册示例（S3）

```go
// pkg/object/s3.go
func init() {
    Register("s3", &s3Client{})
    Register("minio", &minioClient{})     // MinIO 是 S3 的子类型
    Register("wasabi", &wasabiClient{})   // Wasabi 也是
    Register("b2", &b2Client{})           // Backblaze B2（S3 兼容）
    // ... 更多
}
```

### 2.3 CreateStorage 流程

```go
// pkg/object/object_storage.go
func CreateStorage(uri string) (ObjectStorage, error) {
    u, err := url.Parse(uri)
    
    // 1. 从注册表查找后端
    s, ok := storages[u.Scheme]
    if !ok {
        return nil, fmt.Errorf("unsupported scheme: %s (supported: %v)", u.Scheme, supportedSchemes())
    }
    
    // 2. 创建基础后端实例
    obj, err := s.Create(uri)
    
    // 3. 【装饰器1】Sharding — 如果配置了分区，包装 Sharding 层
    if shards > 1 {
        obj = &shardedClient{obj, shards}
    }
    
    // 4. 【装饰器2】Encrypt — 如果配置了加密，包装加密层
    if encryptKey != "" {
        obj = &encryptedClient{obj, encryptKey}
    }
    
    return obj, nil
}
```

**装饰器模式**：Sharding 和 Encrypt 是透明的包装层。Sharding 将 key 哈希分散到多个子目录避免单目录热点；Encrypt 透明加解密数据。

## 三、S3 后端实现详解

### 3.1 s3Client 结构

```go
// pkg/object/s3.go
type s3Client struct {
    bucket     string
    client     *s3.S3           // AWS SDK v1/v2 S3 客户端
    endpoint   string           // S3 兼容端点（如 http://minio:9000）
    region     string
    ...
}
```

### 3.2 Put — 上传对象

```go
func (s *s3Client) Put(key string, in io.Reader) error {
    // 小对象（< MinPartSize）使用简单上传
    if size <= s.limits.MinPartSize {
        _, err := s.client.PutObject(&s3.PutObjectInput{
            Bucket: &s.bucket,
            Key:    &key,
            Body:   in,
        })
        return err
    }
    
    // 大对象使用分片上传
    upload, _ := s.CreateMultipartUpload(key)
    parts := []*Part{}
    
    buf := make([]byte, s.conf.BlockSize)  // 4MB
    for partNum := 1; ; partNum++ {
        n, _ := in.Read(buf)
        if n == 0 { break }
        part, _ := s.UploadPart(key, upload.UploadID, partNum, buf[:n])
        parts = append(parts, part)
    }
    
    return s.CompleteUpload(key, upload.UploadID, parts)
}
```

### 3.3 Get — 范围读取（JuiceFS 的核心优化）

```go
func (s *s3Client) Get(key string, off, limit int64) (io.ReadCloser, error) {
    var rang string
    if limit > 0 {
        rang = fmt.Sprintf("bytes=%d-%d", off, off+limit-1)
    }
    
    resp, err := s.client.GetObject(&s3.GetObjectInput{
        Bucket: &s.bucket,
        Key:    &key,
        Range:  &rang,           // S3 范围读取 —— 只下载需要的字节
    })
    return resp.Body, nil
}
```

> **关键优化**：JuiceFS 的 Chunk/Block 设计就是为了利用 S3 的 `Range` 请求。读取一个 4KB 的数据块只需下载对应的 4MB block 中的 64KB page 范围（如果 page 不在本地缓存中），而不是下载整个 block。

## 四、对象键（Key）命名规则

JuiceFS 在对象存储中的对象键格式：

```
chunks/{partition}/{chunkid}_{blockIndex}_{compressed}_{encrypted}
```

例如：
```
chunks/0/100_0_0_0    → chunkid=100, blockIndex=0, 未压缩, 未加密
chunks/3/200_5_1_1    → chunkid=200, blockIndex=5, 压缩, 加密
```

Sharding 模式下：
```
chunks/0/100_0_0_0
chunks/1/100_1_0_0    ← 同一个 chunk 的不同 block 分散在不同 shard
```

## 五、支持的后端速查

| 类别 | 后端 | Scheme | 文件 |
|------|------|--------|------|
| S3 兼容 | AWS S3 | `s3` | `s3.go` |
| S3 兼容 | MinIO | `minio` | `minio.go` |
| S3 兼容 | Wasabi | `wasabi` | `wasabi.go` |
| S3 兼容 | B2 | `b2` | `b2.go` |
| 云厂商 | 阿里云 OSS | `oss` | `oss.go` |
| 云厂商 | 腾讯云 COS | `cos` | `cos.go` |
| 云厂商 | 华为云 OBS | `obs` | `obs.go` |
| 云厂商 | 百度云 BOS | `bos` | `bos.go` |
| 云厂商 | 火山引擎 TOS | `tos` | `tos.go` |
| 云厂商 | Google GCS | `gs` | `gs.go` |
| 云厂商 | Azure Blob | `azure` | `azure.go` |
| Ceph | Ceph RGW | `ceph` | `ceph.go` |
| 本地 | Filesystem | `file` | `file.go` |
| 网络 | SFTP | `sftp` | `sftp.go` |
| 网络 | WebDAV | `webdav` | `webdav.go` |
| 网络 | HDFS | `hdfs` | `hdfs.go` |
| 内存 | Memory (测试) | `mem` | `mem.go` |

## 六、关键代码位置

| 文件 | 内容 |
|------|------|
| `pkg/object/interface.go` | ObjectStorage 接口 + Object/Limits 类型定义 |
| `pkg/object/object_storage.go` | CreateStorage 工厂 + Register 注册表 |
| `pkg/object/s3.go` | AWS S3 + MinIO + 大部分 S3 兼容后端的基类 |
| `pkg/object/encrypt.go` | 加密装饰器（AES-CTR） |
| `pkg/object/encrypt_chunked.go` | 分块加密装饰器 |
| `pkg/object/sharding.go` | 分片装饰器（key hash 分散） |
| `pkg/object/prefix.go` | 前缀装饰器 |
| `pkg/object/file.go` | 本地文件系统后端 |
| `pkg/object/checksum.go` | 对象校验和 |

## 七、调试与排查

### 7.1 测试对象存储连接

```bash
# 使用 objbench 测试
juicefs objbench --storage s3 \
  --access-key=xxx --secret-key=yyy \
  https://s3.amazonaws.com/mybucket

# 输出: 读写 IOPS, 带宽, 延迟
```

### 7.2 列出对象存储中的对象

```bash
# 使用 awscli 直接看
aws s3 ls s3://mybucket/chunks/ --endpoint-url https://...
aws s3 ls s3://mybucket/chunks/0/ | head -20  # 看前 20 个对象
```

### 7.3 排查上传失败

```bash
# 开启 debug 日志
juicefs mount --verbose redis://localhost/1 /mnt/jfs 2>&1 | grep -i "upload\|put\|s3"

# 常见问题：
# - AccessDenied: 检查 access key 的权限（至少需要 s3:GetObject, s3:PutObject, s3:DeleteObject, s3:ListBucket）
# - NoSuchBucket: 检查 bucket 是否存在，或 endpoint 是否正确
# - Timeout: 检查网络连通性和 endpoint 可达性
```

## 八、常见问题

**Q: 如何添加新的对象存储后端？**
A: 创建新文件，实现 ObjectStorage 接口，在 `init()` 中调用 `Register("scheme", &yourClient{})`。

**Q: 加密装饰器怎么工作？**
A: `encryptedClient` 实现了 ObjectStorage，包装了另一个 ObjectStorage。Put 时先加密再调用底层 Put，Get 时先从底层读取再解密。

**Q: Sharding 有什么作用？**
A: 将对象按 key 哈希分散到 N 个 "子目录"中，避免单目录对象数过多导致 List 操作变慢。类似 S3 的最佳实践。

## 九、改进思路 / 贡献方向

1. **添加存储后端** — 为 JuiceFS 贡献新的对象存储驱动（如支持某个云厂商的存储）
2. **渐进式下载** — 对大 block 下载使用 Range 请求渐进获取
3. **自动故障转移** — 主对象存储故障时自动切换到备用
4. **多级对象存储** — 热数据在 S3 Standard，冷数据自动转移到 Glacier
