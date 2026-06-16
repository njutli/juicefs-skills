---
name: juicefs-gateway
description: JuiceFS S3 Gateway实现，基于MinIO的S3兼容API网关。当用户询问gateway、S3 API、MinIO集成、多用户权限、对象存储转文件系统时调用此技能。
---

# S3 Gateway 实现

## 〇、为什么需要 S3 Gateway？

JuiceFS 是一个 POSIX 文件系统，但很多应用（如 Spark、Flink、AI 训练框架）习惯使用 S3 API 访问数据。Gateway 将 JuiceFS 暴露为 S3 兼容的 HTTP API，让这些应用无需修改代码就能使用 JuiceFS。

```
S3 客户端 (awscli, boto3, Spark...)
  │
  ▼  HTTP REST API (S3 协议)
  │
JuiceFS Gateway
  ├── MinIO Gateway 框架
  ├── S3 → POSIX 语义映射
  └── pkg/fs/ — Go 原生文件系统接口
       │
       ▼
     VFS → Meta + ChunkStore → 对象存储
```

## 一、核心数据结构

### 1.1 Gateway 服务

```go
// pkg/gateway/gateway.go
type juiceFSGateway struct {
    fs       *fs.FileSystem        // pkg/fs/ 的文件系统实现
    conf     *gatewayConfig
    ...
}
```

### 1.2 pkg/fs/ — Go 原生文件系统（不经过 FUSE）

Gateway 不使用 FUSE，而是通过 `pkg/fs/` 直接操作 VFS：

```go
// pkg/fs/fs.go
type FileSystem struct {
    vfs  *vfs.VFS
    conf *Config
}

// pkg/fs/ 提供的 API：
func (fs *FileSystem) Open(ctx, path, flags, mode) (*File, error)
func (fs *FileSystem) Create(ctx, path, mode) (*File, error)
func (fs *FileSystem) Read(ctx, f *File, buf []byte, off int64) (int, error)
func (fs *FileSystem) Write(ctx, f *File, buf []byte, off int64) (int, error)
func (fs *FileSystem) Mkdir(ctx, path, mode) error
func (fs *FileSystem) Remove(ctx, path) error
func (fs *FileSystem) Rename(ctx, oldPath, newPath) error
func (fs *FileSystem) Stat(ctx, path) (*Stat, error)
func (fs *FileSystem) List(ctx, prefix, marker, delimiter, limit) ([]*DirEntry, error)
...
```

### 1.3 MinIO Gateway 模式

JuiceFS 通过实现 MinIO 的 Gateway 接口来复用 MinIO 的 HTTP 服务器、认证、S3 XML 编解码：

```go
// MinIO Gateway 接口（简化）
type Gateway interface {
    Name() string
    NewGatewayLayer(creds auth.Credentials) (ObjectLayer, error)
    Production() bool
}

// ObjectLayer 接口（类似 S3 API 的底层抽象）
type ObjectLayer interface {
    // Bucket 操作
    MakeBucketWithLocation(ctx, bucket, opts) error
    GetBucketInfo(ctx, bucket) (BucketInfo, error)
    ListBuckets(ctx) ([]BucketInfo, error)
    DeleteBucket(ctx, bucket, opts) error
    
    // Object 操作
    GetObject(ctx, bucket, object, startOffset, length, ...) (*ObjectReader, error)
    PutObject(ctx, bucket, object, data, opts) (ObjectInfo, error)
    CopyObject(ctx, srcBucket, srcObject, dstBucket, dstObject, ...) (ObjectInfo, error)
    DeleteObject(ctx, bucket, object, opts) error
    ListObjects(ctx, bucket, prefix, marker, delimiter, maxKeys) (ListObjectsInfo, error)
    
    // Multipart 上传
    NewMultipartUpload(ctx, bucket, object, opts) (string, error)
    PutObjectPart(ctx, bucket, object, uploadID, partID, data, opts) (PartInfo, error)
    CompleteMultipartUpload(ctx, bucket, object, uploadID, parts, opts) (ObjectInfo, error)
    AbortMultipartUpload(ctx, bucket, object, uploadID, opts) error
    ...
}
```

