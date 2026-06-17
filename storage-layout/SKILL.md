---
name: juicefs-storage-layout
description: JuiceFS在三层存储介质上的完整数据布局。当用户询问文件系统磁盘结构、元数据引擎数据组织、对象存储key命名规范、本地缓存目录结构、反向映射关系时调用此技能。
---

# JuiceFS 存储布局全貌

> **核心问题**：JuiceFS 如何把用户看到的文件树（`/data/file.txt`）映射为三个独立存储介质上的比特序列？

## 〇、为什么需要理解存储布局？

传统本地文件系统（如 ext4、XFS）有一个磁盘布局规范文档，描述 superblock、inode table、block bitmap 等结构。JuiceFS 没有"磁盘"，但有**三个存储介质**，每个都有自己的一套数据组织方式。

理解存储布局意味着：
- 不用 JuiceFS 客户端也能**直接从 Redis/SQL 查询元数据**
- 能通过 `aws s3 ls` 直接验证数据完整性
- 能在极端情况下**手动修复损坏的元数据**
- 能理解为什么某个操作在 Redis vs. SQL 引擎下性能不同

```
用户视角:     /data/file.txt          (一棵树)
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   元数据引擎    对象存储      本地缓存
  (Redis/SQL)   (S3/MinIO)    (SSD/HDD)
```

## 一、元数据引擎布局

### 1.1 Redis 引擎 Key 全景

所有 key 以 `{prefix}` 为 hash tag 前缀（`prefix` = `volumeName_` 或自定义前缀），确保同一卷的 key 落在 Redis Cluster 的同一个 slot。

```
Redis 全局 key 列表:
╔══════════════════════════════════════════════════════════════╗
║ Key                      │类型│ 内容                        ║
╠══════════════════════════╪════╪═════════════════════════════╣
║ {pfx}setting             │ Str│ JSON(Format) — 卷配置       ║
║ {pfx}nextinode           │ Str│ 下一个可用 inode 号          ║
║ {pfx}nextchunk           │ Str│ 下一个可用 chunk ID          ║
║ {pfx}nextsession         │ Str│ 下一个可用 session ID        ║
║ {pfx}sessions            │ Hash│ sid → JSON(SessionInfo)     ║
║ {pfx}i{ino}              │ Hash│ field → Attr 字段值          ║
║ {pfx}d{parentIno}        │ Hash│ name → childIno             ║
║ {pfx}c{ino}_{chunkIndex} │ ZSET│ offset → Slice编码           ║
║ {pfx}s{ino}              │ Str │ 符号链接目标路径             ║
║ {pfx}x{ino}              │ Hash│ xattr_name → xattr_value    ║
║ {pfx}l{ino}              │ Hash│ lock_owner → lock_info      ║
║ {pfx}t{parentIno}        │ Hash│ name → ino (回收站)         ║
║ {pfx}ss{sid}             │ ZSET│ ino → refcount (打开文件)   ║
║ {pfx}q{ino}              │ Hash│ quotaType → quotaInfo       ║
║ {pfx}st{ino}             │ Hash│ 目录统计信息                 ║
╚══════════════════════════════════════════════════════════════╝
```

### 1.2 各 Key 的详细格式

#### 1.2.1 `{pfx}setting` — 卷配置（String）

```json
// redis-cli GET "{pfx}setting"
{
  "Name": "myvol",
  "UUID": "a1b2c3d4-...",
  "Storage": "minio://192.168.1.100:9000/mybucket",
  "Bucket": "mybucket",
  "BlockSize": 4096,
  "Compression": "none",
  "EncryptKey": "",
  "TrashInterval": 86400,
  "Partitions": 0,
  "ChunkSize": 67108864,
  "CreatedAt": "2024-01-01T00:00:00Z"
}
```

#### 1.2.2 `{pfx}i{ino}` — 节点属性（Hash）

