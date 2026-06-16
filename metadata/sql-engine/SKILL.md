---
name: juicefs-sql-engine
description: JuiceFS SQL元数据引擎实现（MySQL/PostgreSQL/SQLite）。当用户询问SQL引擎、XORM框架、数据库表设计、事务隔离、MySQL与PostgreSQL差异时调用此技能。
---

# SQL 元数据引擎实现

## 〇、为什么需要 SQL 引擎？

Redis 引擎性能好但受内存限制。当元数据规模超过 Redis 内存容量时，需要基于磁盘的存储方案。SQL 引擎（MySQL/PostgreSQL/SQLite）提供了：

| 优势 | 说明 |
|------|------|
| 磁盘存储 | 可承载 PB 级别的元数据 |
| 成熟的运维生态 | 备份、主从复制、监控工具丰富 |
| SQL 查询 | 可以直接用 SQL 分析元数据 |

## 一、核心数据结构

### 1.1 dbMeta

```go
// pkg/meta/sql.go
type dbMeta struct {
    *sync.Mutex
    *baseEngine
    engine *xorm.Engine       // XORM ORM 引擎
    db     *sql.DB             // 原生 database/sql（用于需要精细控制的场景）
    driver string              // "mysql" / "postgres" / "sqlite3"
    ...
}
```

### 1.2 数据库表设计（三表核心 + 多辅助表）

#### 表 1：jfs_node — 节点属性

```sql
CREATE TABLE jfs_node (
    inode    BIGINT PRIMARY KEY,  -- inode 号
    flags    TINYINT,             -- 标志位
    type     TINYINT,             -- 类型 (1=file, 2=dir, 3=symlink...)
    mode     SMALLINT,            -- 权限位
    uid      INT,
    gid      INT,
    atime    BIGINT,              -- 访问时间（秒）
    mtime    BIGINT,              -- 修改时间（秒）
    ctime    BIGINT,              -- 状态变更时间（秒）
    atimensec INT,                -- atime 纳秒部分
    mtimensec INT,
    ctimensec INT,
    nlink    INT,                 -- 硬链接数
    length   BIGINT,              -- 文件大小
    rdev     INT,                 -- 设备号
    parent   BIGINT               -- 父目录 inode (用于 readdir)
);
```

#### 表 2：jfs_edge — 目录项（父子关系）

```sql
CREATE TABLE jfs_edge (
    parent   BIGINT,              -- 父目录 inode
    name     VARCHAR(255),        -- 文件名
    inode    BIGINT,              -- 子节点 inode
    PRIMARY KEY (parent, name)
);
```

#### 表 3：jfs_symlink — 符号链接目标

```sql
CREATE TABLE jfs_symlink (
    inode    BIGINT PRIMARY KEY,
    target   TEXT                 -- 链接目标路径
);
```

#### 辅助表

| 表名 | 用途 | 关键列 |
|------|------|--------|
| `jfs_chunk` | Slice 索引 | inode, indx (chunk index), slices (JSON/Blob) |
| `jfs_xattr` | 扩展属性 | inode, name, value |
| `jfs_flock` | 文件锁 | inode, sid, owner, ltype, start, end, pid |
| `jfs_plock` | POSIX 锁 | inode, sid, owner, start, end, pid |
| `jfs_session` | 客户端 session | sid, heartbeat, info (JSON) |
| `jfs_session2` | Session 打开文件引用 | sid, inode, count |
| `jfs_quota` | 配额 | inode, qtype, max_space, max_inodes, ... |
| `jfs_delfile` | 删除文件（回收站） | inode, length, expire |
| `jfs_setting` | 卷配置 | name, value (Format JSON) |
| `jfs_sustained` | 持续统计 | inode, ... |

### 1.3 Slice 存储格式

在 SQL 引擎中，一个 Chunk 的所有 Slice 以**二进制格式**存储在一个字段中：

```go
// 编码：{4B chunkid, 4B size, 4B off, 4B len} × N
// 或 JSON 格式（可配置）
func marshalSlices(slices []Slice) []byte { ... }
func unmarshalSlices(data []byte) []Slice { ... }
```

## 二、核心流程

