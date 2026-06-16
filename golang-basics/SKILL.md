---
name: golang-basics
description: Go语言实战速成，覆盖阅读JuiceFS源码所需的全部Go语法知识。当用户询问Go语法、接口、goroutine、channel、错误处理、context、包管理、build tags时调用此技能。
---

# Go 语言实战速成

> **目标**：读完这篇后，你能够在 IDE 中流畅阅读 JuiceFS 源码。不是从头学 Go，而是以 JuiceFS 代码为样本，解释每一个你会遇到的 Go 语法模式。

## 〇、为什么要先学 Go？

JuiceFS 是纯 Go 项目（`go.mod` 声明 module `github.com/juicedata/juicefs`），所有核心逻辑都在 `cmd/` 和 `pkg/` 目录下。如果不熟悉 Go 的以下特性，读源码时会频繁卡住：

- **interface** — JuiceFS 大量使用接口抽象（Meta、ObjectStorage、ChunkStore），不理解 interface 就看不懂分层设计
- **goroutine + channel** — 并发读写、缓存回写、心跳维持都依赖协程
- **error 处理** — Go 不使用 try/catch，错误沿调用链逐层返回
- **struct tag** — `Format` 配置结构使用 tag 做序列化映射
- **build tags** — `//go:build` 控制不同平台/特性的编译（noredis, nomysql, windows 等）

## 一、变量、类型、控制流（30 分钟）

### 1.1 变量声明

```go
var name string = "juicefs"     // 完整声明
name := "juicefs"               // 短声明（最常用，自动推断类型）
var port int                    // 零值初始化（int=0, string="", bool=false, pointer=nil）
```

JuiceFS 实例：
```go
// cmd/main.go
var logger = utils.GetLogger("juicefs")  // 包级变量，整个 cmd 包共享

// pkg/meta/config.go
func (f *Format) Update() {
    for k, v := range f.DirStats {  // range 遍历 map
        ...
    }
}
```

### 1.2 基本类型

```go
bool                          // true / false
string                        // 字符串（不可变）
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64  // 无符号（JuiceFS 中 Ino 就是 uint64）
float32, float64
byte                          // = uint8
rune                          // = int32（表示 Unicode 码点）
error                         // 内置接口类型
```

JuiceFS 核心类型别名（`pkg/meta/interface.go`）：
```go
type Ino uint64      // inode 号
type Fh uint64       // 文件句柄
type Attr struct {   // 文件属性
    Flags     uint8
    Typ       uint8
    Mode      uint16
    Uid, Gid  uint32
    Atime, Mtime, Ctime int64
    Nlink     uint32
    Length    uint64
    ...
}
```

### 1.3 控制流

```go
// if - 可以带初始化语句
if err := doSomething(); err != nil {
    return err
}

// switch - 不需要 break
switch typ {
case meta.TypeFile:
    // ...
case meta.TypeDirectory:
    // ...
default:
    // ...
}

// for - Go 只有 for 一种循环
for i := 0; i < 10; i++ { }          // 经典
for condition { }                      // while
for { }                               // 无限循环
for k, v := range mapOrSlice { }      // 遍历
```

### 1.4 函数

```go
func add(a, b int) int {          // 参数类型合并，返回值类型在最后
    return a + b
}

func lookup(name string) (int, error) {  // 多返回值（Go 的核心错误处理模式）
    if name == "" {
        return 0, fmt.Errorf("empty name")  // 错误用 fmt.Errorf
    }
    return 42, nil  // nil = 空
}

func (v *VFS) Read(ctx meta.Context, ino Ino, buf []byte, off uint64, fh Fh) (int, error) {
    // (v *VFS) 是接收者（receiver），相当于给 *VFS 类型定义了 Read 方法
}
```

## 二、结构体、接口、嵌入（关键章节，必读）

### 2.1 结构体 struct

```go
type Person struct {
    Name string   // 大写开头 = 公开（exported），小写 = 私有
    age  int      // 小写，包外不可见
}

p := Person{Name: "Alice", age: 30}  // 字面量初始化
p := &Person{Name: "Alice"}          // 返回指针
p.Name = "Bob"                       // 用 . 访问，指针会自动解引用
```