```
redis-cli HGETALL "{pfx}i100"

1) "flags"      2) "0"
3) "type"        4) "1"           ← TypeFile
5) "mode"       6) "420"          ← 0644 (八进制)
7) "uid"        8) "1000"
9) "gid"        10) "1000"
11) "atime"      12) "1704067200"
13) "mtime"      14) "1704067200"
15) "ctime"      16) "1704067200"
17) "atimensec" 18) "0"
19) "mtimensec" 20) "0"
21) "ctimensec" 22) "0"
23) "nlink"      24) "1"
25) "length"     26) "1073741824"  ← 1GB
27) "rdev"       28) "0"
29) "parent"     30) "1"          ← 父目录 = 根目录
31) "full"       32) "1"          ← 完整属性
```

#### 1.2.3 `{pfx}d{parentIno}` — 目录项（Hash）

```
redis-cli HGETALL "{pfx}d1"    ← 根目录 (ino=1)

1) "data"        2) "100"       ← /data/ → ino=100
3) "home"        4) "200"       ← /home/ → ino=200
5) "file.txt"   6) "300"        ← /file.txt → ino=300
```

#### 1.2.4 `{pfx}c{ino}_{chunkIndex}` — Slice 索引（ZSET）

```
redis-cli ZRANGE "{pfx}c100_0" 0 -1 WITHSCORES

# 三个 Slice:
1) "\x00\x00\x00*\x00\x00\x10\x00..."   ← 编码的 Slice1
2) "0"                                    ← score = off=0
3) "\x00\x00\x00+\x00\x00 \x00..."       ← 编码的 Slice2  
4) "16384"                                ← score = off=16384
5) "\x00\x00\x00,\x00\x00\x08\x00..."    ← 编码的 Slice3
6) "24576"                                ← score = off=24576
```

**ZSET 设计原理**：score = slice.Off（逻辑偏移），member = 编码的 `{chunkid, size, off, len}`。ZRANGEBYSCORE 可以高效查询覆盖任意 offset 范围的 Slice。

#### 1.2.5 Slice 编码格式

```go
// pkg/meta/redis.go: encodeSlice
// 每个 Slice 编码为 16 字节 + 变长（取决于是否压缩）:
//
// ┌──────────┬──────────┬──────────┬──────────┐
// │  chunkid │   size   │   off    │   len    │
// │  (4B)    │  (4B)    │  (4B)    │  (4B)    │
// └──────────┴──────────┴──────────┴──────────┘
//   uint64 chunkid 的高 32 位存 flags (压缩/加密)
```

#### 1.2.6 `{pfx}sessions` — Session 表（Hash）

```
redis-cli HGETALL "{pfx}sessions"

1) "1001"
2) "{\"Sid\":1001,\"Heartbeat\":\"2024-01-01T12:00:00Z\",\"Hostname\":\"node1\",\"MountPoint\":\"/mnt/jfs\",\"Version\":\"1.2.0\",\"IP\":\"10.0.0.1\"}"
```

### 1.3 SQL 引擎表布局

```
Database: juicefs
╔══════════════╤══════════════════════════════════════════════╗
║ 表名          │ 说明                                       ║
╠══════════════╪══════════════════════════════════════════════╣
║ jfs_node     │ 节点属性 (inode, flags, type, mode, uid,   ║
║              │ gid, atime, mtime, ctime, nlink, length...) ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_edge     │ 目录项 (parent, name, inode)                ║
║              │ PRIMARY KEY (parent, name)                  ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_symlink  │ 符号链接 (inode, target)                    ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_chunk    │ Slice 索引 (inode, indx, slices BLOB)      ║
║              │ PRIMARY KEY (inode, indx)                   ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_xattr    │ 扩展属性 (inode, name, value)               ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_flock    │ 文件锁 (inode, sid, owner, ltype...)        ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_session  │ Session 表 (sid, heartbeat, info JSON)      ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_session2 │ Session 打开文件 (sid, inode, count)        ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_quota    │ 配额信息 (inode, qtype, max_space, ...)     ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_delfile  │ 删除文件/回收站 (inode, length, expire)     ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_setting  │ 卷配置 (name, value) — 单行: Format JSON   ║
╟──────────────┼──────────────────────────────────────────────╢
║ jfs_sustained│ 持续统计 (inode, ...)                       ║
╚══════════════╧══════════════════════════════════════════════╝
```

