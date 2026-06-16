---
name: juicefs-debug-tools
description: JuiceFS调试工具链与代码贡献指南。当用户询问调试、pprof、profile、stats、debug、dump、问题定位、Issue/PR流程、CONTRIBUTING时调用此技能。
---

# 调试工具与代码贡献指南

## 〇、目标

这篇 Skill 的目标是让你具备**独立定位 JuiceFS Bug 并提交代码 PR** 的能力。内容分为两部分：
1. **调试工具链** — 所有可用的诊断命令和工具
2. **贡献流程** — 从发现 Issue 到 Merge 的完整流程

## 一、调试工具链全景

```
┌─────────────────────────────────────────────────────────────┐
│                    JuiceFS 调试工具                          │
├───────────────┬─────────────────────────────────────────────┤
│  health check │ juicefs status META-URL                     │
│               │ juicefs fsck META-URL                       │
├───────────────┼─────────────────────────────────────────────┤
│  metadata     │ juicefs info META-URL /path                 │
│  inspection   │ juicefs dump META-URL dump.json             │
│               │ juicefs summary META-URL /path              │
├───────────────┼─────────────────────────────────────────────┤
│  performance  │ juicefs stats META-URL                      │
│  analysis     │ juicefs profile META-URL                    │
│               │ curl localhost:6060/debug/pprof/            │
├───────────────┼─────────────────────────────────────────────┤
│  debug        │ juicefs debug META-URL /path                │
│  collection   │ juicefs changelog META-URL                  │
├───────────────┼─────────────────────────────────────────────┤
│  internal     │ cat /mnt/jfs/.stats                         │
│  files        │ cat /mnt/jfs/.accesslog                     │
│               │ echo "cmd" > /mnt/jfs/.control              │
└───────────────┴─────────────────────────────────────────────┘
```

### 1.1 `juicefs status` — 查看卷状态

```bash
juicefs status redis://localhost/1
# 输出:
# {
#   "Name": "myvol",
#   "UUID": "xxx-xxx-xxx",
#   "Storage": "minio://...",
#   "BlockSize": 4096,
#   "Compression": "none",
#   "Partitions": 0,
#   "Sessions": [
#     {
#       "Sid": 1001,
#       "Heartbeat": "2024-01-01T12:00:00Z",
#       "Hostname": "server1",
#       "MountPoint": "/mnt/jfs",
#       "Version": "1.2.0"
#     }
#   ]
# }
```

何时使用：挂载有问题时，先看 status 确认卷状态和 session 信息。

### 1.2 `juicefs fsck` — 元数据一致性检查

```bash
juicefs fsck redis://localhost/1

# 检查内容：
# - 所有 Slice 引用的 chunk 是否存在于对象存储
# - 文件大小与 Slice 总长度是否一致
# - 目录项的父子关系是否完整
# - 硬链接计数是否正确
# - 回收站中的文件是否过期
```

何时使用：怀疑数据不一致（文件读不到、大小不对、目录少文件）。

### 1.3 `juicefs info` — 查看文件/目录详情

```bash
juicefs info redis://localhost/1 /path/to/file
# 输出:
#   inode: 100
#   files: 1
#   dirs:  0
#   length: 1.00 GiB (1073741824 Bytes)
#   size:   1.05 GiB (1127428915 Bytes)    ← 对象存储中的实际大小
#   slices: 32
#   chunks: 1

juicefs info redis://localhost/1 /path/to/dir --recursive
# 递归显示目录下所有文件和子目录的汇总
```

何时使用：查看文件的 Slice 分布、实际存储大小、碎片化程度。

### 1.4 `juicefs dump` — 导出元数据

```bash
# 导出完整元数据到 JSON
juicefs dump redis://localhost/1 meta.json

# JSON 格式包含:
# - 所有文件/目录的 inode、属性、父子关系
# - 所有 Slice 索引
# - 所有 xattr、链接

# 重新加载元数据（迁移引擎）
juicefs load redis://localhost/1 meta.json
```

何时使用：迁移元数据引擎、离线分析元数据、备份。

### 1.5 `juicefs stats` — 实时性能监控

```bash
juicefs stats redis://localhost/1

# 实时输出:
# 2024/01/01 12:00:00        ops     read(MB/s)    write(MB/s)    cache(MB)
# 2024/01/01 12:00:01   1023        45.2          12.8           512
# 2024/01/01 12:00:02    982        43.1          11.5           514
```

何时使用：性能问题排查、容量规划、观察读写模式。

### 1.6 `juicefs profile` — 访问模式分析

```bash
# 分析某个进程的访问模式
juicefs profile META-URL --pid 12345

# 输出 Top-N 读取/写入最多的文件
# 输出文件大小分布
# 输出 I/O 模式（顺序 vs 随机）
```

何时使用：优化应用的数据布局、发现热点文件。

### 1.7 `juicefs debug` — 收集调试信息