JuiceFS 实例（`pkg/meta/config.go`）：
```go
type Format struct {
    Name             string
    UUID             string
    Storage          string  // object storage URL
    BlockSize        int64
    Compression      string
    EncryptKey       string `json:",omitempty"`       // struct tag: json序列化时忽略空值
    TrashInterval    int    `json:",omitempty"`
    ...
}
```

Struct tag 常见用法：`json:"name,omitempty"`, `yaml:"name"`, `xml:"name"`。在 JuiceFS 中用于 JSON/YAML 序列化配置。

### 2.2 接口 interface（Go 多态的核心）

Go 的接口是**隐式实现**——不需要声明 `implements`，只要结构体的方法集匹配接口即可。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

// 任何有 Read([]byte) (int, error) 方法的类型都自动实现了 Reader
// 不需要写 "type File implements Reader"
```

JuiceFS 核心接口（`pkg/meta/interface.go`）：
```go
type Meta interface {
    // 文件系统操作
    Lookup(ctx Context, parent Ino, name string, inode *Ino, attr *Attr, syscall bool) error
    GetAttr(ctx Context, ino Ino, attr *Attr) error
    Create(ctx Context, parent Ino, name string, mode uint16, cumask uint16, flags uint32, inode *Ino, attr *Attr) error
    Open(ctx Context, ino Ino, flags uint32, attr *Attr) error
    Read(ctx Context, ino Ino, indx uint32, slices *[]Slice, syscall bool) error
    Write(ctx Context, ino Ino, indx uint32, off uint32, slice Slice, syscall bool) error
    // ... 100+ 方法
}
```

三个引擎家族各自实现 Meta 接口：
- `pkg/meta/redis.go` — `redisMeta` 实现了 Meta
- `pkg/meta/sql.go` — `dbMeta` 实现了 Meta
- `pkg/meta/tkv.go` — `kvMeta` 实现了 Meta

**理解 Go 多态的关键代码**（`pkg/vfs/vfs.go`）：
```go
type VFS struct {
    reader meta.Meta   // 只要是实现了 Meta 接口的类型都可以赋值
    writer meta.Meta   // 运行时决定具体是 redisMeta/dbMeta/kvMeta
    store  chunk.ChunkStore
    ...
}

// NewVFS 接受任何实现了 Meta 接口的对象
func NewVFS(m meta.Meta, store chunk.ChunkStore, cfg *Config) *VFS {
    return &VFS{reader: m, writer: m, store: store, ...}
}
```

### 2.3 嵌入（Embedding）—— Go 的"继承"替代方案

```go
type Logger struct {
    level int
}

func (l *Logger) Info(msg string) { ... }

type Server struct {
    Logger          // 匿名嵌入 —— Server 自动获得 Logger 的所有方法
    addr   string
}

s := &Server{addr: ":8080"}
s.Info("starting")  // 直接调用 Logger 的方法，无需 s.Logger.Info()
```

JuiceFS 实例（`pkg/meta/redis.go`）：
```go
type redisMeta struct {
    *sync.Mutex       // 嵌入互斥锁，redisMeta 直接拥有 Lock/Unlock 方法
    *baseEngine    // 嵌入基础引擎，复用公共的 Meta 方法实现
    // ... 特有字段
}
```

### 2.4 类型断言与类型转换

```go
var m Meta = &redisMeta{}

// 类型断言 —— 判断接口背后的具体类型
if rm, ok := m.(*redisMeta); ok {
    // rm 现在是 *redisMeta 类型
}

// 类型 switch
switch v := m.(type) {
case *redisMeta:
    // v 是 *redisMeta
case *dbMeta:
    // v 是 *dbMeta
}
```

## 三、并发：goroutine、channel、context

### 3.1 goroutine（轻量级线程）

```go
go func() {
    // 这段代码在独立的 goroutine 中运行
    doWork()
}()

// 主 goroutine 继续执行，不等待上面的 go func
```

JuiceFS 实例（`pkg/chunk/cached_store.go`）：
```go
// 后台写回缓存
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        case key := <-store.pending:
            store.upload(key)
        }
    }
}()
```

### 3.2 channel（goroutine 间通信）

```go
ch := make(chan int)         // 无缓冲 channel：发送方阻塞直到接收方就绪
ch := make(chan int, 10)     // 有缓冲 channel：缓冲区满之前不阻塞

