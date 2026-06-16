---
name: juicefs-sync-engine
description: JuiceFS Sync数据同步引擎。当用户询问跨存储同步、多线程拷贝、断点续传、Checkpoint、分布式同步时调用此技能。
---

# Sync 数据同步引擎

## 〇、为什么需要 Sync 引擎？

JuiceFS 的 Sync 命令可以**在任何两个存储系统之间**高效同步数据：
- S3 → JuiceFS（迁移上云）
- JuiceFS → S3（备份到对象存储）
- HDFS → JuiceFS
- 本地文件系统 → JuiceFS
- S3 → S3

与传统 `rsync` 或 `aws s3 sync` 不同，JuiceFS Sync 支持多线程并发、Checkpoint 断点续传、分布式集群同步。

## 一、核心数据结构

### 1.1 Sync 配置

```go
// pkg/sync/sync.go
type Config struct {
    Threads    int           // 并发线程数
    Limit      int64         // 带宽限制（Mbps）
    Update     bool          // 是否覆盖目标已存在的文件
    Force      bool          // 强制覆盖
    Dry        bool          // 试运行（只列不传）
    DeleteSrc  bool          // 同步后删除源文件
    DeleteDst  bool          // 同步前清理目标多余文件
    Exclude    []string      // 排除模式（glob）
    Include    []string      // 包含模式（glob）
    Links      bool          // 是否复制符号链接
    Perms      bool          // 是否保留权限
    Manager    string        // 集群管理地址
    ...
}
```

### 1.2 Sync 任务

```go
// pkg/sync/sync.go
type Sync struct {
    src     object.ObjectStorage    // 源存储
    dst     object.ObjectStorage    // 目标存储
    config  *Config
    cluster *Cluster                 // 集群模式管理器
    ...
}
```

### 1.3 对象同步项

```go
type object struct {
    key   string          // 对象键/路径
    size  int64           // 对象大小
    mtime time.Time       // 修改时间
    isDir bool            // 是否目录
    isSym bool            // 是否符号链接
    ...
}
```

## 二、核心流程

### 2.1 同步主流程

```
sync(srcURL, dstURL)
  │
  ├─ 1. 创建源和目标 ObjectStorage
  │     src := object.CreateStorage(srcURL)
  │     dst := object.CreateStorage(dstURL)
  │
  ├─ 2. 列出源对象
  │     srcObjects := src.ListAll(prefix, marker)
  │
  ├─ 3. 应用 Exclude/Include 过滤
  │     filtered := applyFilters(srcObjects, config)
  │
  ├─ 4. 加载 Checkpoint（如果有）
  │     checkpoint := loadCheckpoint(config.CheckpointFile)
  │     filtered = filtered - checkpoint.done  // 减去已完成的
  │
  ├─ 5. 创建 worker 池
  │     jobs := make(chan *object, config.Threads * 2)
  │     for i := 0; i < config.Threads; i++ {
  │         go worker(jobs, src, dst, config, checkpoint)
  │     }
  │
  ├─ 6. 分发任务到 worker 池
  │     for _, obj := range filtered {
  │         jobs <- obj
  │     }
  │     close(jobs)
  │
  └─ 7. 等待所有 worker 完成
```

### 2.2 Worker 逻辑

```go
// pkg/sync/sync.go
func worker(jobs <-chan *object, src, dst ObjectStorage, config *Config, checkpoint *Checkpoint) {
    for obj := range jobs {
        // 1. 获取源对象信息
        srcObj, err := src.Head(obj.key)
        
        // 2. 检查是否需要同步
        dstObj, err := dst.Head(obj.key)
        if err == nil && !config.Update && !config.Force {
            if dstObj.Mtime == srcObj.Mtime && dstObj.Size == srcObj.Size {
                continue  // 相同，跳过
            }
        }
        
        // 3. 执行复制
        if obj.isDir {
            dst.Put(obj.key+"/", nil)  // 创建目录标记
        } else {
            // 小对象：直接复制
            reader, _ := src.Get(obj.key, 0, -1)
            dst.Put(obj.key, reader)
        }
        
        // 4. 记录 Checkpoint
        checkpoint.Done(obj.key)
        
        // 5. 如果需要，删除源对象
        if config.DeleteSrc {
            src.Delete(obj.key)
        }
    }
}
```

### 2.3 Checkpoint 断点续传

```go
// pkg/sync/checkpoint.go
type Checkpoint struct {
    sync.Mutex
    file   *os.File           // checkpoint 文件
    done   map[string]bool    // 已完成的对象键
}

func (c *Checkpoint) Done(key string) {
    c.Lock()
    defer c.Unlock()
    c.done[key] = true
    c.file.WriteString(key + "\n")  // 追加到文件
    c.file.Sync()
}

func (c *Checkpoint) IsDone(key string) bool {
    c.RLock()
    defer c.RUnlock()
    return c.done[key]
}
```