**与 Redis 的对应关系**：
```
Redis Key                  SQL Table
─────────────────────      ──────────
i{ino}                →    jfs_node
d{parent}             →    jfs_edge (WHERE parent=?)
c{ino}_{indx}         →    jfs_chunk (WHERE inode=? AND indx=?)
s{ino}                →    jfs_symlink
x{ino}                →    jfs_xattr
sessions              →    jfs_session
ss{sid}               →    jfs_session2
t{parent}             →    jfs_delfile
q{ino}                →    jfs_quota
setting               →    jfs_setting
```

### 1.4 特殊 inode 编号（所有引擎通用）

```
Ino 1   = 根目录 (/)
Ino 2   = 回收站 (.trash)
Ino 3   = 控制文件 (.control)
Ino 4   = 统计文件 (.stats)
Ino 5   = 配置文件 (.config)
Ino 6   = 访问日志 (.accesslog)
Ino 7-8 = 预留
Ino 9+  = 用户文件/目录（从 9 开始分配）
```

## 二、对象存储布局

### 2.1 对象键命名规范

JuiceFS 在对象存储中**只存储数据块**。根下只有一个 `chunks/` 前缀目录（如果未开启 sharding）：

```
s3://mybucket/
└── chunks/
    └── 0/                          ← 分区号 (shard=0, 默认不分片)
        ├── 42_0_0_0                ← chunkid=42, block=0, 未压缩, 未加密
        ├── 42_1_0_0                ← chunkid=42, block=1
        ├── 43_0_1_0                ← chunkid=43, block=0, 压缩,  未加密
        ├── 43_0_1_1                ← chunkid=43, block=0, 压缩,  加密
        └── ...
```

### 2.2 对象键编码规则

```
格式: chunks/{partition}/{chunkid}_{blockIndex}_{compressed}_{encrypted}

partition     = chunkid % Partitions       (默认 Partitions=0 → 固定 "0")
chunkid       = 全局自增的 chunk 唯一 ID
blockIndex    = chunk 内的 4MB block 序号 (0, 1, 2, ...)
compressed    = 0(未压缩) / 1(压缩)
encrypted     = 0(未加密) / 1(加密)
```

示例：
```
chunks/0/42_0_0_0    → chunkid=42, 第一个 4MB block, 未压缩, 未加密
chunks/0/42_1_0_0    → chunkid=42, 第二个 4MB block
chunks/0/42_2_1_1    → chunkid=42, 第三个 4MB block, 已压缩, 已加密

chunks/3/100_0_0_0   → 如果 Partitions=4, 100%4=0 在上面的分区
chunks/3/100_1_0_0   → 但这里我写的是3...实际上应该是 100%4=0, chunks/0/
```

> **修正**：partition = `chunkid % Partitions`。如果 Partitions=0 则 partition 固定为 0。

### 2.3 从文件 offset 定位到 S3 key

给定文件 ino=100，offset=8MB，chunkSize=64MB：

```
1. chunkIndex = 8MB / 64MB = 0         ← 在第一个 chunk
2. chunkOff   = 8MB % 64MB = 8MB       ← chunk 内偏移
3. 查询元数据: ZRANGE c100_0 0 8388608  ← 找覆盖此偏移的 Slice
   → Slice: {chunkid=42, size=16MB, off=0, len=16MB}
4. sliceOff = 8MB - 0 = 8MB            ← Slice 内偏移
5. blockIndex = 8MB / 4MB = 2          ← 第 2 个 block
6. blockOff   = 8MB % 4MB = 0          ← block 内偏移
7. S3 Key = chunks/0/42_2_0_0
8. S3 Range: bytes=0-4095              ← 读 4KB
```