### 2.1 Lookup 流程

```go
// pkg/meta/sql.go: Lookup
func (m *dbMeta) Lookup(ctx Context, parent Ino, name string, inode *Ino, attr *Attr, syscall bool) error {
    // 1. 查 jfs_edge 表获取 child inode
    var edge edge
    has, err := m.engine.Where("parent=? AND name=?", parent, name).Get(&edge)
    if !has {
        return syscall.ENOENT
    }
    
    // 2. 查 jfs_node 表获取属性
    var node node
    _, err = m.engine.ID(edge.Inode).Get(&node)
    
    // 3. 填充返回值
    *inode = node.Inode
    node.toAttr(attr)
    return nil
}
```

### 2.2 Create 流程（事务关键）

```go
func (m *dbMeta) Create(ctx Context, parent Ino, name string, ...) error {
    // 事务保护：一次 Create 需要同时修改 jfs_edge 和 jfs_node
    return m.txn(func(s *xorm.Session) error {
        // 1. 检查父目录是否存在且有写权限
        var pNode node
        if has, _ := s.ID(parent).Get(&pNode); !has {
            return syscall.ENOENT
        }
        
        // 2. 检查 name 是否已存在
        if has, _ := s.Where("parent=? AND name=?", parent, name).Exist(&edge{}); has {
            return syscall.EEXIST
        }
        
        // 3. 生成新 inode 号（自增序列）
        newIno := m.nextInode()
        
        // 4. 插入 jfs_node
        n := node{Inode: newIno, Type: TypeFile, Mode: mode, ...}
        s.Insert(&n)
        
        // 5. 插入 jfs_edge
        e := edge{Parent: parent, Name: name, Inode: newIno}
        s.Insert(&e)
        
        // 6. 更新父目录属性
        s.ID(parent).Cols("mtime", "ctime", "nlink").Update(...)
        
        return nil
    })
}
```

### 2.3 事务处理模式

```go
// SQL 引擎包装了 XORM 的事务
func (m *dbMeta) txn(f func(*xorm.Session) error) error {
    session := m.engine.NewSession()
    defer session.Close()
    
    session.Begin()
    if err := f(session); err != nil {
        session.Rollback()
        return err
    }
    return session.Commit()
}
```

### 2.4 Slice Read/Write

与 Redis 不同，SQL 引擎将整个 Chunk 的所有 Slice 存储在一个 `jfs_chunk` 行中：

```go
func (m *dbMeta) Read(ctx Context, ino Ino, indx uint32, slices *[]Slice, syscall bool) error {
    var chunk chunk
    has, _ := m.engine.Where("inode=? AND indx=?", ino, indx).Get(&chunk)
    if !has {
        return nil  // 空 chunk，返回空 Slice 列表
    }
    *slices = unmarshalSlices(chunk.Slices)
    return nil
}

func (m *dbMeta) Write(ctx Context, ino Ino, indx uint32, off uint32, slice Slice, syscall bool) error {
    return m.txn(func(s *xorm.Session) error {
        // 1. 读现有 Slice 列表
        var chunk chunk
        has, _ := s.Where("inode=? AND indx=?", ino, indx).Get(&chunk)
        
        // 2. 插入新 Slice（保持按 offset 排序）
        slices := unmarshalSlices(chunk.Slices)
        slices = insertSlice(slices, slice)
        
        // 3. 回写
        chunk.Slices = marshalSlices(slices)
        if has {
            s.ID(chunkKey{ino, indx}).Update(&chunk)
        } else {
            s.Insert(&chunk)
        }
        return nil
    })
}
```

## 三、关键设计特性

### 3.1 MySQL vs PostgreSQL vs SQLite 差异

| 特性 | MySQL | PostgreSQL | SQLite |
|------|-------|-----------|--------|
| 主键 | BIGINT AUTO_INCREMENT | BIGSERIAL | INTEGER PRIMARY KEY AUTOINCREMENT |
| 索引 hint | USE INDEX | 不需要 | 不需要 |
| 事务隔离 | REPEATABLE READ (默认) | READ COMMITTED | SERIALIZABLE |
| 条件 Upsert | ON DUPLICATE KEY UPDATE | ON CONFLICT DO UPDATE | INSERT OR REPLACE |
| 连接参数 | `parseTime=true` | `sslmode=disable` | `_journal_mode=WAL` |
| 适合场景 | 生产大规模 | 生产大规模 | 单机测试/嵌入式 |