重启同步时，加载 checkpoint 文件，跳过已完成的 key。

### 2.4 集群模式

```go
// pkg/sync/cluster.go
type Cluster struct {
    nodes    []string               // 集群节点地址列表
    manager  string                 // 管理节点地址
    ...
}

// 集群模式下，多个节点并行同步，通过 manager 协调任务分配
func (c *Cluster) syncCluster(src, dst string, config *Config) error {
    // 1. 注册自身到 manager
    // 2. 从 manager 获取待同步对象批次
    // 3. 同步本批次
    // 4. 向 manager 报告进度
    // 5. 循环直到所有对象同步完成
}
```

任务分配策略：
- 按文件大小排序，大文件优先分配（避免小文件完成太快导致负载不均）
- 每个 worker 一次分配一个 batch（默认 1000 个对象）
- Manager 节点负责协调、汇总 checkpoint

## 三、关键设计特性

### 3.1 增量同步

```go
// 四种同步模式
if config.Force {
    // 强制覆盖所有文件
} else if config.Update {
    // 仅覆盖新文件（mtime 更新）和大小不同的文件
} else {
    // 跳过已存在且 mtime+size 相同的文件
}
```

### 3.2 Include/Exclude 过滤

```go
// 支持 glob 模式匹配
config.Exclude = []string{"*.tmp", ".git/", "node_modules/"}
config.Include = []string{"*.csv", "data/*.parquet"}

// Include 先于 Exclude 生效
```

### 3.3 大文件优化

```go
// 对大文件使用 Multipart Upload 并行上传
// 支持 UploadPartCopy，如果源和目标在同一 S3 后端，使用服务端复制避免下载
func syncLargeObject(src, dst ObjectStorage, key string, size int64) error {
    if dst.SupportsMultipartUpload() && src.SupportsUploadPartCopy() {
        // 服务端复制（零流量）
        parts := splitIntoParts(size, dst.Limits().MinPartSize)
        for _, part := range parts {
            dst.UploadPartCopy(key, uploadID, part.num, key, part.off, part.size)
        }
        dst.CompleteUpload(key, uploadID, parts)
    } else {
        // 正常分片上传
        ...
    }
}
```

## 四、关键代码位置

| 文件 | 内容 | 关键函数 |
|------|------|---------|
| `cmd/sync.go` | CLI 命令 | `sync()`, `syncAction()` |
| `pkg/sync/sync.go` | 同步核心 | `Sync`, `worker`, `syncObject` |
| `pkg/sync/checkpoint.go` | 断点续传 | `Checkpoint`, `Done`, `IsDone`, `loadCheckpoint` |
| `pkg/sync/cluster.go` | 集群同步 | `Cluster`, `syncCluster` |
| `pkg/sync/download.go` | 下载管理 | `DownloadManager` |

## 五、调试与排查

### 5.1 试运行

```bash
# --dry 参数：只列出要同步的文件，不实际传输
juicefs sync s3://source/ redis://localhost/1/vol/ --dry

# 输出：
# Found: 1000 objects
# To sync: 500 objects (new), 200 objects (updated)
# Total size: 10.5 GB
```

### 5.2 监控同步进度

```bash
# 使用 --progress 显示进度条
juicefs sync s3://src/ redis://localhost/1/dst/ --progress

# 使用 --verbose 查看详细日志
juicefs sync s3://src/ redis://localhost/1/dst/ --verbose

# Checkpoint 文件位置
# 默认: ./juicefs-sync.checkpoint
# 可自定义: --checkpoint /path/to/cp
```

### 5.3 排查同步失败

```bash
# 1. 检查源和目标存储是否可访问
juicefs objbench --storage s3 s3://source/
juicefs objbench --storage redis redis://localhost/1

# 2. 使用 --dry 查看哪些文件需要同步
# 3. 先用单线程(--threads 1) 测试
# 4. 开启 --verbose 看详细错误
```

## 六、常见问题

**Q: Sync 如何保证不丢数据？**
A: 每个文件写入完成后记录 checkpoint，中断后重启从 checkpoint 继续。如果写入过程中中断，目标上的残缺文件会在下一次同步时被 Force 覆盖。

**Q: 大量小文件同步慢怎么办？**
A: 增加 `--threads`（并发数），减小 `--bwlimit` 的约束。小文件的瓶颈通常是 List 操作和元数据操作延迟，增加线程可以并发处理。

**Q: Sync 支持定时任务吗？**
A: 不支持内置定时器。可以使用 cron/systemd timer 定期执行 sync 命令。

## 七、改进思路 / 贡献方向

1. **增量 Checkpoint** — 支持只 checkpoint 修改过的文件，减少 checkpoint 文件大小
2. **速率自适应** — 根据对象存储的响应时间动态调整并发数
3. **校验和验证** — 同步后验证源和目标的校验和一致性
4. **事件触发同步** — 监听文件系统事件（inotify）触发增量同步
