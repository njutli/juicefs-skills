---
name: juicefs-chunk-cache
description: JuiceFS Chunk存储与缓存机制（双层缓存：Disk + Memory）。当用户询问ChunkStore、Page、磁盘缓存、内存缓存、Writeback、Prefetch、cache-dir目录结构时调用此技能。
---

# Chunk 存储与缓存机制

## 〇、为什么需要 Chunk 缓存层？

如果每次读写都直接访问远端对象存储（S3），延迟会非常差（几十到几百毫秒）。Chunk 缓存层在本地维护了**磁盘+内存双层缓存**，将热数据保留在本地，冷数据透明地从对象存储加载。

## 一、核心数据结构

### 1.1 ChunkStore 接口

```go
// pkg/chunk/chunk.go
type ChunkStore interface {
    NewReader(chunkid uint64, length int) Reader   // 创建 chunk 读取器
    NewWriter(chunkid uint64) Writer                // 创建 chunk 写入器
    Remove(chunkid uint64, length int) error        // 删除 chunk
    Flush(chunkid uint64) error                     // 刷脏数据到对象存储
    Stats() (totalBytes, usedBytes int64)            // 缓存统计
}
```

### 1.2 Page — 最小 I/O 单元

```go
// pkg/chunk/page.go
type Page struct {
    Chunkid uint64     // 所属 chunk
    Offset  int        // 在 chunk 内的偏移（必须是 PageSize 的倍数）
    Data    []byte     // 页面数据（64KB）
    Dirty   bool       // 是否为脏页（内存中修改了但未刷到磁盘）
    ...
}

const PageSize = 64 * 1024  // 64KB
```

### 1.3 CachedStore — 核心实现

```go
// pkg/chunk/cached_store.go
type cachedStore struct {
    sync.Mutex
    object     object.ObjectStorage   // 远端对象存储
    conf       Config
    
    // 双层缓存
    memCache   *memCache              // 内存页面缓存
    diskCache  *diskCache             // 本地磁盘缓存
    
    // 后台任务
    pending    chan string            // 待上传的 chunk 队列
    wg         sync.WaitGroup        // 等待后台 goroutine 结束
    
    // 限流
    uploadLimiter   *rate.Limiter    // 上传带宽限制
    downloadLimiter *rate.Limiter    // 下载带宽限制
}
```

### 1.4 磁盘缓存 — 4 层目录结构

```
cacheDir/
├── rawstaging/   ← 写缓冲区，待上传到对象存储
│   └── {chunkid}
├── rawslices/    ← 碎片数据（小于 4MB 或不完整的 block）
│   └── {chunkid}_{offset}_{length}
├── rawfull/      ← 完整的 4MB 数据块
│   └── {chunkid}_{blockIndex}
└── cached/       ← 对齐到 4MB 的 Block 对象（已压缩/加密）
    └── {chunkid}_{blockIndex}
```

### 1.5 内存缓存 — LRU

```go
// pkg/chunk/mem_cache.go
type memCache struct {
    sync.Mutex
    pages    map[string]*Page    // key = "{chunkid}_{pageIndex}"
    lru      *list.List          // LRU 淘汰链表
    capacity int64               // 最大容量（字节）
    used     int64               // 当前用量
}
```

## 二、核心流程

### 2.1 读流程（Read）

```
chunkStore.Read(chunkid, buf, offset, size)
  │
  ├─ compute page index: pageIdx = offset / PageSize (64KB)
  │
  ├─ for each page in range:
  │    │
  │    ├─ 1. memCache.lookup(chunkid, pageIdx)
  │    │     └─ 命中: 复制 Page.Data 到 buf，return
  │    │
  │    ├─ 2. diskCache.lookup(chunkid, pageIdx)
  │    │     ├─ 命中: 从磁盘读入 → 写入 memCache → 返回
  │    │     │
  │    │     └─ 未命中: 
  │    │         object.Get(chunkKey(chunkid, pageIdx))  # 从对象存储下载
  │    │           ├─ 下载 4MB 的 block
  │    │           ├─ 解压/解密（如果需要）
  │    │           ├─ 写入 diskCache（rawfull/）
  │    │           ├─ 裁剪需要的 64KB page → 写入 memCache
  │    │           └─ 返回
  │    │
  │    └─ 3. 如果对象存储也没有 → 返回全零（未写入的区域）
  │
  └─ 返回读取的字节数
```

