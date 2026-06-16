---
name: juicefs-cli-command-system
description: JuiceFS命令行框架与子命令注册系统。当用户询问urfave/cli、命令注册、全局选项处理、main函数入口、子命令路由时调用此技能。
---

# JuiceFS 命令行框架

## 〇、为什么需要理解 CLI 框架？

JuiceFS 所有功能都以子命令的形式提供（`juicefs mount`, `juicefs format`, `juicefs gc`...）。理解 CLI 框架可以：
1. 快速定位某个命令的入口代码
2. 理解全局选项（--verbose, --debug）如何传递
3. 有能力添加新的调试命令
4. 理解 `mount -t juicefs` 系统挂载的兼容逻辑

## 一、核心数据结构

### 1.1 main.go — 纯入口

```go
// main.go
package main

import "github.com/juicedata/juicefs/cmd"

func main() {
    cmd.Main(os.Args)  // 全部委托给 cmd 包
}
```

JuiceFS 的 `main()` 是 Go 项目中最简洁的入口之一——没有信号处理，没有初始化，全部交给 `cmd.Main()`。

### 1.2 cmd/main.go — 真正的主函数

```go
// cmd/main.go
func Main(args []string) {
    // 1. 设置日志格式
    // 2. 注册所有子命令
    // 3. urfave/cli 解析参数并路由
    app := &cli.App{
        Name:    "juicefs",
        Usage:   "A POSIX file system...",
        Version: version.Version(),
        Commands: []*cli.Command{
            cmdFlags().format(),
            cmdFlags().config(),
            cmdFlags().mount(),
            cmdFlags().gateway(),
            // ... 30+ 子命令
        },
    }
    app.Run(args)
}
```

### 1.3 urfave/cli 框架核心类型

```go
// github.com/urfave/cli/v2
type App struct {
    Name     string
    Usage    string
    Version  string
    Commands []*Command
    Flags    []Flag
    Action   func(*Context) error
}

type Command struct {
    Name        string
    Aliases     []string
    Usage       string
    Flags       []Flag
    Subcommands []*Command
    Action      func(*Context) error  // 命令执行函数
}
```

## 二、核心流程

### 2.1 命令注册流程

```
main()
  → cmd.Main(os.Args)
    → new CLI App
    → cmdFlags() 结构体方法链生成各子命令的 Flags
    → 组装 []*cli.Command{format, config, mount, gateway, ...}
    → app.Run(args)
      → urfave/cli 解析 args[1] 匹配子命令名
      → 调用对应的 Action 函数
```

### 2.2 cmdFlags 结构体 — 统一的 flag 管理

```go
// cmd/main.go
type cmdFlags struct {
    // 通过方法链设置各子命令的 flags
}
```

每个子命令通过方法获得自己的 Flag 集合，例如 mount 命令的 flags：

```go
// cmd/mount.go 中的模式
func (f *cmdFlags) mount() *cli.Command {
    return &cli.Command{
        Name:      "mount",
        Usage:     "mount a volume",
        ArgsUsage: "META-URL MOUNTPOINT",
        Flags: []cli.Flag{
            // -o 选项对应 FUSE 挂载选项
            &cli.StringFlag{Name: "o"},
            // --enable-xattr
            &cli.BoolFlag{Name: "enable-xattr"},
            // --cache-dir
            &cli.StringFlag{Name: "cache-dir"},
            // --cache-size
            &cli.IntFlag{Name: "cache-size"},
            // ... 更多
        },
        Action: func(ctx *cli.Context) error {
            return mount(ctx)
        },
    }
}
```

### 2.3 系统挂载兼容

JuiceFS 也支持通过标准 `mount -t juicefs` 方式挂载。为此 `cmd/main.go` 有特殊的参数解析逻辑：

```go
// cmd/main.go (简化逻辑)
func Main(args []string) {
    // 检测是否是系统 mount 调用
    // mount -t juicefs META-URL MOUNTPOINT
    // 实际传入 args: ["mount.juicefs", "META-URL", "MOUNTPOINT"]
    // 需要重排为: ["juicefs", "mount", "META-URL", "MOUNTPOINT"]
    if strings.HasSuffix(args[0], "mount.juicefs") {
        args = []string{"juicefs", "mount", args[1], args[2]}
    }
    // 或者同时有 -o 选项
}
```

