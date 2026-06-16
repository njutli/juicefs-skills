---
name: juicefs-redis-engine
description: JuiceFS Redis元数据引擎实现。当用户询问Redis引擎、Lua事务脚本、Redis Cluster/Sentinel、Session心跳、Client Side Caching时调用此技能。
---

# Redis 元数据引擎实现

## 〇、为什么选择 Redis 作为元数据引擎？

Redis 是 JuiceFS 社区版**默认推荐**的元数据引擎，因为：

| 优势 | 说明 |
|------|------|
| 极致性能 | 纯内存操作，亚毫秒级延迟 |
| 原子操作 | Lua 脚本天然支持事务，一条命令完成 Lookup+GetAttr |
| 丰富数据结构 | Hash/String/List/Set/ZSet 直接映射文件系统元数据 |
| 高可用 | Sentinel 自动故障转移，Cluster 分片扩展 |

**代价**：
- 内存存元数据，元数据量受限于 Redis 的内存
- AOF/RDB 持久化有数据丢失窗口（但 JuiceFS 的设计容忍这一点）

## 一、核心数据结构

### 1.1 redisMeta

```go
// pkg/meta/redis.go
type redisMeta struct {
    *sync.Mutex             // 嵌入互斥锁
    *baseEngine             // 嵌入基础引擎（公共方法）
    rd         redis.UniversalClient  // Redis 客户端（单机/Cluster/Sentinel 统一接口）
}
```

`redis.UniversalClient` 是 go-redis 的统一接口，同时支持单机、Cluster、Sentinel 三种模式。

### 1.2 Redis Key 设计

JuiceFS 使用以下 Key 命名规范（通过 `prefix()` 方法加前缀，避免多卷冲突）：

| 用途 | Redis 类型 | Key 格式 | Value 说明 |
|------|-----------|---------|-----------|
| 系统设置 | String | `{prefix}setting` | JSON(Format) |
| Session | Hash | `{prefix}sessions` | sid → SessionInfo JSON |
| 节点属性 | Hash | `{prefix}i{ino}` | field → Attr 各字段值 |
| 目录项 | Hash | `{prefix}d{parentIno}` | name → childIno |
| Slice 索引 | Sorted Set | `{prefix}c{ino}_{chunkIndex}` | offset → Slice{chunkid,size,off,len} |
| 符号链接 | String | `{prefix}s{ino}` | 目标路径 |
| 扩展属性 | Hash | `{prefix}x{ino}` | name → value |
| 文件锁 | Hash | `{prefix}l{ino}` | owner → lock info |
| 配额 | Hash | `{prefix}q{ino}` | type → quota info |
| 删除文件 | Hash | `{prefix}t{parentIno}` | name → inode (用于回收站) |
| Session 引用计数 | Sorted Set | `{prefix}ss{sid}` | inode → count (打开的文件) |

### 1.3 Lua 脚本模式

redisMeta 大量使用 Lua 脚本将多个 Redis 命令打包为一个原子操作。脚本存储在 `pkg/meta/redis.go` 的一开始：

```go
// pkg/meta/redis.go (前 200 行都是 Lua 脚本定义)
var scriptLookup = redis.NewScript(`
    local ino = redis.call('HGET', KEYS[1], ARGV[1])  -- 查目录项
    if not ino then
        return errno(ENOENT)
    end
    local attr = redis.call('HGETALL', KEYS[2] .. tostring(ino))  -- 查节点属性
    return {ino, attr}
`)
```

**Lua 脚本事务模式的优势：**
1. 避免多次网络往返（Lookup 一次往返 vs 至少 2 次）
2. 保证原子性（Lua 脚本在 Redis 中是原子执行的）
3. 错误检查在 Redis 服务端完成

## 二、核心流程

### 2.1 Lookup 流程

```go
// pkg/meta/redis.go: lookup
func (m *redisMeta) Lookup(ctx Context, parent Ino, name string, inode *Ino, attr *Attr, syscall bool) error {
    // 1. 构造 Lua 脚本参数
    keys := []string{
        m.entryKey(parent),   // "prefix d{parent}"
        m.inodeKey(),         // "prefix i"   (prefix，后面拼 ino)
    }
    args := []interface{}{name, ...}
    
    // 2. 执行 Lua 脚本（单次 Redis 往返）
    res, err := m.rd.Eval(ctx, scriptLookup.Hash(), keys, args).Result()
    
    // 3. 解析结果
    *inode = Ino(res[0].(int64))
    m.parseAttr(res[1].([]interface{}), attr)
    
    // 4. 如果配置了 Client Side Caching，缓存结果
    if m.csc != nil {
        m.csc.Cache(inode, attr, timeout)
    }
    
    return nil
}
```

### 2.2 Create 流程

```go
func (m *redisMeta) Create(ctx Context, parent Ino, name string, mode uint16, ...) error {
    // Lua 脚本一次完成：
    // 1. 检查父目录是否存在、是否有写权限
    // 2. 检查 name 是否已存在
    // 3. 检查父目录容量（配额检查）
    // 4. 分配新的 inode 号（自增计数器）
    // 5. HSET d{parent} name childIno
    // 6. HMSET i{newIno} mode uid gid mtime ctime nlink ...
    // 7. 更新父目录的 mtime/ctime
    // 8. 增加父目录的 nlink
    // 9. 统计信息更新
}
```

