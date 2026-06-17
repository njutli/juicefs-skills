---
name: juicefs-end-to-end-write
description: JuiceFS端到端写入追踪。当用户询问一次write()系统调用如何从用户态到达对象存储、完整调用链、各层函数签名与数据变换时调用此技能。
---

# 端到端写入追踪

> **核心问题**：用户执行 `write(fd, "Hello", 5)` 之后，数据是怎么落到 S3 上的？

## 〇、为什么需要端到端追踪？

理解分布式文件系统最有效的方式就是**跟踪一条写入的完整路径**。这篇 Skill 以一次 4KB 写入为例，逐层追踪从 `write()` 系统调用到数据最终出现在 S3 对象存储上的每一步，包括：

- 每层调用的**函数签名**和**文件:行号**
- **数据发生了什么变换**（大小、格式、编码）
- **元数据发生了什么变化**（哪些 key/行被修改）

## 一、写入全貌（12 步速览）

```
write(fd, buf, 4096)                          [用户进程]
  │
  ├─ 1. sys_write()                           [Linux 内核 VFS]
  │     └─ 判断 fd 是 FUSE 文件系统
  │
  ├─ 2. FUSE_WRITE 请求 → /dev/fuse            [FUSE 内核模块]
  │     └─ 封装: {NodeId, Fh, Offset, Size, Data}
  │
  ├─ 3. go-fuse/v2 从 /dev/fuse 读取请求       [用户态 FUSE 库]
  │     └─ fuse.Server.loop() → dispatch()
  │
  ├─ 4. fs.Write(cancel, input, data)          [pkg/fuse/fuse.go]
  │     └─ 构造 FUSE Context (Pid, Uid, Gid)
  │     └─ 调用 vfs.Write()
  │
  ├─ 5. v.Write(ctx, ino, buf, off, fh)        [pkg/vfs/vfs.go]
  │     ├─ 查找 Handle → 检查权限 → 计算 chunk 边界
  │     ├─ 对每个 Chunk:
  │     │
  │     ├─ 6. store.Write(chunkid, data, off)  [pkg/chunk/cached_store.go]  ← 数据路径
  │     │     ├─ 写入 memCache (64KB Page)
  │     │     ├─ 标记 Page.Dirty = true
  │     │     ├─ 达到 128KB writeback 阈值 →
  │     │     │   写入 diskCache (rawstaging/)
  │     │     │   异步上传:
  │     │     │     ├─ 7. 压缩 (lz4/zstd)     [pkg/compress/]
  │     │     │     ├─ 8. 加密 (AES-CTR)       [pkg/object/encrypt.go]（如果配置了）
  │     │     │     └─ 9. object.Put(key, data) [pkg/object/s3.go]
  │     │     │           └─ S3 PutObject API  → 对象存储
  │     │     │     └─ diskCache: rawstaging/ → rawfull/
  │     │     └─ 未达阈值: 仅写 memCache, 跳过 7-9
  │     │
  │     └─ 10. writer.Write(ino, idx, off, slice) [pkg/meta/redis.go]  ← 元数据路径
  │            └─ Lua: ZADD c{ino}_{idx} off slice
  │            └─ 更新 jfs_node.length
  │
  ├─ 11. go-fuse 构造 FUSE_WRITE reply → /dev/fuse
  │
  └─ 12. sys_write() 返回写入字节数            [回到用户进程]
```

## 二、逐层详细追踪

### 第 1-2 步：内核到 FUSE 设备（约 10-50μs）

```
用户进程: write(fd, "AAAA...", 4096)
  │
  ▼
Linux VFS: sys_write()
  → 判断 fd 对应的 inode 是 FUSE 类型
  → fuse_file_write_iter()
    → 构造 FUSE_WRITE_IN 结构:
      {
        fh:        内核文件句柄
        offset:    文件写入偏移
        size:      4096
        write_flags: 0
      }
    → fuse_dev_do_write() → 写入 /dev/fuse
    → 阻塞等待用户态进程回复
```

**数据变换**：无。用户态 buf 原样进入 FUSE 请求。

### 第 3-4 步：go-fuse 接收并分发（`pkg/fuse/fuse.go`）

