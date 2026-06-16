---
name: juicefs-vfs-layer
description: JuiceFS VFS虚拟文件系统层实现，连接FUSE与Meta/Chunk层的核心桥梁。当用户询问VFS、Lookup/Read/Write实现、Handle管理、内部文件（.control/.stats）时调用此技能。
---

# VFS 虚拟文件系统层

## 〇、为什么需要 VFS 层？

VFS 是 JuiceFS 架构中的**核心大脑**。FUSE 层负责接收内核请求，Meta 层负责元数据操作，Chunk 层负责数据缓存，而 VFS 负责**编排这三者的协作**：

```
FUSE 请求 → VFS → Meta(元数据) + Chunk(数据缓存)
```

VFS 的职责：
1. 将 POSIX 语义翻译为 Meta 接口调用 + Chunk 读写
2. 管理文件句柄（打开的文件/目录状态）
3. 处理内部虚拟文件（.control, .stats, .config, .accesslog, .trash）
4. 属性/目录项缓存管理
5. 访问日志记录

## 一、核心数据结构

### 1.1 VFS 结构体

```go
// pkg/vfs/vfs.go
type VFS struct {
    reader      meta.Meta          // 元数据引擎（只读路径）
    writer      meta.Meta          // 元数据引擎（读写路径，通常 == reader）
    store       chunk.ChunkStore   // Chunk 缓存存储
    conf        *Config             // VFS 配置
    handles     handle.HandleMap    // 文件句柄表
    internal    *internalNodes      // 内部特殊文件
    ...
}
```

### 1.2 Handle — 文件句柄

```go
// pkg/vfs/handle.go
type Handle struct {
    inode    Ino                    // 关联的 inode
    fh       Fh                     // 句柄 ID
    flags    uint32                 // 打开标志 (O_RDONLY/O_WRONLY/O_RDWR...)
    locks    map[uint64]fileLock    // 文件锁
    reader   *fileReader            // 读取器（维护读位置）
    writer   *fileWriter            // 写入器（维护写缓冲）
    ...
}

// dirHandle — 目录句柄
type dirHandle struct {
    inode    Ino
    fh       Fh
    entries  []*meta.Entry          // 已缓存的目录项
    offset   int                    // 当前读取位置
    ...
}
```

### 1.3 读取器与写入器

```go
// pkg/vfs/reader.go
type fileReader struct {
    inode   Ino
    pos     uint64                 // 当前读取位置
    slices  []meta.Slice           // 缓存的 Slice 列表
    ...
}

// pkg/vfs/writer.go
type fileWriter struct {
    inode   Ino
    buf     []byte                 // 写缓冲区
    dirty   bool                   // 是否有未刷脏数据
    ...
}
```

### 1.4 内部特殊文件

```go
// pkg/vfs/internal.go
const (
    ControlIno Ino = 3    // .control  — 发送控制命令
    StatsIno   Ino = 4    // .stats    — 只读，返回运行时统计
    ConfigIno  Ino = 5    // .config   — 读写卷配置
    LogIno     Ino = 6    // .accesslog — 只读，读取访问日志
    TrashIno   Ino = 2    // .trash    — 回收站
)
```

这些文件不是存在元数据引擎中的，而是 VFS 层**动态生成**的。读取它们会触发自定义的处理逻辑而非查 Meta 接口。

## 二、核心流程

### 2.1 Lookup — 路径解析

```go
// pkg/vfs/vfs.go: Lookup
func (v *VFS) Lookup(ctx meta.Context, parent Ino, name string) (entry *meta.Entry, errno syscall.Errno) {
    // 1. 检查是否为内部特殊文件
    if entry = v.internal.Lookup(parent, name); entry != nil {
        return entry, 0
    }
    
    // 2. 调用元数据引擎
    var inode Ino
    var attr meta.Attr
    if err := v.reader.Lookup(ctx, parent, name, &inode, &attr, true); err != nil {
        return nil, errnoFromError(err)
    }
    
    // 3. 构造返回
    return &meta.Entry{Inode: inode, Attr: &attr}, 0
}
```

### 2.2 Read — 读文件数据（核心流程）