## 二、核心流程

### 2.1 S3 Bucket → JuiceFS 目录映射

```
S3 概念                JuiceFS 映射
─────────────────────  ─────────────────────
Bucket                / 目录下的一个子目录
Object Key           该目录下的文件路径
Prefix               目录前缀
Delimiter            路径分隔符 (/)
Multipart Upload     文件的部分写入
```

### 2.2 GetObject 流程

```go
// pkg/gateway/gateway.go (简化)
func (j *juiceFSGateway) GetObject(ctx, bucket, object string, startOffset, length int64, ...) (*ObjectReader, error) {
    // 1. S3 key → JuiceFS 路径
    path := filepath.Join("/", bucket, object)
    // 例如: s3://mybucket/data/file.csv → /mybucket/data/file.csv
    
    // 2. 通过 pkg/fs/ 打开文件
    f, err := j.fs.Open(ctx, path, os.O_RDONLY, 0444)
    
    // 3. 范围读取（支持 S3 Range 请求）
    if startOffset >= 0 && length >= 0 {
        // S3 Range: bytes=startOffset-startOffset+length-1
        buf := make([]byte, length)
        j.fs.Read(ctx, f, buf, startOffset)
        return &ObjectReader{Data: buf, ...}
    }
    
    // 4. 完整读取
    stat, _ := j.fs.Stat(ctx, path)
    buf := make([]byte, stat.Size)
    j.fs.Read(ctx, f, buf, 0)
    return &ObjectReader{Data: buf, Size: stat.Size, ...}
}
```

### 2.3 PutObject 流程

```go
func (j *juiceFSGateway) PutObject(ctx, bucket, object string, data *PutObjReader, opts) (ObjectInfo, error) {
    // 1. S3 key → JuiceFS 路径
    path := filepath.Join("/", bucket, object)
    
    // 2. 确保父目录存在
    parentDir := filepath.Dir(path)
    j.fs.MkdirAll(ctx, parentDir, 0755)
    
    // 3. 创建/截断文件
    f, err := j.fs.Create(ctx, path, 0644)
    
    // 4. 写入数据
    reader := data.Reader
    buf := make([]byte, 4*1024*1024)  // 4MB buffer
    for {
        n, _ := reader.Read(buf)
        if n == 0 { break }
        j.fs.Write(ctx, f, buf[:n], off)
        off += int64(n)
    }
    
    // 5. 关闭文件（触发 flush）
    j.fs.Close(ctx, f)
    
    return ObjectInfo{Name: path, Size: off, ...}
}
```

### 2.4 Multi-User 权限隔离

```go
// 每个 S3 访问密钥（Access Key）映射到一个 JuiceFS 用户
// Gateway 根据 AK/SK 确定操作的 UID/GID
func (j *juiceFSGateway) authenticate(accessKey, secretKey string) (uid, gid uint32, err error) {
    // 1. 从 MinIO 的 IAM 系统中查找用户
    // 2. 或从配置文件中读取 AK/SK → UID/GID 映射
    // 3. 将 UID/GID 注入到所有文件操作的 ctx 中
}

// 文件权限检查由 JuiceFS VFS 层完成
// 使用 ctx.Uid()/ctx.Gid() 做 POSIX 权限检查
```

## 三、关键设计特性

### 3.1 不通过 FUSE

Gateway 直接使用 `pkg/fs/` 的 Go 原生文件系统 API，避免了 FUSE 的内核态/用户态切换开销，且不要求挂载点：

```
Gateway 路径:  S3 Client → HTTP → MinIO → pkg/fs/ → VFS → Meta + Chunk
FUSE 路径:    应用 → syscall → FUSE 内核 → pkg/fuse/ → VFS → Meta + Chunk

Gateway 路径更短，但因为 HTTP 协议的开销，延迟反而更高。
```