```go
// ====== go-fuse/v2 内部（非 JuiceFS 代码）======
// fuse.Server.loop() → 从 /dev/fuse 读请求
//   → parseHeader() + parseWrite()
//   → dispatch() → opWrite

// ====== pkg/fuse/fuse.go (594 行) ======
func (fs *fileSystem) Write(cancel <-chan struct{}, input *WriteIn, data []byte) (uint32, Status) {
    // 1. 构造 JuiceFS FUSE Context
    ctx := newContext(cancel, &input.InHeader)
    // ctx.Pid = input.Pid  (~12345)
    // ctx.Uid = input.Uid  (~1000)
    // ctx.Gid = input.Gid  (~1000)
    
    // 2. 映射 FUSE Fh → VFS 内部 Fh
    fh := Fh(input.Fh)
    
    // 3. 调用 VFS 层
    n, err := fs.vfs.Write(ctx, Ino(input.NodeId), data, input.Offset, fh)
    
    // 4. 返回写入字节数
    return uint32(n), ToStatus(err)
}
```

**数据变换**：
- `InHeader.NodeId` → `Ino` (uint64)
- `InHeader.Pid` → `ctx.Pid`
- `input.Fh` → `Fh` (uint64)
- `data` 原样传递（`[]byte` 零拷贝）

### 第 5 步：VFS 层核心编排（`pkg/vfs/vfs.go`）

```go
// pkg/vfs/vfs.go: Write (约 1493 行文件中的核心函数)
func (v *VFS) Write(ctx meta.Context, ino Ino, buf []byte, off uint64, fh Fh) (int, error) {
    
    // --- 5.1 查找文件句柄 ---
    h := v.handles.Find(ino, fh)
    if h == nil {
        return 0, syscall.EBADF  // 无效的文件描述符
    }
    
    // --- 5.2 获取当前文件属性 ---
    var attr Attr
    if err := v.reader.GetAttr(ctx, ino, &attr); err != nil {
        return 0, err
    }
    // attr.Length = 当前文件大小 (例如 0，首次写入)
    
    // --- 5.3 确定 Chunk 边界 ---
    chunkSize := uint64(v.conf.Chunk.ChunkSize)  // 固定 64MB
    // 对于 4KB 写入，只涉及 1 个 chunk (chunkIndex=0)
    
    bytesWritten := 0
    for bytesWritten < len(buf) {
        chunkIndex := uint32((off + uint64(bytesWritten)) / chunkSize)
        chunkOff := uint32((off + uint64(bytesWritten)) % chunkSize)
        
        writeLen := min(len(buf)-bytesWritten, int(chunkSize-uint64(chunkOff)))
        // writeLen = 4096 (写入不跨 chunk 边界)
        
        // --- 5.4 【数据路径】写入 Chunk ---
        n, err := v.store.Write(ctx, h.inode, chunkIndex, 
            buf[bytesWritten:bytesWritten+writeLen], chunkOff)
        //      ↑ 进入第 6 步，见下节
        
        // --- 5.5 【元数据路径】记录 Slice 索引 ---
        slice := Slice{
            Chunkid: chunkID,     // 分配的 chunk 唯一 ID (例如 42)
            Size:    uint32(n),   // 写入大小
            Off:     chunkOff,   // chunk 内偏移
            Len:     uint32(n),  // 有效数据长度
        }
        err = v.writer.Write(ctx, ino, chunkIndex, chunkOff, slice, true)
        //      ↑ 进入第 10 步，见元数据节
        
        bytesWritten += n
    }
    
    // --- 5.6 更新文件大小 ---
    newSize := attr.Length
    if off+uint64(bytesWritten) > newSize {
        newSize = off + uint64(bytesWritten)
    }
    v.writer.SetAttr(ctx, ino, SetAttrSize, 0, &Attr{Length: newSize})
    
    return bytesWritten, nil
}
```

**关键设计**：VFS 不关心数据是否上传到 S3 —— 它只做两件事：
1. 调 `store.Write()` 写入本地缓存（同步，快）
2. 调 `writer.Write()` 记录 Slice 索引（同步）
3. 数据到 S3 的传输由 Chunk 层异步完成

### 第 6-9 步：Chunk 缓存层（`pkg/chunk/cached_store.go`）