```go
// pkg/vfs/vfs.go: Read
func (v *VFS) Read(ctx meta.Context, ino Ino, buf []byte, off uint64, fh Fh) (int, error) {
    // 1. 获取文件句柄
    h := v.handles.Find(ino, fh)
    if h == nil {
        return 0, syscall.EBADF
    }
    
    // 2. 检查文件大小 — 不能读取超过文件末尾
    var attr Attr
    v.reader.GetAttr(ctx, ino, &attr)
    if off >= attr.Length {
        return 0, io.EOF  // 读位置超出文件大小
    }
    readLen := min(uint64(len(buf)), attr.Length - off)
    
    // 3. 计算 off 属于哪个 Chunk
    chunkSize := uint64(v.conf.Chunk.ChunkSize)  // 64MB
    chunkIndex := uint32(off / chunkSize)
    chunkOff := uint32(off % chunkSize)
    
    // 4. 查询该 Chunk 的 Slice 列表
    var slices []meta.Slice
    v.reader.Read(ctx, ino, chunkIndex, &slices, true)
    
    // 5. 遍历 Slice，找到覆盖 [chunkOff, chunkOff+readLen] 的 Slice
    bytesRead := 0
    for _, s := range slices {
        if chunkOff >= s.Off + s.Len {
            continue  // 跳过已经读过的区域
        }
        // 计算在这个 Slice 内需要读的范围
        sliceStart := max(chunkOff, s.Off) - s.Off
        sliceEnd := min(chunkOff + uint32(readLen), s.Off + s.Len) - s.Off
        
        // 6. 通过 ChunkStore 读取数据
        n, err := v.store.Read(ctx, s.Chunkid, buf[bytesRead:], int(sliceStart), int(sliceEnd-sliceStart))
        
        bytesRead += n
        if bytesRead >= readLen {
            break
        }
    }
    
    // 7. 更新 atime
    v.reader.UpdateAtime(ctx, ino)
    
    return bytesRead, nil
}
```

### 2.3 Write — 写文件数据

```go
// pkg/vfs/vfs.go: Write
func (v *VFS) Write(ctx meta.Context, ino Ino, buf []byte, off uint64, fh Fh) (int, error) {
    h := v.handles.Find(ino, fh)
    chunkSize := uint64(v.conf.Chunk.ChunkSize)
    
    bytesWritten := 0
    for bytesWritten < len(buf) {
        // 1. 分配/获取 chunk（chunkStore 可能已经打开了这个 chunk）
        chunkIndex := uint32((off + uint64(bytesWritten)) / chunkSize)
        
        // 2. 将数据写入 chunk（到本地缓存，不会立即上传到对象存储）
        n, err := v.store.Write(ctx, ..., buf[bytesWritten:], ...)
        
        // 3. 记录 Slice 索引到元数据引擎
        v.writer.Write(ctx, ino, chunkIndex, uint32(off%chunkSize), meta.Slice{
            Chunkid: chunkID,
            Size:    uint32(n),
            Off:     uint32((off + uint64(bytesWritten)) % chunkSize),
            Len:     uint32(n),
        }, true)
        
        bytesWritten += n
    }
    
    // 4. 更新文件大小
    newSize := max(attr.Length, off + uint64(len(buf)))
    v.writer.SetAttr(ctx, ino, SetAttrSize, 0, &Attr{Length: newSize})
    
    return bytesWritten, nil
}
```

> **关键理解**：`v.store.Write()` 写到本地磁盘缓存（不超过阈值时不触发上传），`v.writer.Write()` 将 Slice 索引写入元数据引擎。数据到对象存储的上传是**异步后台**完成的。

### 2.4 内部文件的读处理

```go
// pkg/vfs/internal.go
// .stats 文件：读取时返回 JSON 格式的运行时统计
func (v *VFS) readStats(ctx meta.Context) ([]byte, error) {
    stats := map[string]interface{}{
        "fuse_ops":      v.stats.fuseOps,
        "read_bytes":    v.stats.readBytes,
        "write_bytes":   v.stats.writeBytes,
        "cache_hits":    v.stats.cacheHits,
        "cache_misses":  v.stats.cacheMisses,
        ...
    }
    return json.Marshal(stats)
}

// .control 文件：写入特定命令触发操作
// echo "compact" > .control    → 触发 slice 压缩
// echo "gc.start" > .control   → 触发垃圾回收
// echo "backup.start" > .control → 触发元数据备份
```

## 三、关键设计特性

### 3.1 属性缓存

```go
// VFS 在内存中缓存 Attr，减少元数据引擎访问
func (v *VFS) GetAttr(ctx Context, ino Ino, attr *Attr) error {
    if cached, ok := v.attrCache.Get(ino); ok {
        if time.Since(cached.loadTime) < v.conf.AttrTimeout {
            *attr = cached.attr
            return nil
        }
    }
    // 缓存过期，重新查询
    v.reader.GetAttr(ctx, ino, attr)
    v.attrCache.Set(ino, *attr)
    return nil
}
```

