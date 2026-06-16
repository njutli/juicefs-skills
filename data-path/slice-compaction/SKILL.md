---
name: juicefs-slice-compaction
description: JuiceFS Slice压缩与碎片整理机制。当用户询问Slice碎片、Compact、文件碎片化、BST树管理Slice时调用此技能。
---

# Slice 压缩与碎片整理

## 〇、为什么需要 Slice 压缩？

JuiceFS 的数据模型是 **文件 → Chunk (64MB) → Slice (变长) → Block (4MB)**。每次 `write()` 系统调用都会生成一个新的 Slice。如果对一个文件反复进行小范围写入（如数据库的随机写），会产生大量碎片化的 Slice。

```
例如，同一个 Chunk 内的多次写入：

第一次写入 offset=0,    len=1MB  → Slice1 {Chunkid=1, Off=0,  Len=1MB}
第二次写入 offset=5MB,   len=1MB  → Slice2 {Chunkid=1, Off=5MB, Len=1MB}
第三次写入 offset=1MB,   len=3MB  → Slice3 {Chunkid=1, Off=1MB, Len=3MB}
第四次写入 offset=10MB,  len=2MB  → Slice4 {Chunkid=1, Off=10MB,Len=2MB}
第五次写入 offset=0,     len=6MB  → Slice5 {Chunkid=1, Off=0,  Len=6MB}
                   ↑ 覆盖了 Slice1, Slice3, Slice2 的部分

结果：Slice1,Slice2,Slice3 成为"孤儿"数据（被 Slice5 覆盖了）
     但它们仍在对象存储中占用空间，在元数据中也占记录
```

**Compact 的目的**：合并重叠和相邻的 Slice，删除被覆盖的孤儿数据，减少元数据记录数和对象存储浪费。

## 一、核心数据结构

### 1.1 Slice 类型回顾

```go
// pkg/meta/interface.go
type Slice struct {
    Chunkid uint64   // Slice 所在 chunk 的唯一 ID
    Size    uint32   // Slice 在对象存储中的大小
    Off     uint32   // Slice 在 Chunk 内的偏移（逻辑偏移）
    Len     uint32   // Slice 内有效数据的长度
}
```

`Slice.Off` 是逻辑位置，`Slice.Size` 是物理大小。`Size >= Len`，差值是压缩/对齐导致的。

### 1.2 SliceTree — Chunk 内的 Slice 管理器

```go
// pkg/meta/slice.go
type SliceTree struct {
    slices []Slice
}

// 核心操作：
func (st *SliceTree) Add(slice Slice)        // 添加新 Slice，保持按 Off 排序
func (st *SliceTree) Find(offset uint32) int  // 二分查找 offset 对应的 Slice
func (st *SliceTree) Compact() []Slice        // 合并重叠和相邻的 Slice
func (st *SliceTree) Build()                  // 从已有 Slice 列表构建 BST
```

### 1.3 Compact 配置

```go
// pkg/meta/config.go
type ChunkConf struct {
    ChunkSize      uint64  // 64MB (编译时常量)
    MaxRetries     int     // 上传重试次数
    MaxUploads     int     // 并发上传数
    MaxDeletes     int     // 并发删除数
    ...
}
```

Compact 操作在以下时机触发：
- 手动：`echo "compact" > /mnt/jfs/.control` 或 `juicefs compact META-URL`
- 自动：Chunk 内的 Slice 数超过阈值（内部触发）
- 自动：gc 命令的一部分

## 二、核心流程

### 2.1 Compact 算法

```go
// pkg/vfs/compact.go (简化)
func (v *VFS) compactChunk(ino Ino, indx uint32, slices []Slice) error {
    if len(slices) <= 1 {
        return nil  // 单个 Slice 无需压缩
    }
    
    // 1. 排序 Slice 列表（按 Off 升序）
    sortSlices(slices)
    
    // 2. 合并重叠和相邻的 Slice
    merged := mergeSlices(slices)
    // 例如: [{Off:0,Len:10},{Off:8,Len:10}] → [{Off:0,Len:18}]
    // 例如: [{Off:0,Len:10},{Off:10,Len:10}] → [{Off:0,Len:20}] (相邻也可合并)
    
    // 3. 如果合并后有减少，执行重组
    if len(merged) < len(slices) {
        // 4. 从对象存储读取合并范围的所有数据
        rawData := v.store.Read(ctx, chunkKeys(merged)...)
        
        // 5. 将合并后的数据写入新的 chunk
        newChunkID := v.allocateChunkID()
        v.store.Write(ctx, newChunkID, rawData, 0)
        v.store.Flush(newChunkID)  // 确保上传到对象存储
        
        // 6. 更新元数据引擎：删除旧 Slice，添加新 Slice
        v.writer.Transaction(func() error {
            // 删除所有旧 Slice
            v.writer.DeleteSlice(ctx, ino, indx, oldSlices)
            // 写入新 Slice
            v.writer.Write(ctx, ino, indx, 0, newSlices[0])
            return nil
        })
        
        // 7. 垃圾回收旧 chunk（标记为可删除）
        v.store.Remove(oldChunkIDs...)
    }
    
    return nil
}
```

### 2.2 Slice 合并规则

