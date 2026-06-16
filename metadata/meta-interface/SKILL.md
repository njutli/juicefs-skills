---
name: juicefs-meta-interface
description: JuiceFS Meta接口定义与核心类型。当用户询问Meta接口、Ino/Attr/Slice类型、Config/Format配置、Context上下文、Session管理时调用此技能。
---

# Meta 接口定义与核心类型

## 〇、为什么需要 Meta 接口？

JuiceFS 的文件系统元数据并不是存在本地磁盘，而是存储在**外部数据库**（Redis/SQL/TiKV 等）中。为了让所有引擎都有一致的行为，JuiceFS 定义了一个统一的 `Meta` 接口——约 100+ 个方法，涵盖所有 POSIX 文件系统操作。

这是 JuiceFS 最重要的抽象层。理解了 Meta 接口，就理解了 JuiceFS 能做什么、不能做什么。

## 一、核心类型定义

### 1.1 Ino — inode 号

```go
// pkg/meta/interface.go
type Ino uint64
```

JuiceFS 中一切文件、目录、符号链接都用 `Ino` 唯一标识。根目录的 inode 号固定为 `1`。

```go
const (
    RootIno      Ino = 1     // 根目录
    TrashIno     Ino = 2     // 回收站
    ControlIno   Ino = 3     // .control 虚文件
    StatsIno     Ino = 4     // .stats 虚文件
    ConfigIno    Ino = 5     // .config 虚文件
    LogIno       Ino = 6     // .accesslog 虚文件
    MinSpecialIno = 9        // 特殊 inode 保留区
)
```

### 1.2 Attr — 文件属性

```go
// pkg/meta/interface.go
type Attr struct {
    Flags     uint8  // 标志位 (IMMUTABLE, APPEND, ...)
    Typ       uint8  // 类型: TypeFile=1, TypeDirectory=2, TypeSymlink=3, ...
    Mode      uint16 // 权限位 (0644, 0755, ...)
    Uid       uint32
    Gid       uint32
    Atime     int64  // 访问时间（秒）
    Mtime     int64  // 修改时间（秒）
    Ctime     int64  // 状态变更时间（秒）
    Atimensec uint32 // 访问时间纳秒部分
    Mtimensec uint32
    Ctimensec uint32
    Nlink     uint32 // 硬链接数
    Length    uint64 // 文件大小（字节）
    Rdev      uint32 // 设备号（特殊文件）
    Parent    Ino    // 父目录 inode (用于 getdents)
    Full      bool   // 是否为完整属性（或仅部分字段有效）
}
```

**关键理解**：`Attr.Length` 是文件逻辑大小，`Full` 标志表示是否所有字段有效。`Mtime`/`Atime` 使用秒+纳秒两部分存储。

### 1.3 Slice — 文件数据块索引

```go
// pkg/meta/interface.go
type Slice struct {
    Chunkid uint64  // Chunk 的唯一 ID
    Size    uint32  // Slice 大小
    Off     uint32  // Slice 在 Chunk 内的偏移
    Len     uint32  // Slice 内有效数据长度
}
```

```
文件 "hello.txt" (size=10MB)
  │
  └─ Chunk 0 (64MB 空间，只有 10MB 数据)
       ├── Slice 1: {Chunkid=1, Size=2MB, Off=0,     Len=2MB}     # 第一次写入 0-2MB
       ├── Slice 2: {Chunkid=2, Size=5MB, Off=2MB,   Len=5MB}     # 第二次写入 2-7MB
       └── Slice 3: {Chunkid=3, Size=4MB, Off=7MB,   Len=3MB}     # 第三次写入 7-10MB

每个 Slice 的 {Chunkid, Size, Off, Len} 记录了一个连续数据块在对象存储中的位置。
```

### 1.4 Config — 挂载配置

```go
// pkg/meta/config.go
type Config struct {
    Version           string
    Format            Format      // 从元数据引擎加载的卷格式
    Chunk             *ChunkConf
    Name              string
    UUID              string
    ClientVersion     string
    Heartbeat         time.Duration  // session 心跳间隔
    AttrTimeout       time.Duration  // 属性缓存超时
    DirEntryTimeout   time.Duration  // 目录项缓存超时
    ReadOnly          bool
    NoBGJob           bool
    OpenCache         time.Duration  // 打开文件缓存超时
    ...
}
```

### 1.5 Context — 请求上下文

```go
// pkg/meta/context.go
type Context interface {
    context.Context                    // 嵌入标准库 context
    Pid() uint32                       // 发起请求的进程 PID
    Uid() uint32                       // 用户 ID
    Gid() uint32                       // 组 ID
    SetValue(key, value interface{})
    WithValue(key, value interface{}) Context
    Gids() []uint32                    // 附加组
    LookupGroup()                      // 检查组成员资格
    ...
}
```

## 二、Meta 接口全貌

Meta 接口定义在 `pkg/meta/interface.go`，约 100+ 方法。按功能分类如下：

### 2.1 基础文件操作