### 2.4 调试服务自动注入

Main() 在启动时为每个挂载实例自动注入 pprof 调试端口：

```go
// cmd/main.go
func Main(args []string) {
    // 启动 pprof debug agent (端口 6060-6099)
    // 启动 pyroscope profile agent（如果环境变量指定）
    // 设置 nofile rlimit
    // 启动一个 goroutine 等待 SIGUSR1 时 dump goroutine 堆栈
}
```

## 三、关键设计特性

### 3.1 Action 函数的统一签名

所有命令的 Action 都是 `func(*cli.Context) error`：
- 用 `ctx.Args().Get(0)` 等获取位置参数
- 用 `ctx.String("flag-name")` 获取选项值
- 返回 `error` 表示命令执行失败

### 3.2 错误退出

```go
func Main(args []string) {
    if err := app.Run(args); err != nil {
        logger.Fatalf("%s", err)  // 打印错误并 os.Exit(1)
    }
}
```

### 3.3 命令分组

30+ 命令按功能分为 5 类：

| 分组 | 命令 | 说明 |
|------|------|------|
| ADMIN | format, config, quota, destroy, gc, fsck, restore, dump, changelog, load | 卷生命周期管理 |
| SERVICE | mount, umount, gateway, webdav | 长期运行的服务 |
| TOOL | sync, bench, objbench, mdtest, warmup, rmr, clone, compact | 数据操作 |
| INSPECTOR | status, stats, profile, info, debug, summary | 诊断监控 |

### 3.4 Build Tags 控制编译

```go
//go:build !nogateway
// cmd/gateway.go - 只在未设置 nogateway 时编译

//go:build !noredis
// cmd/redis.go - 只在未设置 noredis 时编译
```

## 四、关键代码位置

| 文件 | 内容 | 关键行/函数 |
|------|------|-----------|
| `main.go` | 入口 | `main()` |
| `cmd/main.go` | App 组装 + Main() | `Main()`, `cmdFlags{}` |
| `cmd/mount.go` | mount 命令 | `mount()`, `mountAction()` |
| `cmd/format.go` | format 命令 | `format()`, `formatAction()` |
| `cmd/gateway.go` | gateway 命令 | `gateway()`, `gatewayAction()` |
| `cmd/sync.go` | sync 命令 | `sync()`, `syncAction()` |
| `cmd/gc.go` | gc 命令 | `gc()`, `gcAction()` |
| `cmd/debug.go` | debug 命令 | `debug()`, `debugAction()` |
| `cmd/status.go` | status 命令 | `status()`, `statusAction()` |
| `cmd/stats.go` | stats 命令 | `stats()`, `statsAction()` |

## 五、调试与排查

### 5.1 添加一个调试命令

如果你想临时添加一个命令来测试某个内部 API，可以在 `cmd/main.go` 的 `Commands` 数组中追加：

```go
{
    Name:  "debug-myfeature",
    Usage: "test my new feature",
    Action: func(ctx *cli.Context) error {
        // 你的测试代码
        return nil
    },
},
```

### 5.2 理解命令帮助

```bash
juicefs --help        # 列出所有子命令
juicefs mount --help  # 查看 mount 命令的选项
```

### 5.3 查看某个命令对应哪个文件

```bash
# 在 juicefs 源码目录
grep -r '"mount"' cmd/ | grep Command
```

## 六、常见问题

**Q: 为什么有 30+ 个 .go 文件在 cmd/ 下？**
A: 每个子命令一个文件（或一组文件），通过 urfave/cli 的 `Commands` 数组合并到一个 App 中。

**Q: cmdFlags 是什么设计模式？**
A: 这是"方法链"模式，集中管理所有子命令的 flag 定义，避免散落在各文件中难以维护。

**Q: 如何知道一个 flag 在哪个命令中可用？**
A: 阅读对应命令文件中的 Flag 定义，或运行 `juicefs <command> --help`。

## 七、改进思路 / 贡献方向

1. **添加 debug 子命令** — 暴露内部状态快照，方便问题排查
2. **改进 help 信息** — 为复杂命令添加更多使用示例
3. **命令补全** — 优化 `hack/autocomplete/` 下的 shell 补全脚本