```
                    文件 ino=100 (offset=8MB 读取 4KB)

    Chunk 0 (64MB)
    ┌──────────────────────────────────────────────┐
    │          Slice {chunkid=42, off=0, len=16MB} │
    │ ┌────────┬────────┬────────┬────────────────┐ │
    │ │Block 0 │Block 1 │Block 2 │   Block 3      │ │
    │ │ (4MB)  │ (4MB)  │ (4MB)  │   (4MB)        │ │
    │ └────────┴────────┴───▲────┴────────────────┘ │
    └───────────────────────│──────────────────────┘
                            │
              offset=8MB ───┘
              
    S3: GET /chunks/0/42_2_0_0  Range: bytes=0-4095
```

### 2.4 从 S3 key 反向定位到文件和 offset

```
给定 S3 key: chunks/0/42_2_0_0
  chunkid=42, blockIndex=2, uncompressed, unencrypted

1. 在元数据引擎中搜索 chunkid=42 的 Slice 引用
   → 扫描所有 c{ino}_{chunkIndex} ZSET 或 jfs_chunk 表
   → 找到: ino=100, chunkIndex=0, slice.Off=0

2. slice 起点 offset = slice.Off = 0
3. block 起点 offset = 0 + 2*4MB = 8MB
4. 所以这个 block 对应的文件位置是:
   文件 ino=100 的 [8MB, 12MB) 范围

5. 查文件属性:
   HGETALL {pfx}i100 → parent=1 → 父目录 ino=1
   HGET {pfx}d1 "文件名" → ino=100 → 反查文件名
```

> 这种反向映射在 gc/fsck 中频繁使用：从 S3 对象找对应的元数据引用，标记为"已引用"或"孤儿"。

## 三、本地磁盘缓存布局

### 3.1 缓存目录结构

```
{CacheDir}/
├── rawstaging/           # 写缓冲区 — 待上传到对象存储的 blocks
│   ├── 42_0              # chunkid=42, blockIndex=0
│   ├── 42_1
│   └── 43_0
│
├── rawslices/            # 碎片数据 — 小于 4MB 或不完整的写入
│   ├── 42_0_1024         # chunkid=42, offset=0, length=1024
│   └── 42_2048_512       # chunkid=42, offset=2048, length=512
│
├── rawfull/              # 完整 blocks — 已上传或缓存的全 4MB 块
│   ├── 42_0              # chunkid=42, blockIndex=0
│   ├── 42_1
│   └── 43_0
│
└── cached/               # 缓存的 blocks — 可选，与 rawfull 类似
    ├── 42_0              # 如果启用了压缩/加密，这里存处理后的数据
    └── ...
```

### 3.2 各目录的作用与生命周期

```
写入阶段:
  write() → 数据写入 memCache (DRAM)
    │
    ├─ writeback 触发 → 数据写入 rawstaging/{chunkid}_{blockIndex}
    │                      (本地 SSD/HDD)
    │
    ├─ 异步上传 → object.Put(key, data)  → S3
    │
    │  上传成功:
    │  ┌── rawstaging/42_0 → 移动到 rawfull/42_0
    │  │   (标记为已上传的缓存 block)
    │  │
    │  └── 或直接删除 (如果 cache-size 已满)
    │
    └─ 不完整的数据块（小于 4MB 或未对齐）:
        写入 rawslices/{chunkid}_{offset}_{length}
```

### 3.3 缓存恢复机制

```go
// 挂载时扫描已有缓存
func (c *diskCache) scanExisting(dir string) error {
    // 1. 扫描 rawfull/ — 已上传的完整 blocks
    //    → 解析文件名 → 重建缓存索引
    //    → 记录每个 block 的大小、访问时间
    
    // 2. 扫描 rawslices/ — 碎片数据
    //    → 合并到对应 chunk 的 Page 结构中
    
    // 3. 扫描 rawstaging/ — 未上传的 blocks
    //    → 重新安排上传（之前可能因为进程被杀而未上传）
    //    → 上传成功后移动到 rawfull/
}
```

**断电场景**：
```
断电前: rawstaging/ 中有未上传的 blocks
断电后重新 mount:
  1. 扫描 rawstaging/
  2. 发现 42_0 → 重新上传到 S3
  3. 上传成功 → rawstaging/42_0 → rawfull/42_0
```