ch <- 42                     // 发送
val := <-ch                  // 接收
close(ch)                    // 关闭，之后接收返回零值

// select —— 同时等待多个 channel
select {
case val := <-ch1:
    // 处理 ch1
case ch2 <- 42:
    // 发送到 ch2
case <-time.After(1*time.Second):
    // 超时
}
```

### 3.3 Context（取消信号 + 截止时间 + 值传递）

```go
ctx, cancel := context.WithCancel(parent)   // 可取消
ctx, cancel := context.WithTimeout(parent, 5*time.Second)  // 超时自动取消
ctx := context.WithValue(parent, key, value) // 传递请求范围的值

select {
case <-ctx.Done():      // ctx 被取消时 Done() 返回的 channel 关闭
    return ctx.Err()    // 返回取消原因
case result := <-work():
    return result
}
```

JuiceFS 中 `meta.Context` 是对 `context.Context` 的扩展（`pkg/meta/context.go`）：
```go
type Context interface {
    context.Context               // 嵌入标准 Context
    Pid() uint32                  // 请求进程 PID
    Uid() uint32                  // 用户 ID
    Gid() uint32                  // 组 ID
    SetValue(key, value interface{})
    WithValue(key, value interface{}) Context
    ...
}
```

### 3.4 sync 包常用工具

```go
var mu sync.Mutex       // 互斥锁
mu.Lock()
// 临界区
mu.Unlock()

var rw sync.RWMutex     // 读写锁（读共享，写独占）
rw.RLock()  / rw.RUnlock()
rw.Lock()   / rw.Unlock()

var wg sync.WaitGroup   // 等待一组 goroutine 完成
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}
wg.Wait()

sync.Once               // 只执行一次
var once sync.Once
once.Do(func() { initConnection() })

sync.Map                // 并发安全的 map
```

## 四、错误处理

Go 没有 try/catch，错误是**返回值**的一部分。

```go
func ReadFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("open %s: %w", path, err)  // %w 包装错误，保留错误链
    }
    defer f.Close()  // defer: 函数返回前执行

    data, err := io.ReadAll(f)
    if err != nil {
        return nil, fmt.Errorf("read %s: %w", path, err)
    }
    return data, nil
}
```

JuiceFS 错误处理模式：
```go
// 逐层传递，每层添加上下文
if err := m.checkPermission(ctx, ino, MODE_MASK); err != nil {
    return syscall.EACCES  // 返回系统调用错误码
}
```

常用错误检查函数：
```go
errors.Is(err, targetError)    // 检查错误链中是否有特定错误
errors.As(err, &target)        // 提取特定类型错误
```

## 五、包管理与项目结构

### 5.1 Go Modules

```go
// go.mod 文件
module github.com/juicedata/juicefs

go 1.25.0

require (
    github.com/hanwen/go-fuse/v2 v2.8.0
    github.com/redis/go-redis/v9 v9.12.0
    xorm.io/xorm v1.3.10
    ...
)
```

`go mod tidy` 自动整理依赖，`go mod vendor` 将依赖缓存到 vendor/ 目录。

### 5.2 包导入

```go
import (
    "fmt"                                    // 标准库
    "github.com/juicedata/juicefs/pkg/meta"  // 自己项目（完整 module 路径）
    "github.com/hanwen/go-fuse/v2/fuse"      // 第三方
)
```

### 5.3 可见性规则

- **大写开头** = 公开（exported），其他包可访问：`type VFS struct`, `func NewVFS()`
- **小写开头** = 私有，仅当前包可访问：`type page struct`, `func newPage()`
- 目录名 ≠ 包名。例如 `pkg/chunk/` 目录下的文件声明 `package chunk`

### 5.4 init() 函数

```go
// 包加载时自动执行，用于注册驱动
func init() {
    Register("redis", newRedisMeta)
    Register("mysql", newSQLMeta)
}
```

JuiceFS 实例：`pkg/object/object_storage.go` 中的 `init()` 注册所有对象存储后端。

## 六、Build Tags（条件编译）

JuiceFS 大量使用 build tags 控制编译内容：

```go
//go:build !windows
// +build !windows                    // 旧格式兼容