```bash
juicefs debug META-URL /mnt/jfs

# 自动收集:
# - 系统信息（OS, 内核版本, 内存, 磁盘）
# - JuiceFS 日志
# - pprof goroutine dump
# - pprof heap dump
# - mount 进程的 stack trace
# - 打包为 tar.gz 用于提交 Issue
```

### 1.8 pprof 深度分析

每个 mount 实例自动开启 pprof HTTP 端口（6060-6099，自动选择未占用端口）：

```bash
# 查看 goroutine 状态（检测死锁/泄漏）
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# 查看堆内存分配
curl http://localhost:6060/debug/pprof/heap

# CPU 采样 30 秒
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof cpu.prof
(pprof) top
(pprof) list Read

# 查看内存分配
curl http://localhost:6060/debug/pprof/allocs > allocs.prof
go tool pprof allocs.prof

# 在线 Web 查看（需要 graphviz）
go tool pprof -http=:8080 cpu.prof
```

### 1.9 内部文件

```bash
# .stats — 运行统计（只读）
cat /mnt/jfs/.stats
# {"fuseOps":12345, "readBytes":1073741824, "writeBytes":536870912, ...}

# .accesslog — 访问日志（只读，需要挂载时指定 --access-log）
cat /mnt/jfs/.accesslog
# [2024-01-01 12:00:00] [12345] [1000:1000] [OPEN] [100] [/data/file.csv] [0.002s]

# .config — 卷配置（可读写）
cat /mnt/jfs/.config      # 查看配置
# 修改配置（如缓存大小）:
echo '{"CacheSize": 204800}' > /mnt/jfs/.config

# .control — 控制命令（可写）
echo "compact" > /mnt/jfs/.control      # 触发 Slice 压缩
echo "gc.start" > /mnt/jfs/.control     # 触发垃圾回收
echo "backup.start" > /mnt/jfs/.control # 触发元数据备份
```

## 二、日志体系

```go
// JuiceFS 使用 logrus 风格的日志
utils.GetLogger("juicefs")  // 获取带模块名的 logger

// 日志级别
// --verbose  →  INFO
// --debug    →  DEBUG (verbose)
// --trace    →  TRACE (最详细，输出每个 FUSE 操作)
```

```bash
# 开启 trace 日志
juicefs mount --trace redis://localhost/1 /mnt/jfs 2>/tmp/juicefs.log

# 实时查看日志
tail -f /tmp/juicefs.log

# 日志格式
# 2024/01/01 12:00:00.000000 juicefs[12345] <INFO>: mount success <mountpoint>
# 2024/01/01 12:00:01.000000 juicefs[12345] <DEBUG>: Read inode=100 offset=0 size=4096
```

## 三、问题定位流程

```
发现异常（挂载失败/性能差/数据不一致/进程崩溃）
  │
  ├─ 1. 收集基本信息
  │     juicefs status META-URL       # 卷状态
  │     juicefs version                # 版本号
  │     uname -a / 内存 / 磁盘         # 系统环境
  │
  ├─ 2. 确认问题类型
  │     ├─ 挂载失败 → 检查元数据引擎连接、对象存储访问、FUSE 内核模块
  │     ├─ 性能差   → juicefs stats + pprof CPU profile
  │     ├─ 内存高   → pprof heap profile
  │     ├─ 数据异常 → juicefs fsck / info / dump
  │     └─ 进程崩溃 → 检查 goroutine dump + 日志 + dmesg
  │
  ├─ 3. 缩小范围
  │     ├─ 对比环境：同一卷在其他客户端是否正常？
  │     ├─ 最小复现：能否用最小化步骤复现？
  │     └─ 回归测试：最近一次正常工作的版本是哪个？
  │
  ├─ 4. 深入分析（代码层面）
  │     ├─ 根据日志定位出错的代码路径
  │     ├─ 阅读相关模块的源码（参考本项目 skills）
  │     ├─ 添加临时 debug 日志重新编译
  │     └─ 使用 dlv (Go debugger) 附加到进程调试
  │
  └─ 5. 修复并验证
        ├─ 修改代码
        ├─ make juicefs && 本地验证
        ├─ 跑相关测试
        └─ 提交 PR
```

## 四、代码贡献流程

### 4.1 找到你的第一个 Issue

```bash
# 1. Fork JuiceFS 仓库
# 2. 浏览 Issue 列表
#    - Label: "good first issue"   → 新手友好
#    - Label: "help wanted"        → 需要帮助
#    - Label: "bug"                → 明确的 Bug
#    - Label: "enhancement"        → 功能增强
```

### 4.2 开发环境搭建

```bash
# 1. 安装 Go
# Go >= 1.25 (参考 go.mod)

# 2. Clone 仓库
git clone https://github.com/<your-fork>/juicefs.git
cd juicefs

# 3. 安装依赖
go mod download

# 4. 编译（完整版）
make juicefs

# 5. 编译（精简版，裁掉所有可选驱动）
make juicefs.lite

# 6. 运行测试
go test ./pkg/meta/ -v -count=1
go test ./pkg/chunk/ -v -count=1
go test ./pkg/vfs/ -v -count=1
```