### 3.4 缓存文件命名汇总

```
文件名格式                   │ 位置         │ 含义
═══════════════════════════╪══════════════╪═════════════════════
{chunkid}_{blockIndex}     │ rawstaging/  │ 待上传的 4MB block
{chunkid}_{offset}_{len}   │ rawslices/   │ 碎片数据
{chunkid}_{blockIndex}     │ rawfull/     │ 已上传的 4MB block 缓存
{chunkid}_{blockIndex}     │ cached/      │ 处理后的 block 缓存
```

## 四、三层存储的对应关系总图

```
用户文件: /data/file.txt (ino=100, 1GB)
════════════════════════════════════════════════════════════════

┌─ 元数据引擎 ────────────────────────────────────────────────┐
│ Redis: HGETALL {pfx}i100 → {type:1, mode:644, length:1GB..}│
│ Redis: HGET {pfx}d1 "data" → 100                            │
│ Redis: ZRANGE {pfx}c100_0 → [Slice1, Slice2, ...]           │
│ Redis: ZRANGE {pfx}c100_1 → [SliceN, ...]                   │
│                                                              │
│ [SQL 等价]                                                   │
│ SELECT * FROM jfs_node WHERE inode=100                       │
│ SELECT * FROM jfs_edge WHERE parent=1 AND name='data'        │
│ SELECT slices FROM jfs_chunk WHERE inode=100 AND indx=0      │
│ SELECT slices FROM jfs_chunk WHERE inode=100 AND indx=1      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ 对象存储 ──────────────────────────────────────────────────┐
│ chunks/0/42_0_0_0    ← chunkid=42, block=0, 4MB            │
│ chunks/0/42_1_0_0    ← chunkid=42, block=1, 4MB            │
│ chunks/0/42_2_0_0    ← chunkid=42, block=2, 4MB            │
│ chunks/0/42_3_0_0    ← chunkid=42, block=3, 4MB            │
│ ...                                                        │
│ chunks/0/50_0_0_0    ← chunkid=50, block=0, 4MB            │
│ ...                                                        │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼             (只包含热数据)
┌─ 本地缓存 ──────────────────────────────────────────────────┐
│ rawfull/42_0          ← chunkid=42, block=0  (最近读过)     │
│ rawfull/42_1          ← chunkid=42, block=1  (最近读过)     │
│ memCache: {42,page0}  ← chunkid=42, page=0 (最近访问)      │
└──────────────────────────────────────────────────────────────┘
```

## 五、数据重建验证

你可以不依赖 JuiceFS 客户端，手动验证数据完整性：

### 5.1 列出目录内容（纯 Redis 命令）

```bash
# 查看根目录下的文件和子目录:
redis-cli HGETALL "{pfx}d1"
# "data" → "100"
# "home" → "200"

# 查看 /data/ 目录下的内容:
redis-cli HGETALL "{pfx}d100"

# 查看每个文件的属性:
redis-cli HGETALL "{pfx}i100"  → 得到文件大小、权限、修改时间
```

### 5.2 重建文件内容（纯 Redis + S3 命令）

```bash
# 1. 从 Redis 获取 Slice 列表
redis-cli ZRANGE "{pfx}c100_0" 0 -1 WITHSCORES

# 2. 对于每个 Slice，解码获取 chunkid + offset + length
# 3. 对于每个 block，从 S3 下载数据
aws s3 cp s3://{bucket}/chunks/0/42_0_0_0 /tmp/block_0
aws s3 cp s3://{bucket}/chunks/0/42_1_0_0 /tmp/block_1
...

# 4. 拼接成完整文件
cat /tmp/block_* > /tmp/reconstructed_file
```

### 5.3 验证孤儿对象（纯 S3 命令 + Redis）