//go:build linux
//go:build darwin
//go:build !noredis
```

JuiceFS 中的 build tags 用法：

| Tag | 效果 |
|-----|------|
| `!noredis` | 默认编译 Redis 引擎 |
| `!nomysql` | 默认编译 MySQL 引擎 |
| `!nosqlite` | 默认编译 SQLite 引擎 |
| `!notikv` | 默认编译 TiKV 引擎 |
| `!nogateway` | 默认编译 S3 Gateway |
| `!nowebdav` | 默认编译 WebDAV |
| `windows` | Windows 专用文件 |
| `linux` | Linux 专用文件 |
| `darwin` | macOS 专用文件 |

Makefile 中的精简构建：
```makefile
juicefs.lite:     # nogateway nowebdav noredis nosqlite nomysql nopg notikv ...
juicefs.ceph:     # 包含 Ceph RADOS 支持
juicefs.fdb:      # 包含 FoundationDB 支持
juicefs.gluster:  # 包含 GlusterFS 支持
```

## 七、Go 在 JuiceFS 中的惯用模式

### 7.1 NewXxx() 构造函数模式

```go
// Go 没有构造函数，习惯用 New+T 函数
func NewVFS(m meta.Meta, store chunk.ChunkStore, cfg *Config) *VFS
func NewCachedStore(object object.ObjectStorage, conf Config) (ChunkStore, error)
func NewRedisMeta(address string, conf *Config) (Meta, error)
```

### 7.2 函数式选项模式

```go
// 用于灵活配置，不是 JuiceFS 本身的模式，但 Go 生态常见
type Option func(*Config)

func WithLogger(logger *Logger) Option {
    return func(c *Config) { c.logger = logger }
}
```

### 7.3 defer 资源清理

```go
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()  // 确保文件在函数退出时关闭

mu.Lock()
defer mu.Unlock()  // 确保锁释放
```

### 7.4 接口+工厂模式（对象存储层）

```go
// pkg/object/object_storage.go
var storages = make(map[string]ObjectStorage)

func Register(name string, s ObjectStorage) {
    storages[name] = s
}

func CreateStorage(uri string) (ObjectStorage, error) {
    // 解析 URI，查找 storages 注册表，创建对应后端
}
```

## 八、阅读 JuiceFS 代码的速查表

| 遇到什么 | 去向哪里 |
|---------|---------|
| `type XXX interface { ... }` | 这是抽象层，定义了一组行为规范 |
| `func (x *Type) Method() ...` | 这是 Method，对 Type 类型添加方法 |
| `var _ Interface = (*Type)(nil)` | 编译时断言 Type 实现了 Interface |
| `go func() { ... }()` | 启动了一个新的 goroutine |
| `<-chan` / `chan<- ` | 从 channel 接收 / 向 channel 发送 |
| `ctx.Done()` | 检查 context 是否被取消 |
| `return ..., err` | 每层都要检查 err，这是 Go 的纪律 |
| `//go:build !linux` | 条件编译，此文件只在非 Linux 平台编译 |
| `defer f.Close()` | 函数返回前执行 Close |

## 九、推荐学习资源

1. **官方 Tour of Go** — https://go.dev/tour/ （2-3 小时过一遍）
2. **Effective Go** — https://go.dev/doc/effective_go （深入理解 Go 惯用法）
3. **Go by Example** — https://gobyexample.com/ （按主题查语法）
4. **直接在 JuiceFS 源码中学习** — 打开 `pkg/vfs/vfs.go`，对照本文逐行阅读，Go 语法在实战中最容易理解

## 十、关键代码位置

| 需要看的文件 | 学习目标 |
|-------------|---------|
| `pkg/meta/interface.go` | 理解 Go interface 和类型别名 |
| `pkg/vfs/vfs.go:1-100` | 理解 struct、method、构造函数模式 |
| `cmd/main.go` | 理解包导入、init、main 入口 |
| `pkg/chunk/cached_store.go` | 理解 goroutine + channel 实战用法 |
| `pkg/object/object_storage.go` | 理解接口+注册+工厂模式 |
| `go.mod` | 理解 Go modules 依赖管理 |