### 3.2 目录项缓存

```go
// 类似 Attr 缓存，目录项（inode + name → entry）也有超时缓存
func (v *VFS) readDirEntries(ctx Context, ino Ino) ([]*Entry, error) {
    if cached, ok := v.dirCache.Get(ino); ok {
        return cached, nil
    }
    entries, _ := v.reader.OpenDir(...)
    v.dirCache.Set(ino, entries)
    return entries, nil
}
```

### 3.3 Handle 生命周期

```
用户 open() → VFS.Open() → 创建 Handle → 存入 handles map
用户 read()/write() → VFS.Read()/Write() → 通过 fh 查找 Handle
用户 close() → VFS.Close() → 从 handles map 移除 Handle
```

Handle 到关闭时才释放，确保同一个 fd 的多次 read/write 使用同一个读位置。

### 3.4 并发控制

```go
// VFS 使用读写锁保护关键数据结构
type VFS struct {
    sync.RWMutex  // 保护 handles、缓存等
    ...
}
```

每个文件操作（Read/Write/Lookup）先获取读锁，修改操作（Create/Unlink）获取写锁。

## 四、关键代码位置

| 文件 | 内容 | 关键函数 |
|------|------|---------|
| `pkg/vfs/vfs.go` | VFS 主体（1493 行） | `NewVFS`, `Lookup`, `Read`, `Write`, `Create`, `Open`, `GetAttr`, `SetAttr`, `Rename`, `Unlink`, `Mkdir`, `Link`, `Symlink`, `StatFS`, `Fsync`, `Flock`, `Fallocate` |
| `pkg/vfs/handle.go` | 文件句柄管理 | `Handle`, `dirHandle`, `newHandle`, `closeHandle` |
| `pkg/vfs/reader.go` | 文件读取逻辑 | `fileReader.read()`, `readSlices()` |
| `pkg/vfs/writer.go` | 文件写入逻辑 | `fileWriter.write()`, `fileWriter.flush()` |
| `pkg/vfs/internal.go` | 内部特殊文件 | `.control`, `.stats`, `.config`, `.accesslog`, `.trash` |
| `pkg/vfs/accesslog.go` | 访问日志 | 记录每个文件操作 |
| `pkg/vfs/backup.go` | 元数据备份 | 定期备份到对象存储 |
| `pkg/vfs/fill.go` | 缓存预热 | `warmup` 命令的实现 |
| `pkg/vfs/compact.go` | Slice 压缩 | 碎片整理触发逻辑 |

## 五、调试与排查

### 5.1 查看 VFS 运行时状态

```bash
# 访问内部 .stats 文件
cat /mnt/jfs/.stats
# 返回 JSON: {"fuse_ops":..., "read_bytes":..., "write_bytes":..., ...}

# 查看卷配置
cat /mnt/jfs/.config

# 发送控制命令
echo "compact" > /mnt/jfs/.control
```

### 5.2 开启 pprof 观察 VFS 性能

```bash
# VFS 的 Read/Write 是 CPU 热点，可以用 pprof 分析
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
# (pprof) top     — 看最耗时的函数
# (pprof) list Read  — 看 Read 函数的逐行耗时
```

### 5.3 日志排查

```bash
# 开启访问日志
cat /mnt/jfs/.accesslog
# 每行: [timestamp] [pid] [uid:gid] [op] [inode] [path] [size] [duration]

# 开启 debug 日志
juicefs mount --verbose redis://localhost/1 /mnt/jfs
```

## 六、常见问题

**Q: VFS 直接访问对象存储吗？**
A: 不。VFS 只通过 `chunk.ChunkStore` 接口读写数据，ChunkStore 内部管理缓存和对象存储访问。VFS 不关心数据是否在本地缓存中。

**Q: reader 和 writer 可以不同吗？**
A: 可以。例如只读挂载时，writer 可以为 nil，所有修改操作会返回 `EROFS`。

**Q: 属性缓存和目录项缓存何时失效？**
A: 超过 `AttrTimeout`/`DirEntryTimeout` 后失效。修改操作（write/create/rename...）会主动刷新相关缓存。

## 七、改进思路 / 贡献方向

1. **一致性快照** — 添加 snapshot 语义，某个时刻给目录打个快照（只读）
2. **读取预取** — 顺序读取时预加载下一个 Chunk 的 Slice 信息
3. **Attr 缓存命中率监控** — 添加 prometheus metrics
4. **更细粒度的锁** — 将全局读写锁拆分为 per-inode 锁