```go
// pkg/chunk/cached_store.go (1247 行)

// 6.1 cachedStore.Write —— 写入入口
func (c *cachedStore) Write(ctx context.Context, chunkid uint64, data []byte, offset uint32) (int, error) {
    // 计算该 offset 属于哪个 Page
    pageSize := uint32(PageSize)         // 64KB
    pageIdx := offset / pageSize         // offset=0 → pageIdx=0
    pageOff := offset % pageSize         // offset=0 → pageOff=0
    
    // 分配或获取 Page
    key := fmt.Sprintf("%d_%d", chunkid, pageIdx)
    
    c.Lock()
    // --- 6.2 查内存缓存 ---
    page := c.memCache.lookup(key)
    if page == nil {
        page = &Page{
            Chunkid: chunkid,
            Offset:  int(pageIdx * pageSize),
            Data:    make([]byte, PageSize),   // 分配 64KB
        }
        c.memCache.insert(key, page)
    }
    c.Unlock()
    
    // --- 6.3 复制数据到 Page ---
    copy(page.Data[pageOff:], data)      // 4KB 数据写入 Page[0:4096]
    page.Dirty = true                     // 标记脏页
    
    // --- 6.4 检查是否需要 writeback ---
    dirtyPages := c.memCache.countDirty(key)
    if dirtyPages*int64(PageSize) >= c.conf.MaxWriteback {
        // 默认 128KB (2 个 Page)，首次 4KB 不触发
        // 不触发时: 数据只存在于 memCache 中，等待后续写入触发或 fsync
    }
    
    return len(data), nil
}

// 6.5 【异步】当 writeback 触发时:
// (可能在本次写入或后续写入或 fsync 时触发)
func (c *cachedStore) flushPage(chunkid uint64, page *Page) {
    // Step A: 写入 diskCache (rawstaging/)
    c.diskCache.storeRaw(chunkid, page.Offset, page.Data)
    // 本地磁盘路径: $CACHE_DIR/rawstaging/{chunkid}_{pageIndex}
    
    // Step B: 异步上传到对象存储
    go c.uploadBlock(chunkid, page)
}

// 6.6 异步上传
func (c *cachedStore) uploadBlock(chunkid uint64, page *Page) {
    // Step A: 带宽限制
    c.uploadLimiter.WaitN(ctx, len(page.Data))
    
    // Step B: 压缩（如果配置了 --compress lz4/zstd）
    // pkg/compress/compress.go
    data := page.Data
    if c.conf.Compress != "none" {
        data = compress(c.conf.Compress, data)
        // 4KB → 可能压缩到 3KB
    }
    
    // Step C: 加密（如果配置了 --encrypt-rsa-key）
    // pkg/object/encrypt.go:  AES-256-CTR
    if c.conf.EncryptKey != "" {
        data = encrypt(c.conf.EncryptKey, data)
        // 长度不变（CTR 模式）
    }
    
    // Step D: 构造对象键
    // 格式: chunks/{partition}/{chunkid}_{blockIndex}_{compressed}_{encrypted}
    key := fmt.Sprintf("chunks/%s/%d_%d_%d_%d",
        partition, chunkid, blockIndex,
        boolToInt(compressed), boolToInt(encrypted))
    // 例如: chunks/0/42_0_0_0  (chunkid=42, block=0, 未压缩, 未加密)
    
    // Step E: 上传到对象存储
    c.object.Put(key, bytes.NewReader(data))
    //   → pkg/object/s3.go: s3Client.Put()
    //     → S3 API: PUT /{bucket}/chunks/0/42_0_0_0    [HTTP]
    //       → 对象存储 (MinIO/AWS S3/阿里云 OSS/...)
    
    // Step F: 标记上传完成
    // diskCache: 从 rawstaging/ 移动到 rawfull/ 或直接删除
    c.diskCache.markUploaded(chunkid, page.Offset)
    // mv rawstaging/42_0 → rawfull/42_0
    
    // Step G: Page 标记为干净
    page.Dirty = false
}

// 6.7 【同步写入路径】如果用户调用了 fsync():
func (c *cachedStore) Flush(chunkid uint64) error {
    // 遍历该 chunk 的所有脏页
    for _, page := range c.memCache.dirtyPages(chunkid) {
        c.flushPage(chunkid, page)  // 同步等待上传完成
    }
    return nil
}
```

**数据变换总览**：

```
用户数据: 4096 bytes (明文)
  ├─ 写入 memCache Page[0:4096] (64KB Page 中只填了前 4KB)
  ├─ writeback 触发时:
  │   ├─ [可选] 压缩: 4096 → ~3500 bytes (lz4)
  │   ├─ [可选] 加密: 3500 → 3500 bytes (AES-CTR)
  │   └─ 上传到 S3 key: chunks/0/42_0_0_0
  └─ Page.Dirty = false
```

### 第 10 步：元数据更新（`pkg/meta/redis.go`）