```go
type Meta interface {
    // 查找
    Lookup(ctx Context, parent Ino, name string, inode *Ino, attr *Attr, syscall bool) error
    Resolve(ctx Context, parent Ino, path string, inode *Ino, attr *Attr) error
    GetAttr(ctx Context, ino Ino, attr *Attr) error
    SetAttr(ctx Context, ino Ino, set uint16, sugidclearmode uint8, attr *Attr) error

    // 创建 / 删除
    Create(ctx Context, parent Ino, name string, mode uint16, cumask uint16, flags uint32, inode *Ino, attr *Attr) error
    Mknod(ctx Context, parent Ino, name string, typ uint8, mode uint16, cumask uint16, rdev uint32, inode *Ino, attr *Attr) error
    Mkdir(ctx Context, parent Ino, name string, mode uint16, cumask uint16, inode *Ino, attr *Attr) error
    Unlink(ctx Context, parent Ino, name string) error
    Rmdir(ctx Context, parent Ino, name string) error

    // 重命名 / 链接
    Rename(ctx Context, parentSrc Ino, nameSrc string, parentDst Ino, nameDst string, flags uint32, inode *Ino, attr *Attr) error
    Link(ctx Context, inodeSrc Ino, parent Ino, name string, attr *Attr) error
    Symlink(ctx Context, parent Ino, name string, path string, inode *Ino, attr *Attr) error
    ReadLink(ctx Context, ino Ino, path *[]byte) error
}
```

### 2.2 文件读写（数据索引操作）

```go
    // 打开/关闭
    Open(ctx Context, ino Ino, flags uint32, attr *Attr) error
    Close(ctx Context, ino Ino) error

    // 读 Slice 索引（不是读数据本身！）
    Read(ctx Context, ino Ino, indx uint32, slices *[]Slice, syscall bool) error
    // 写 Slice 索引
    Write(ctx Context, ino Ino, indx uint32, off uint32, slice Slice, syscall bool) error
    // 截断
    Truncate(ctx Context, ino Ino, flags uint8, length uint64, attr *Attr, skipGC bool) error
    // 回退（删除指定 offset 后的所有 Slice）
    Fallocate(ctx Context, ino Ino, mode uint8, off uint64, size uint64) error
    CopyFileRange(ctx Context, fino Ino, off uint64, tino Ino, toff uint64, size uint64, flags uint32, copied *uint64) error
```

> **重要**：`Read/Write` 操作的不是文件数据，而是**文件数据块的索引（Slice 列表）**。真实数据通过 `pkg/chunk/` 直接从对象存储读/写。

### 2.3 目录遍历

```go
    OpenDir(ctx Context, ino Ino, attr *Attr) (uint64, error)
    ReadDir(ctx Context, ino Ino, fh uint64, plus bool, entries *[]*Entry, limit int) error
    CloseDir(ctx Context, ino Ino, fh uint64) error
```

### 2.4 扩展属性 (xattr)

```go
    SetXattr(ctx Context, ino Ino, name string, value []byte, flags uint32) error
    GetXattr(ctx Context, ino Ino, name string, vbuff *[]byte) error
    ListXattr(ctx Context, ino Ino, dbuff *[]byte) error
    RemoveXattr(ctx Context, ino Ino, name string) error
```

### 2.5 文件锁

```go
    Flock(ctx Context, ino Ino, owner uint64, ltype uint32, block bool) error
    Getlk(ctx Context, ino Ino, owner uint64, ltype *uint32, start, end *uint64, pid *uint32) error
    Setlk(ctx Context, ino Ino, owner uint64, block bool, ltype uint32, start, end uint64, pid uint32) error
```

### 2.6 Session 管理

```go
    NewSession(ctx Context) error
    GetSession(sid uint64) (*Session, error)
    ListSessions() ([]*Session, error)
    CleanStaleSessions() error
    CheckStaleSessions(deadline time.Time) error
```

### 2.7 配额、回收站、统计

```go
    // 配额
    SetQuota(ctx Context, quota *Quota) error
    GetQuota(ctx Context, ino Ino, qtype uint8) (*Quota, error)
    DelQuota(ctx Context, ino Ino, qtype uint8) error

    // 回收站
    FindTrash(ctx Context, parent Ino, name string, inode *Ino) error
    RestoreTrash(ctx Context, ino Ino, parent Ino, name string) error
    CleanupTrash(ctx Context, before time.Time) error

    // 统计
    StatFS(ctx Context, totalspace, availspace, iused, iavail *uint64) error
    Summary(ctx Context, ino Ino, summary *Summary, recursive bool, strict bool) error
```

### 2.8 管理操作

```go
    Init(format *Format, force bool) error   // 初始化（format）
    Load(checkNew bool) (*Format, error)     // 加载已存在的卷
    DumpMeta(w io.Writer, root Ino) error    // 导出元数据
    Chroot(ctx Context, name string) error   // 切换根目录
    ...
```

## 三、Slice 树 — 文件数据组织