### 2.2 写流程（Write）

```
chunkStore.Write(chunkid, buf, offset)
  │
  ├─ compute page index
  │
  ├─ for each page in range:
  │    │
  │    ├─ 查找或分配 Page
  │    │     ├─ memCache 有 → 使用已有 Page
  │    │     └─ memCache 没有 → 分配新 Page
  │    │
  │    ├─ 复制数据到 Page.Data
  │    ├─ 标记 Page.Dirty = true
  │    │
  │    └─ 如果 page 累计达到一个 block (4MB) 或超时：
  │         ├─ 将 Page 写入 diskCache (rawstaging/)
  │         ├─ 触发异步上传到对象存储
  │         │    go uploadToObjectStorage(chunkid, blockIndex, data)
  │         │      ├─ 压缩/加密
  │         │      ├─ object.Put(key, data)    # 上传到 S3
  │         │      ├─ 移动 rawstaging → rawfull（标记上传完成）
  │         │      └─ 可选：保留在 diskCache 一段时间（热数据）
  │         └─ Page.Dirty = false
  │
  └─ 返回写入的字节数
```

### 2.3 缓存淘汰（Eviction）

```go
// pkg/chunk/cache_eviction.go
func (c *cachedStore) evictionBackground() {
    for {
        time.Sleep(evictionInterval)
        
        totalUsed := c.diskCache.usedBytes()
        if totalUsed <= c.conf.CacheSize {
            continue
        }
        
        // FreeSpace 策略：确保磁盘剩余空间不低于配置的比例
        // 淘汰最久未访问的 chunk（LRU by atime）
        toEvict := totalUsed - c.conf.CacheSize * 0.9  // 清到 90%
        c.diskCache.evictLRU(toEvict)
    }
}
```

淘汰策略：
1. 检查缓存总大小是否超过 `--cache-size`
2. 检查磁盘剩余空间是否低于 `--free-space-ratio`
3. 按最后访问时间 (atime) 淘汰最旧的 chunk
4. **不删除**正在写入的 staging chunk
5. `--cache-evict-interval` 控制淘汰检查间隔

### 2.4 预取（Prefetch）

```go
// pkg/chunk/prefetch.go
// 顺序读取时，后台预取下一个 chunk
func (c *cachedStore) prefetch(chunkid uint64, index int) {
    nextChunkID := chunkid + 1
    // 异步从对象存储下载下一个 chunk
    go c.object.Get(chunkKey(nextChunkID, index), ...)
}
```

## 三、关键设计特性

### 3.1 上传时机

| 触发条件 | 说明 |
|---------|------|
| Page 累计满 128KB | 默认 writeback 大小，触发一次上传 |
| Chunk 关闭 | close() 或文件的最后一个 write |
| 超时 | 一定时间后自动上传未满的 block（默认 5s） |
| Fsync | 用户调用 fsync()，立即上传所有脏数据 |

### 3.2 磁盘缓存恢复

```go
// 启动时扫描已有缓存
func (c *diskCache) scanExisting() error {
    // 扫描 rawstaging/ — 未上传完成的 chunk
    // 扫描 rawslices/ — 碎片数据
    // 扫描 rawfull/   — 完整 blocks
    // 扫描 cached/    — 缓存的 blocks
    // 恢复缓存元数据（大小、访问时间）
}
```

如果发现 `rawstaging/` 中有未完成的上传，mount 后会**自动重新上传**。

### 3.3 并发控制

```go
// singleflight 模式 — 避免多个 reader 同时下载同一个 chunk
// pkg/chunk/singleflight.go
type call struct {
    wg  sync.WaitGroup
    val interface{}
    err error
}

type Group struct {
    mu sync.Mutex
    m  map[string]*call
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
    // 如果 key 已有进行中的调用，等待其完成并共享结果
    // 否则启动新调用
}
```