```go
// pkg/meta/redis.go (6263 行)

func (m *redisMeta) Write(ctx Context, ino Ino, indx uint32, off uint32, slice Slice, syscall bool) error {
    // 编码 Slice → string
    // Slice{Chunkid: 42, Size: 4096, Off: 0, Len: 4096}
    // → encoded: binary packed {42, 4096, 0, 4096}
    encoded := encodeSlice(slice)
    
    // Lua 脚本：ZADD c{ino}_{chunkIndex} score member
    // KEYS[1] = "myprefix c100_0"    ← ino=100, chunkIndex=0
    // ARGV[1] = slice.Off = 0        ← ZSET score
    // ARGV[2] = encoded              ← ZSET member
    
    LuaScript := `
        redis.call('ZADD', KEYS[1], ARGV[1], ARGV[2])
        -- 同时更新 inode 的 mtime 和 length
        redis.call('HSET', KEYS[2], 'mtime', ..., 'length', ...)
    `
    
    m.rd.Eval(ctx, LuaScript, 
        []string{m.sliceKey(ino, indx), m.inodeKey(ino)},
        off, encoded, now)
    
    return nil
}

// Redis 中的实际效果:
// ZADD myprefix c100_0 0 "\x00\x00\x00*\x00\x00\x10\x00..."
//   在 ino=100 的第 0 个 chunk 中:
//     score=0  →  member=Slice{chunkid=42,size=4096,off=0,len=4096}
//
// HSET myprefix i100 mtime 1704067200 length 4096
//   更新文件属性: mtime + 文件大小
```

**元数据变化**：

```
Redis 中新增/修改:
  ZSET  myprefix c100_0
        0 → Slice(42, 4096, 0, 4096)
  
  HASH  myprefix i100
        length → 4096
        mtime  → 1704067200
```

### 第 11-12 步：返回到用户进程

```
go-fuse/v2 构造 FUSE_WRITE_OUT reply:
  { size: 4096 }  → write to /dev/fuse

Linux FUSE 内核模块读取 reply:
  → fuse_file_write_iter() 返回 4096

用户进程: write() returns 4096
```

## 三、关键时间节点

| 时间点 | 发生了什么 | 数据在哪 | 元数据在哪 |
|--------|----------|---------|-----------|
| T+0 | write() 返回 | memCache (DRAM) | — (还没写) |
| T+0 (同步) | VFS.Write → meta.Write | — | Redis ZSET 已更新 |
| T+0 到 T+5s | 等待 writeback 触发或后续写入累积 | memCache → diskCache staging/ | Redis 已有 |
| T+5s (异步) | writeback 触发，async upload | staging/ → S3 | — |
| T+5s+N | S3 PutObject 完成 | S3 存储 | — |
| T+5s+N | diskCache 标记上传完成 | staging/ → rawfull/ | — |

**结论**：`write()` 返回 ≠ 数据在 S3。数据可能还在 memCache 中。只有 `fsync()` 后才保证数据在对象存储中。

## 四、SQL 引擎下的元数据变化

如果用 SQL 引擎代替 Redis，第 10 步的变化：

```sql
-- VFS.Write → dbMeta.Write():
-- 在事务中执行：

-- 1. 读取当前 chunk 的所有 Slice
SELECT slices FROM jfs_chunk WHERE inode=100 AND indx=0;

-- 2. 插入新 Slice，重新编码
-- len(slices) = 0 → 新增: [Slice{42,4096,0,4096}]
-- len(slices) = 3 → 插入排序: [... Slice2 Slice3]

-- 3. 写回
INSERT INTO jfs_chunk (inode, indx, slices) VALUES (100, 0, <encoded>)
  ON CONFLICT (inode, indx) DO UPDATE SET slices = <encoded>;

-- 4. 更新文件大小
UPDATE jfs_node SET length=4096, mtime=NOW() WHERE inode=100;
```

## 五、多次写入后的 Slice 积累

```
第 1 次: write(fd, data, 4096)  at offset=0
  → Slice1: {chunkid=42, off=0,    len=4096}

第 2 次: write(fd, data, 8192)  at offset=16384  (跳过 4KB 间隙)
  → Slice2: {chunkid=43, off=16384, len=8192}

第 3 次: write(fd, data, 2048)  at offset=4096   (填补间隙)
  → Slice3: {chunkid=44, off=4096,  len=2048}

ZSET c100_0:
  0     → {chunkid=42, off=0,     len=4096}
  4096  → {chunkid=44, off=4096,  len=2048}    ← 填补了间隙！
  16384 → {chunkid=43, off=16384, len=8192}
```

## 六、完整调用图（附文件:行号）