### 3.2 S3 协议兼容性

Gateway 支持以下 S3 API（部分列表）：
- Bucket: CreateBucket, ListBuckets, DeleteBucket, HeadBucket
- Object: GetObject, PutObject, DeleteObject, CopyObject, HeadObject
- List: ListObjects, ListObjectsV2
- Multipart: CreateMultipartUpload, UploadPart, CompleteMultipartUpload, AbortMultipartUpload
- ACL: GetBucketACL, PutBucketACL（有限支持）

### 3.3 性能考虑

| 操作 | 性能特征 |
|------|---------|
| GetObject（小文件） | 1 次 stat + 1 次 read，缓存命中时很快 |
| GetObject（大文件） | 多次 Chunk 读取，可能触发对象存储下载 |
| PutObject（小文件） | 简单，直接写入缓存 |
| PutObject（大文件） | 分片上传转换为多次 write，后台异步上传到对象存储 |
| ListObjects | 等价于遍历目录，大量对象时较慢 |

### 3.4 启动命令

```bash
juicefs gateway redis://localhost/1 localhost:9000 \
  --access-key=myak \
  --secret-key=mysk
```

## 四、关键代码位置

| 文件 | 内容 | 关键区域 |
|------|------|---------|
| `cmd/gateway.go` | Gateway CLI 命令 | `gateway()`, `gatewayAction()` |
| `pkg/gateway/gateway.go` | Gateway 核心 | `juiceFSGateway`, GetObject, PutObject, DeleteObject, ListObjects |
| `pkg/fs/fs.go` | Go 原生文件系统（1611 行） | `Open`, `Create`, `Read`, `Write`, `Mkdir`, `Remove`, `Stat`, `List` |
| `pkg/fs/http.go` | HTTP 文件处理器 | Web 访问接口 |

## 五、调试与排查

### 5.1 测试 S3 Gateway

```bash
# 启动 Gateway
juicefs gateway redis://localhost/1 localhost:9000 \
  --access-key=testak --secret-key=testsk

# 使用 awscli 测试
aws configure set aws_access_key_id testak
aws configure set aws_secret_access_key testsk
aws --endpoint-url http://localhost:9000 s3 mb s3://mybucket
aws --endpoint-url http://localhost:9000 s3 cp file.txt s3://mybucket/
aws --endpoint-url http://localhost:9000 s3 ls s3://mybucket/
```

### 5.2 启用访问日志

```bash
# Gateway 支持 S3 标准的 access log
juicefs gateway ... --access-log /var/log/juicefs-gateway.log
```

## 六、常见问题

**Q: Gateway 和直接 mount 哪个更快？**
A: Mount (FUSE) 更快。Gateway 多了 HTTP 协议开销（Header 解析、XML 编解码、认证），延迟更高约 2-10ms。

**Q: Gateway 支持多用户吗？**
A: 通过不同的 AK/SK 和 UID/GID 映射可以实现多用户权限隔离。每个 AK/SK 对应一个 JuiceFS 用户，文件权限由 JuiceFS 的 POSIX 权限控制。

**Q: 为什么 Gateway 基于 MinIO？**
A: MinIO 提供了成熟的 S3 协议实现（HTTP 服务器、认证、XML 编解码），JuiceFS 只需要实现 Gateway 插件接口，复用 MinIO 的 HTTP 层。

## 七、改进思路 / 贡献方向

1. **支持 S3 Select** — 服务端过滤数据，减少传输量
2. **支持 S3 Object Lock** — 合规场景的 WORM 存储
3. **支持 Versioning** — 利用 JuiceFS 的快照能力实现 S3 版本控制
4. **支持 Lifecycle** — 将对象自动转移到冷存储
5. **性能优化** — ListObjects 的目录遍历优化