### 4.3 代码结构规范

```
juicefs/
├── cmd/        # CLI 命令（每个命令一个文件）
├── pkg/        # 核心库（被 cmd/ 引用）
│   ├── meta/   # 元数据引擎
│   ├── object/ # 对象存储
│   ├── chunk/  # 数据缓存
│   ├── vfs/    # 虚拟文件系统
│   ├── fuse/   # FUSE 接口
│   ├── fs/     # Go 原生文件系统
│   ├── gateway/# S3 网关
│   └── sync/   # 数据同步
├── sdk/        # SDK
├── docs/       # 文档
├── integration/# 集成测试
└── hack/       # 构建辅助
```

### 4.4 Commit 规范

```bash
# Commit message 格式
type(scope): short description

Detailed description (optional)

# type:
# - fix: Bug 修复
# - feat: 新功能
# - perf: 性能优化
# - refactor: 重构
# - test: 测试相关
# - docs: 文档更新

# 示例:
git commit -m "fix(chunk): handle edge case when cache dir is full"
git commit -m "feat(meta): add support for TiKV pessimistic transactions"
```

### 4.5 PR 流程

```
1. 创建 feature 分支
   git checkout -b fix/cache-eviction-oom main

2. 开发 + 本地测试
   go test ./pkg/chunk/ -v -run TestEviction
   make juicefs && ./juicefs mount ...  # 手动验证

3. Commit & Push
   git add .
   git commit -m "fix(chunk): prevent OOM during cache eviction"
   git push origin fix/cache-eviction-oom

4. 在 GitHub 创建 PR
   - 填写 PR 描述（问题描述、修复方案、测试结果）
   - 关联 Issue: "Fixes #1234"
   - 等待 CI 通过 + Code Review

5. 根据 Review 修改
   git add .
   git commit -m "address review comments"
   git push origin fix/cache-eviction-oom
```

### 4.6 CI/CD

```bash
# JuiceFS 的 CI 会检查:
# - gofmt / goimports 代码格式
# - go vet 静态分析
# - golangci-lint 全面的 lint 检查
# - 单元测试 (pkg/*/...)
# - 集成测试 (integration/*.sh)
# - Build (跨平台编译)

# 本地可以先跑:
gofmt -w .
golangci-lint run ./...
go test ./...
```

## 五、常用 Go 调试技巧

### 5.1 添加临时调试日志

```go
import "github.com/juicedata/juicefs/pkg/utils"

func someFunction() {
    logger := utils.GetLogger("my-module")
    logger.Debugf("someFunction called with inode=%d", ino)
    logger.Infof("chunk %d uploaded to %s", chunkid, key)
    logger.Warnf("retry #%d after error: %s", retry, err)
}
```

### 5.2 使用 dlv 调试器

```bash
# 安装 dlv
go install github.com/go-delve/delve/cmd/dlv@latest

# 调试 mount 命令
dlv exec ./juicefs -- mount redis://localhost/1 /mnt/jfs

# 设置断点
(dlv) break pkg/vfs/vfs.go:Read
(dlv) continue

# 查看变量
(dlv) print ino
(dlv) print attr.Length

# 单步执行
(dlv) step
(dlv) next
```

### 5.3 性能分析常用命令

```bash
# 找出 CPU 热点
go tool pprof -top cpu.prof

# 火焰图
go tool pprof -http=:8080 cpu.prof

# 找出内存分配最多的函数
go tool pprof -top allocs.prof

# 对比两个 profile
go tool pprof -base before.prof after.prof
```

## 六、关键代码位置

| 文件/目录 | 内容 |
|-----------|------|
| `cmd/debug.go` | debug 命令实现 |
| `cmd/stats.go` | stats 命令实现 |
| `cmd/profile.go` | profile 命令实现 |
| `cmd/status.go` | status 命令实现 |
| `cmd/info.go` | info 命令实现 |
| `cmd/dump.go` | dump 命令实现 |
| `cmd/fsck.go` | fsck 命令实现 |
| `pkg/vfs/internal.go` | .control / .stats / .accesslog 等内部文件 |
| `pkg/metric/` | Prometheus 指标 |
| `Makefile` | 构建脚本 |
| `CONTRIBUTING.md` | 贡献指南 |
| `CODE_OF_CONDUCT.md` | 行为准则 |

## 七、参考资源

1. **JuiceFS Community Issue Tracker**: https://github.com/juicedata/juicefs/issues
2. **CONTRIBUTING.md**: https://github.com/juicedata/juicefs/blob/main/CONTRIBUTING.md
3. **Go pprof 文档**: https://go.dev/blog/pprof
4. **dlv 调试器**: https://github.com/go-delve/delve
5. **JuiceFS 社区论坛**: https://github.com/juicedata/juicefs/discussions