### 3.2 连接管理

```go
// MySQL
engine, _ := xorm.NewEngine("mysql", "user:pass@tcp(host:3306)/db?parseTime=true&charset=utf8mb4")

// PostgreSQL
engine, _ := xorm.NewEngine("postgres", "host=localhost port=5432 user=... dbname=... sslmode=disable")

// SQLite
engine, _ := xorm.NewEngine("sqlite3", "file:juicefs.db?cache=shared&_journal_mode=WAL")
```

### 3.3 性能优化

1. **批量操作**：`ReadDir` 用一次 SQL 查询取回所有条目
2. **批量 Slice 读**：`Read`（读 Slice 索引）直接查 `jfs_chunk` 表，无需遍历多个 key
3. **连接池**：XORM 管理连接池，避免频繁建连

## 四、关键代码位置

| 文件 | 内容 | 关键区域 |
|------|------|---------|
| `pkg/meta/sql.go` | dbMeta 核心实现（6148 行） | `dbMeta`, `Lookup`, `Create`, `Read`, `Write`, `txn()` |
| `pkg/meta/sql_mysql.go` | MySQL 特有逻辑 | 建表语句、索引建议 |
| `pkg/meta/sql_pg.go` | PostgreSQL 特有逻辑 | PL/pgSQL 函数、序列 |
| `pkg/meta/sql_sqlite.go` | SQLite 特有逻辑 | WAL 模式、事务处理 |
| `pkg/meta/sql_lock.go` | SQL 引擎的锁实现 | `Flock`, `Setlk`, `Getlk` |

## 五、调试与排查

### 5.1 直接查数据库

```bash
# MySQL
mysql> USE juicefs;
mysql> SELECT * FROM jfs_node WHERE inode = 1;   # 看根目录
mysql> SELECT * FROM jfs_edge WHERE parent = 1;  # 看根目录下的文件
mysql> SELECT * FROM jfs_chunk WHERE inode = 100; # 看文件数据块
mysql> SELECT COUNT(*) FROM jfs_node;             # 总节点数

# PostgreSQL
psql> \c juicefs
psql> SELECT * FROM jfs_node WHERE inode = 1;

# SQLite
sqlite3 juicefs.db "SELECT * FROM jfs_node WHERE inode=1;"
```

### 5.2 慢查询分析

```bash
# MySQL
mysql> SHOW VARIABLES LIKE 'slow_query%';
mysql> SELECT * FROM mysql.slow_log;

# PostgreSQL
psql> SELECT * FROM pg_stat_statements ORDER BY mean_exec_time DESC;
```

### 5.3 事务死锁排查

```bash
# MySQL
mysql> SHOW ENGINE INNODB STATUS\G  # 查看最近死锁

# PostgreSQL
psql> SELECT * FROM pg_locks WHERE NOT granted;
```

## 六、常见问题

**Q: SQLite 可以用于生产吗？**
A: 官方推荐单机或测试场景使用。SQLite 不支持网络并发写入，多客户端 mount 同一 SQLite 文件会出问题。

**Q: MySQL/PostgreSQL 需要什么配置？**
A: 至少 `READ COMMITTED` 隔离级别（PostgreSQL 默认即可，MySQL 需要确认）。推荐开启 binlog 用于恢复。

**Q: Slice 数据膨胀问题？**
A: 大量小文件的频繁写入会导致 `jfs_chunk` 表增长，`juicefs gc` 的 compact 子命令可以压缩 Slice 减少行数。

## 七、改进思路 / 贡献方向

1. **迁移工具** — 从 Redis 迁移到 SQL 引擎的数据导出/导入工具
2. **查询优化** — 在特定场景下添加数据库索引建议
3. **PostgreSQL 特性利用** — 利用 PG 的 CTE/窗口函数优化复杂元数据查询
4. **SQLite WAL 模式改进** — 多读取者场景下的性能调优