```
write(fd, buf, 4096)                          [glibc]
  sys_write()
    → fuse_file_write_iter()                  [Linux: fs/fuse/file.c]
      → fuse_dev_do_write()                   [Linux: fs/fuse/dev.c]
        → /dev/fuse
          │
          ▼ [用户态]
        go-fuse/v2: server.go loop()          [vendor/.../fuse/server.go]
          → opWrite dispatch
            │
            ▼
        pkg/fuse/fuse.go: (fs *fileSystem) Write()        [~580行]
          → ctx = newContext(cancel, &input.InHeader)     [pkg/fuse/context.go]
          → fs.vfs.Write(ctx, ino, buf, off, fh)
            │
            ▼
        pkg/vfs/vfs.go: (v *VFS) Write()                  [~800行]
          → h = v.handles.Find(ino, fh)                   [pkg/vfs/handle.go]
          → v.reader.GetAttr(ctx, ino, &attr)             [pkg/meta/redis.go]
          │
          ├─ [数据路径] v.store.Write(...)
          │   ▼
          │   pkg/chunk/cached_store.go: (c *cachedStore) Write()   [~200行]
          │     → memCache.lookup(key)                    [pkg/chunk/mem_cache.go]
          │     → page.Dirty = true
          │     → [异步] uploadBlock(chunkid, page)       [pkg/chunk/cached_store.go]
          │         → compress.Compress(data)             [pkg/compress/compress.go]
          │         → encrypt.Encrypt(data)               [pkg/object/encrypt.go]
          │         → object.Put(key, data)               [pkg/object/s3.go]
          │           → s3client.PutObject(...)           [AWS S3 HTTP API]
          │
          └─ [元数据路径] v.writer.Write(ctx, ino, idx, off, slice)
              ▼
            pkg/meta/redis.go: (m *redisMeta) Write()     [~3500行]
              → Lua: ZADD c{ino}_{idx} off encoded_slice
              → Lua: HSET i{ino} length mtime
              
              [OR]
              
            pkg/meta/sql.go: (m *dbMeta) Write()           [~4000行]
              → txn: SELECT / INSERT/UPDATE jfs_chunk
              → txn: UPDATE jfs_node SET length=...
```

## 七、关键设计要点

1. **写入不是原子的**：`write()` 返回后，数据可能在 memCache / diskCache / S3 三者之一。断电可能丢未上传的数据。
2. **元数据先写**：VFS 先写 Slice 索引到元数据引擎，再异步上传数据。即使 S3 上传失败，gc 也可以根据 Slice 索引检测到缺失的 chunk。
3. **Chunk 不变性**：上传到 S3 的 chunk 不再修改。数据修改 = 生成新 chunk + 新 Slice 索引。
4. **写放大**：一个 4KB 写入可能触发 64KB Page 分配 + 4MB Block 的上传（如果该 block 没有其他数据可聚合）。

## 八、调试与验证

```bash
# 1. 写入前后查看 Slice 变化
juicefs info META-URL /path/to/file
# slices: 0 → 1

# 2. 查看 Redis 中的 Slice
redis-cli ZRANGE "prefix c100_0" 0 -1 WITHSCORES

# 3. 查看本地缓存
ls -la /var/jfsCache/rawstaging/  # 可能有待上传的 chunk
ls -la /var/jfsCache/rawfull/     # 已上传的 chunk

# 4. 查看对象存储中的实际对象
aws s3 ls s3://bucket/chunks/0/ --endpoint-url=...

# 5. 强制刷盘后确认
# (在另一个终端) sync /mnt/jfs/
```

## 九、常见问题

**Q: write() 返回 4096 但数据还没到 S3，断电会丢吗？**
A: 会丢。JuiceFS 默认 writeback 模式。写操作返回只保证数据在内核缓冲区或 memCache 中。使用 `sync` 或配置 `--writeback` 关闭可改为 writethrough。

**Q: 如果上传到一半进程崩溃怎么办？**
A: 下次 mount 时会扫描 `rawstaging/` 中的残留文件并重新上传。

**Q: Slice 索引先写，数据后传，中间某一步失败了怎么办？**
A: Slice 引用的 chunk 可能不存在于对象存储中，读取时返回零填充。gc 命令可以清理这类孤立的 Slice。

## 十、改进思路 / 贡献方向

1. **写入聚合优化** — 小写入在 memCache 中聚合更久以减少小对象上传
2. **Writethrough 模式** — 支持 write() 返回前确保数据到达 S3（牺牲性能换安全性）
3. **Checksum 验证** — 上传后读取并验证校验和
4. **更好的写入诊断** — 暴露每个写入阶段（memCache/diskCache/S3）的延迟到 metrics