```go
func canMerge(a, b Slice) bool {
    // 条件1：连续（a.Off + a.Len == b.Off）→ 相邻可合并
    if a.Off + a.Len == b.Off {
        return true
    }
    // 条件2：重叠（a.Off + a.Len >= b.Off）→ 取最大覆盖
    if a.Off + a.Len >= b.Off {
        return true
    }
    return false
}

func merge(a, b Slice) Slice {
    newOff := min(a.Off, b.Off)
    newEnd := max(a.Off + a.Len, b.Off + b.Len)
    return Slice{
        Chunkid: a.Chunkid,  // 保留第一个的 Chunkid
        Off:     newOff,
        Len:     newEnd - newOff,
        Size:    newEnd - newOff,  // 直接合并，实际 Size 可能调整
    }
}
```

### 2.3 全卷 Compact 流程

```go
// cmd/compact.go: compact()
func compact(c *cli.Context) error {
    // 1. 遍历所有文件 inode
    for each file inode {
        // 2. 遍历文件的每个 Chunk
        for chunkIndex := 0; ; chunkIndex++ {
            var slices []Slice
            m.Read(ctx, ino, chunkIndex, &slices, false)
            
            if len(slices) == 0 {
                break  // 没有更多 Chunk
            }
            
            // 3. 检查是否需要压缩
            if len(slices) > compactThreshold {
                v.compactChunk(ino, chunkIndex, slices)
            }
        }
    }
}
```

## 三、关键设计特性

### 3.1 无锁 Compact

Compact 操作**不阻塞**正常文件读写：
- 读取过程中：旧的 Slice 仍然可用（先查元数据获取 Slice 列表，再读数据）
- 写入过程中：新写入的 Slice 不会被 Compact 干扰（互斥的 offset 区域）
- Compact 在更新完元数据后才删除旧 chunk

### 3.2 孤儿 Slice 回收

被覆盖的旧 Slice **不在 Compact 时立即删除**，而是：
1. `gc` 命令扫描所有文件的 Slice 引用
2. 找到没有引用的 chunk → 从对象存储删除
3. 清理元数据中的垃圾记录

### 3.3 trash 与 Compact 的关系

```go
// 被删除的文件先移到 trash（.trash 目录）
// trash 中的文件在 TrashInterval 时间后由 gc 清理
// Compact 不处理 trash 中的文件，由 gc 统一处理
```

### 3.4 碎片化监控

```go
// 碎片化率 = (实际 Slice 数) / (理想 Slice 数)
// 理想 Slice 数 = ceil(文件大小 / 64MB)  ← 每个 Chunk 一个 Slice
fragmentation := float64(actualSlices) / float64(idealSlices)
// 如果 fragmentation > 2.0，说明碎片化严重
```

## 四、关键代码位置

| 文件 | 内容 | 关键区域 |
|------|------|---------|
| `pkg/meta/slice.go` | Slice 树操作 | `SliceTree`, `Add`, `Find`, `Compact`, `Build` |
| `pkg/vfs/compact.go` | VFS 层的 Compact 触发 | `compactChunk`, `mergeSlices` |
| `cmd/compact.go` | CLI compact 命令 | `compact()` |
| `cmd/gc.go` | 垃圾回收（删除孤儿 chunk） | `gc()` |

## 五、调试与排查

### 5.1 查看文件的 Slice 碎片情况

```bash
# 使用 info 命令查看文件详情
juicefs info META-URL /path/to/file
# 输出：
#   inode: 100
#   files: 1
#   dirs:  0
#   length: 100.00 MiB
#   slices: 500     ← 如果很多说明碎片化严重
#   chunks: 2
```

### 5.2 手动触发 Compact

```bash
# 触发全卷 Compact（生产环境建议先测试）
juicefs compact META-URL

# 或通过 internal 文件触发
echo "compact" > /mnt/jfs/.control
```

### 5.3 分析碎片化程度

```bash
# 用 summary 命令看目录汇总
juicefs summary META-URL /mnt/jfs/ --depth 1
# slices/files 比值高说明碎片化严重
```

## 六、常见问题

**Q: Compact 会影响正在写入的文件吗？**
A: 不会。Compact 只处理已写入完成的 Slice。正在进行中的写入有自己的 Slice，写入完成前不会被合并。

**Q: Compact 需要多少额外空间？**
A: 需要完整读取旧 chunk 数据，写入新 chunk。在最坏情况下，需要与原始数据等量的临时空间（在本地缓存中）。

**Q: 多久做一次 Compact？**
A: 取决于写入模式。随机写入多的场景（如数据库）建议定期执行，顺序写入多的场景（日志归档）几乎不需要。

**Q: Compact 可以与 mount 同时运行吗？**
A: 可以。Compact 是独立命令，可以在有其他客户端 mount 时运行。但建议在低负载时执行。

## 七、改进思路 / 贡献方向

1. **增量 Compact** — 只处理增量的碎片化 Slice，避免每次全量扫描
2. **Compact 策略配置** — 允许配置合并阈值（当前固定为 2 个以上 Slice 即可合并）
3. **碎片化率上报** — 添加 Prometheus 指标，展示卷级别的碎片化率
4. **在线 Compact** — 不需要单独运行命令，mount 进程自动在后台 Compact