### 2.3 Read（读 Slice 索引）流程

```go
func (m *redisMeta) Read(ctx Context, ino Ino, indx uint32, slices *[]Slice, syscall bool) error {
    // ZRANGEBYSCORE c{ino}_{indx} 0 +inf
    // 返回 Slice 列表，按 offset 升序
    results := m.rd.ZRangeByScoreWithScores(ctx, m.sliceKey(ino, indx), ...).Val()
    for _, z := range results {
        // 解析 {Chunkid, Size, Off, Len}
        s := parseSlice(z.Member.(string))
        *slices = append(*slices, s)
    }
}
```

### 2.4 Session 管理与心跳

```go
// NewSession — 注册新 session
func (m *redisMeta) NewSession(ctx Context) error {
    // Lua 脚本：
    // 1. 生成递增的 session ID
    // 2. HSET sessions sid sessionInfo
    // 3. 启动心跳 goroutine
    go m.heartbeat()
    return nil
}

// heartbeat — 定期更新 session 心跳
func (m *redisMeta) heartbeat() {
    ticker := time.NewTicker(m.conf.Heartbeat)
    for range ticker.C {
        m.rd.HSet(ctx, m.sessionKey(), m.sid, m.sessionInfo())
        // 同时检查是否有其他 session 断开
        m.cleanStaleSessions()
    }
}
```

### 2.5 Client Side Caching (CSC)

```go
// pkg/meta/redis_csc.go
// 利用 Redis 6.0+ 的 Client Side Caching 加速 Lookup/GetAttr
// Redis 客户端维护本地缓存，服务端推送失效通知
type clientSideCache struct {
    cache *lru.Cache  // 本地 LRU 缓存
    ...
}
```

## 三、关键设计特性

### 3.1 Redis Cluster 支持

当配置了多个 Redis 地址时，使用 `redis.NewClusterClient()`：
```go
// hash tag 确保相关 key 在同一个 slot
// {prefix} 就是 hash tag，保证同一卷的所有 key 在同一节点
const hashTag = "{%s}"
```

### 3.2 Sentinel 自动故障转移

配置 sentinel 地址后，go-redis 自动处理主从切换：
```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "mymaster",
    SentinelAddrs: []string{":26379"},
})
```

### 3.3 Redis + AOF 持久化模式

生产环境强烈建议开启 AOF (`appendonly yes`)，JuiceFS 容忍在极端情况下 AOF 丢失少量最新写入（因为数据在对象存储中是不变的，最多丢失 Slice 索引）。

## 四、关键代码位置

| 文件 | 内容 | 关键区域 |
|------|------|---------|
| `pkg/meta/redis.go` | Lua 脚本定义（前 200 行）+ 所有方法实现 | `scriptLookup`, `scriptCreate`, `scriptWrite` |
| `pkg/meta/redis_csc.go` | Client Side Caching | `clientSideCache` |
| `pkg/meta/redis_lock.go` | 文件锁实现（基于 Redis 锁） | `Flock`, `Setlk` |

## 五、调试与排查

### 5.1 直接看 Redis 中的数据

```bash
# 查看所有 key
redis-cli KEYS "*"

# 查看某个 inode 的属性
redis-cli HGETALL "prefix i100"

# 查看某个目录的内容
redis-cli HGETALL "prefix d1"

# 查看 Slice 索引
redis-cli ZRANGE "prefix c100_0" 0 -1 WITHSCORES

# 查看当前 sessions
redis-cli HGETALL "prefix sessions"

# 监控实时命令
redis-cli MONITOR
```

### 5.2 监控 Redis 性能

```bash
# JuiceFS 内置的 stats 命令
juicefs stats META-URL

# 查看 Redis 的慢查询日志
redis-cli SLOWLOG GET 10
```

### 5.3 常见问题检测

```bash
# 检查 Redis 连接
redis-cli ping

# 检查内存使用（小心 OOM）
redis-cli INFO memory | grep used_memory_human

# 检查 key 数量
redis-cli DBSIZE
```

## 六、常见问题

**Q: Redis 出故障后挂载点会怎样？**
A: Session 心跳停止后，其他客户端会在 `Heartbeat * 3` 时间内检测到该 session 断开。断开 session 的未完成操作会被丢弃，已写入的 Slice 索引保留。

**Q: Redis 内存不够怎么办？**
A: 切换到 SQL（MySQL/PostgreSQL）引擎，或增大 Redis 内存。每个文件/目录大约占用几百字节到几 KB（取决于 Slice 数量）。

**Q: Lua 脚本报错 "NOSCRIPT No matching script"？**
A: go-redis 的 `Eval` 会自动先用 `EVALSHA` 尝试，失败后用 `EVAL`。如果 Redis 重启导致脚本缓存清空，会自动重新加载。

## 七、改进思路 / 贡献方向

1. **优化大量小文件场景** — 小文件 Slice 很少，可以用更紧凑的存储格式
2. **Lua 脚本错误处理** — 部分脚本的错误信息不够友好，可以改进
3. **Client Side Caching 的命中率监控** — 添加 metrics

## 八、参考文献

- Redis 官方文档：https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
- go-redis 文档：https://redis.uptrace.dev/