JuiceFS 用 BST（二叉搜索树）管理文件的 Slice 列表。每个文件的每个 Chunk 对应一棵 Slice 树：

```go
// pkg/meta/slice.go
type sliceTree struct {
    slices []Slice  // 有序的 Slice 列表
    tree   *RBTree  // 或 BST，加速查找
}
```

```
Chunk 0 (inode=100) 的 Slice 树:

     Slice{Off=0,  Len=2M}         ← offset 0 到 2M
          \
     Slice{Off=2M,  Len=5M}       ← offset 2M 到 7M
              \
     Slice{Off=7M,  Len=3M}       ← offset 7M 到 10M

查找 offset=8M 的数据 → 遍历树找到 Slice{Off=7M} → 计算 slice 内偏移 = 8M-7M=1M
```

> 详细分析见 [Slice 压缩与碎片整理](data-path/slice-compaction/SKILL.md)。

## 四、三大引擎实现 Meta 接口

| 引擎 | 实现类型 | 文件 | 行数 | 特点 |
|------|---------|------|------|------|
| Redis | `redisMeta` | `redis.go` | 6263 | Lua 事务脚本，支持 Cluster/Sentinel |
| SQL | `dbMeta` | `sql.go` | 6148 | XORM ORM，MySQL/PostgreSQL/SQLite |
| TKV | `kvMeta` | `tkv.go` | 5105 | 事务 KV，TiKV/etcd/Badger/FDB |

所有引擎的公共逻辑在 `baseEngine`（`pkg/meta/base.go`）中。

```go
// baseEngine 实现了许多 Meta 接口方法的公共部分
type baseEngine struct {
    conf     *Config
    fmt      *Format
    sid      uint64   // session ID
    ...
}
```

`redisMeta`、`dbMeta`、`kvMeta` 通过嵌入 `*baseEngine` 复用公共方法。

## 五、关键设计特性

### 5.1 reader/writer 分离

VFS 使用两个独立的 Meta 引用：
```go
type VFS struct {
    reader meta.Meta  // 只读路径
    writer meta.Meta  // 读写路径
}
```
正常情况下 reader == writer，但某些场景（如只读挂载）可以不同。

### 5.2 syscall 参数

很多方法有 `syscall bool` 参数：
- `syscall=true`：来自内核系统调用，需要严格检查权限
- `syscall=false`：来自内部操作（如 gc），跳过权限检查

### 5.3 Atime 更新策略

```go
Open(ctx Context, ino Ino, flags uint32, attr *Attr) error
```
`Open` 时更新 `Atime`，但可以配置 `AttrTimeout` 来缓存一段时间内的属性，避免频繁写元数据引擎。

## 六、关键代码位置

| 文件 | 核心内容 |
|------|---------|
| `pkg/meta/interface.go` | Ino, Attr, Slice, Meta 接口定义 |
| `pkg/meta/config.go` | Format, Config, ChunkConf 定义 |
| `pkg/meta/context.go` | Context 接口及其实现 |
| `pkg/meta/slice.go` | Slice 树实现 |
| `pkg/meta/base.go` | baseEngine 公共逻辑 |
| `pkg/meta/redis.go` | Redis 引擎实现 |
| `pkg/meta/sql.go` | SQL 引擎实现 |
| `pkg/meta/tkv.go` | TKV 引擎实现 |
| `pkg/meta/openfile.go` | 打开文件缓存管理 |

## 七、调试与排查

### 7.1 查看某个 inode 的元数据

```bash
juicefs info META-URL /path/to/file
# 输出: inode, mode, uid, gid, size, slices, ...
```

### 7.2 查看 volume 配置

```bash
juicefs status META-URL
# 输出: Name, UUID, Storage, BlockSize, Compression, Sessions, ...
```

### 7.3 用 dump 导出元数据

```bash
juicefs dump META-URL dump.json
# 导出为 JSON，可以查看完整的元数据树
```

## 八、常见问题

**Q: 为什么 Meta 接口有 100+ 方法？**
A: 因为要实现完整的 POSIX 文件系统语义（lookup, create, read, write, rename, link, symlink, xattr, lock, quota...），每个语义至少一个方法。

**Q: Read/Write 方法操作的是什么？**
A: 操作的是 Slice 索引（chunk ID + offset + length），不是文件数据。数据通过 ChunkStore 直接从对象存储读写。

**Q: 如何新增一个元数据引擎？**
A: 创建新类型并实现 Meta 接口的全部 100+ 方法。可以嵌入 `*baseEngine` 复用公共方法，只需实现引擎特有的方法（主要是 Lookup, Create, Read, Write 等核心操作）。

## 九、改进思路 / 贡献方向

1. **简化 Meta 接口** — 部分方法可以用默认实现+可选覆盖的方式减少冗余
2. **元数据缓存一致性** — 多个 mount 实例间的元数据缓存失效机制优化
3. **StatFS 准确性** — 当前 StatFS 的 `availspace` 基于对象存储配额，可能不准确