```bash
# 1. 列出 S3 中的所有对象
aws s3 ls s3://{bucket}/chunks/ --recursive > s3_objects.txt

# 2. 从 Redis 收集所有 Slice 引用的 chunkid 集合
redis-cli --scan --pattern "{pfx}c*" | while read key; do
  redis-cli ZRANGE "$key" 0 -1 | decode_chunk_ids >> referenced_chunks.txt
done

# 3. 对比: S3 中有但 Redis 未引用的 → 孤儿对象
comm -23 <(sort s3_objects.txt) <(sort referenced_chunks.txt)
```

## 六、关键代码位置

| 文件 | 内容 | 关键区域 |
|------|------|---------|
| `pkg/meta/redis.go` | Redis key 命名与 Slice 编解码 | `inodeKey()`, `entryKey()`, `sliceKey()`, `encodeSlice()`, `parseSlice()` |
| `pkg/meta/sql.go` | SQL 表定义与 Slice 序列化 | `marshalSlices()`, `unmarshalSlices()`, 建表语句 |
| `pkg/meta/config.go` | Format 配置序列化 | `Format.Load()`, `Format.Dump()` |
| `pkg/chunk/disk_cache.go` | diskCache 文件命名与路径 | `key()`, `scanExisting()` |
| `pkg/chunk/cached_store.go` | uploadBlock key 构造 | `upload()` 中的 fmt.Sprintf("chunks/%s/...") |
| `pkg/object/interface.go` | ObjectStorage 接口 | `Put(key, reader)`, `Get(key, off, limit)` |
| `pkg/vfs/internal.go` | 特殊 inode 定义 | `RootIno=1`, `TrashIno=2`, `ControlIno=3`... |

## 七、调试与排查

```bash
# === 元数据引擎 ===
# 查看卷配置
redis-cli GET "{pfx}setting"

# 查看文件属性
redis-cli HGETALL "{pfx}i100"

# 查看目录内容
redis-cli HGETALL "{pfx}d1"

# 查看 Slice 索引
redis-cli ZRANGE "{pfx}c100_0" 0 -1 WITHSCORES

# 查看所有 sessions
redis-cli HGETALL "{pfx}sessions"

# === 对象存储 ===
# 查看所有 chunks
aws s3 ls s3://{bucket}/chunks/ --recursive | head -50

# 查看某个 chunk
aws s3 ls s3://{bucket}/chunks/0/42_

# 下载某个 block 查看
aws s3 cp s3://{bucket}/chunks/0/42_0_0_0 /tmp/block.bin
xxd /tmp/block.bin | head

# === 本地缓存 ===
# 查看缓存使用情况
du -sh /var/jfsCache/*
ls -la /var/jfsCache/rawstaging/
ls -la /var/jfsCache/rawfull/
```

## 八、常见问题

**Q: JuiceFS 的"超级块"在哪里？**
A: 没有传统意义上的超级块。卷配置（等价于 superblock）存在元数据引擎的 `setting` key 或 `jfs_setting` 表中。

**Q: inode 号是怎么分配的？**
A: 通过 `{pfx}nextinode` key（Redis）或数据库自增序列（SQL）分配。inode 9 开始分配给用户文件（1-8 是内部 inode）。

**Q: 如果我不小心删除了 Redis 中的 key，能恢复吗？**
A: 部分可以。如果只丢失了 Slice 索引，对象存储中的数据还在，可以通过扫描 S3 chunks 重建，但需要编写复杂的恢复工具。元数据备份 (`juicefs dump`) 是推荐的做法。

**Q: 本地缓存中 rawfull 和 cached 有什么区别？**
A: `rawfull` 存储的是原始（未处理）的 block 数据，`cached` 存储的是经过压缩/加密处理后的数据（如果配置了压缩/加密）。大部分场景只需要关注 `rawstaging` 和 `rawfull`。

## 九、改进思路 / 贡献方向

1. **离线恢复工具** — 不依赖 JuiceFS 客户端的元数据重建/修复工具
2. **可视化存储布局** — 一个 Web 工具展示三层存储的实时对应关系
3. **存储使用分析** — 分析哪些 Slice 导致了最多的碎片，哪些 chunk 是冷数据
4. **元数据引擎迁移工具** — Redis ↔ SQL 之间的元数据迁移，包括 Slice 编码格式转换