### 3.4 带宽限制

```go
// 使用 rate.Limiter 限制上传/下载速度
c.uploadLimiter.WaitN(ctx, n)    // 上传前等待令牌
c.downloadLimiter.WaitN(ctx, n)  // 下载前等待令牌

// 配置:
// --upload-limit=100    # 限制上传 100 Mbps
// --download-limit=200  # 限制下载 200 Mbps
```

## 四、关键代码位置

| 文件 | 内容 | 关键行/函数 |
|------|------|-----------|
| `pkg/chunk/chunk.go` | ChunkStore 接口 + Reader/Writer | `ChunkStore`, `Reader`, `Writer` |
| `pkg/chunk/cached_store.go` | 核心实现（1247 行） | `cachedStore`, `NewCachedStore`, `Read`, `Write`, `Remove` |
| `pkg/chunk/page.go` | Page 定义 | `Page{Chunkid, Offset, Data, Dirty}` |
| `pkg/chunk/disk_cache.go` | 磁盘缓存 | `diskCache`, `scanExisting`, `lookup`, `store` |
| `pkg/chunk/disk_cache_state.go` | 缓存状态管理 | 缓存元数据（大小、访问时间） |
| `pkg/chunk/mem_cache.go` | 内存 LRU 缓存 | `memCache`, `lookup`, `evict` |
| `pkg/chunk/cache_eviction.go` | 缓存淘汰 | `evictionBackground`, `evictLRU` |
| `pkg/chunk/prefetch.go` | 预取 | `prefetch` |
| `pkg/chunk/singleflight.go` | 去重并发读取 | `singleflight.Group` |
| `pkg/chunk/metrics.go` | Prometheus 指标 | 缓存命中率、上传/下载速率 |

## 五、调试与排查

### 5.1 查看缓存状态

```bash
# 查看缓存目录各子目录的大小
du -sh /var/jfsCache/*
# rawstaging/  — 如果很大说明上传卡住了
# rawfull/     — 本地热数据

# 查看 stats
cat /mnt/jfs/.stats | jq .

# 查看缓存的 block 数量
find /var/jfsCache/rawfull -type f | wc -l
```

### 5.2 检查上传是否正常

```bash
# 查看日志中的上传相关日志
grep "upload" /var/log/juicefs.log

# 检查 staging 目录中的文件是否在减少
watch -n 1 "ls /var/jfsCache/rawstaging/ | wc -l"

# 如果有文件长期在 staging/ 中，可能是对象存储连接问题
```

### 5.3 缓存命中率分析

```bash
# 使用 profile 命令分析
juicefs profile META-URL --uid=1000 --gid=1000 --pid=12345

# 查看 FUSE 请求的响应时间，低命中率时读延迟会显著升高
```

## 六、常见问题

**Q: 缓存可以跨多个 mount 实例共享吗？**
A: 如果多个 mount 使用相同的 `--cache-dir`，缓存可以共享。但要注意写冲突（两个实例同时写同一个 chunk）。

**Q: 缓存占满磁盘怎么办？**
A: 缓存淘汰会自动删除最旧的冷数据。设置 `--cache-size` 和 `--free-space-ratio` 可以有效控制。

**Q: 断电重启后 staging 中的数据会丢失吗？**
A: 不会。`rawstaging/` 中的数据会重新上传。但如果是在写入过程中断电，正在传输的 page 可能会丢失。

**Q: 为什么缓存目录占用比实际文件大？**
A: 因为缓存以 4MB block 为单位存储，小文件的最后一个 block 会有碎片空间。加上 `rawstaging/` 和 `rawslices/` 中的临时数据。

## 七、改进思路 / 贡献方向

1. **自适应预取** — 根据访问模式（顺序/随机）动态调整预取策略
2. **缓存分层** — 支持 NVMe + HDD 分层，热数据在 NVMe，冷数据在 HDD
3. **压缩缓存** — 对缓存中的 block 启用透明压缩（lz4/zstd）
4. **更好的写聚合** — 优化小文件写入的 block 填充，减少碎片
5. **Prometheus 指标增强** — 更详细的缓存分层统计
