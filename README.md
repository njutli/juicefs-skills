# JuiceFS Skills

基于 [JuiceFS](https://github.com/juicedata/juicefs) 源码的结构化学习工程，以代码为核心，逐层剖析 JuiceFS 的整体架构和具体实现，目标是能够定位问题并为官方做出贡献。

> 本项目的组织方式参考了 [linux-kernel-skills](https://github.com/linux-kernel-skills) 和 [ceph-skills](https://github.com/ceph-skills) 的模式，以源码为基础，从问题起源到代码实现逐层剖析。与 ceph-skills 侧重"概念"不同，本项目侧重"代码阅读与调试能力"。

## 快速导航

### 🎓 Go 语言基础

| Skill | 内容 |
|-------|------|
| [Go 语言实战速成](golang-basics/SKILL.md) | 变量/类型/控制流 → 接口/嵌入/组合 → goroutine/channel/context → 错误处理 → 包管理 → build tags → 阅读 JuiceFS 代码所需的全部 Go 知识 |

### 🏛️ 架构概览

| Skill | 内容 |
|-------|------|
| [JuiceFS 整体架构](overview/SKILL.md) | 三层架构、组件全景图、数据流与元数据流、设计哲学与权衡 |

### 🔌 CLI 与入口

| Skill | 内容 |
|-------|------|
| [命令行框架](cli/command-system/SKILL.md) | urfave/cli 框架、子命令注册与路由、全局选项处理、系统挂载参数解析 |
| [Mount 全流程](cli/mount-flow/SKILL.md) | format→mount 完整链路、FUSE 挂载流程、Go-FUSE 集成、守护进程模式 |

### 🗄️ 元数据引擎

| Skill | 内容 |
|-------|------|
| [Meta 接口定义](metadata/meta-interface/SKILL.md) | Ino/Attr/Slice 核心类型、Meta 接口（100+方法）、Context/Config/Format、Session 管理 |
| [Redis 引擎实现](metadata/redis-engine/SKILL.md) | Lua 事务脚本、Sentinel/Cluster 模式、Session 心跳、Client Side Caching |
| [SQL 引擎实现](metadata/sql-engine/SKILL.md) | XORM 框架、三表 Schema、MySQL/PostgreSQL/SQLite 差异、事务与隔离级别 |

### 📁 数据路径

| Skill | 内容 |
|-------|------|
| [VFS 虚拟文件系统层](data-path/vfs-layer/SKILL.md) | VFS 作为核心桥梁、Lookup/Open/Read/Write 实现、Handle 管理、内部文件系统 |
| [Chunk 存储与缓存](data-path/chunk-cache/SKILL.md) | ChunkStore 接口、Disk Cache/Memory Cache 双层缓存、Page 64KB 对齐、Writeback/Prefetch |
| [Slice 压缩与碎片整理](data-path/slice-compaction/SKILL.md) | BST Slice 树、碎片化问题、Compact 策略、文件结束位置处理 |

### 🔗 FUSE 与接入层

| Skill | 内容 |
|-------|------|
| [FUSE 接口实现](fuse/SKILL.md) | go-fuse/v2 RawFileSystem、内核协议交互、设备/上下文处理、GID 缓存 |
| [S3 Gateway](gateway/SKILL.md) | MinIO Gateway 模式、S3-Compat API 映射、多用户权限隔离 |
| [Sync 同步引擎](sync-engine/SKILL.md) | 多线程对象同步、Checkpoint 断点续传、Cluster 分布式同步、Include/Exclude 过滤 |

### 📦 对象存储层

| Skill | 内容 |
|-------|------|
| [ObjectStorage 层](object-storage/SKILL.md) | ObjectStorage 接口、S3 后端实现、工厂模式 CreateStorage、Encrypt/Sharding 装饰器、30+ 后端适配 |

### 🔧 调试与贡献

| Skill | 内容 |
|-------|------|
| [调试工具与代码贡献](debug/SKILL.md) | pprof/profile/stats/debug/dump 工具链、日志体系、问题定位流程、Issue/PR 贡献指南、提交规范 |

## 推荐学习路线

路线将 16 篇专题与 JuiceFS 源码阅读穿插结合，按四阶段推进，共约 6 周。

---

### 📅 第一阶段：Go 语言 + 架构全景（第 1 周，3 篇 Skill）

**目标**：掌握阅读 JuiceFS 源码所需的 Go 语言知识，理解 JuiceFS 的整体架构和数据流。

| 天数 | Skill | 核心收获 |
|------|-------|---------|
| 第 1-3 天 | [Go 语言实战速成](golang-basics/SKILL.md) | 在 IDE 中能流畅阅读 Go 代码：理解 interface 多态、goroutine 并发、error 处理链、struct tag、build tags |
| 第 4-5 天 | [JuiceFS 整体架构](overview/SKILL.md) | 画出三层架构图，说出一次 Read 请求经过的所有组件 |
| 第 6-7 天 | [命令行框架](cli/command-system/SKILL.md) | 理解 main.go → cmd.Main() → urfave/cli 的调用链，能注册一个新的调试子命令 |

**里程碑**：在 JuiceFS 源码目录下，能用 `go build` 编译，能说出任意一个 `*.go` 文件中 interface/struct/goroutine 的用途。

---

### 📅 第二阶段：核心数据流（第 2-3 周，5 篇 Skill）

**目标**：追踪从 FUSE 到对象存储的完整读写链路，理解元数据引擎的核心抽象。

| 天数 | Skill | 核心收获 |
|------|-------|---------|
| 第 8-9 天 | [Meta 接口定义](metadata/meta-interface/SKILL.md) | 理解 Ino/Attr/Slice 类型，Meta 接口的 100+ 方法分类，Config/Format 配置体系 |
| 第 10-12 天 | [VFS 虚拟文件系统层](data-path/vfs-layer/SKILL.md) | 追踪 Read 从 VFS → meta.Lookup → chunk.Read 的完整调用链，理解 Handle 生命周期 |
| 第 13-14 天 | [Chunk 存储与缓存](data-path/chunk-cache/SKILL.md) | ChunkStore 如何管理双层缓存，Page 对齐机制，Writeback 回写策略 |
| 第 15-16 天 | [Slice 压缩与碎片整理](data-path/slice-compaction/SKILL.md) | BST 树如何索引文件数据，Compact 触发时机与流程 |
| 第 17-18 天 | [FUSE 接口实现](fuse/SKILL.md) | go-fuse/v2 RawFileSystem 与 POSIX 的映射关系，内核态→用户态数据传递 |

**里程碑**：能画出一条 4KB 写入从 `write()` syscall → FUSE → VFS → Cache → ObjectStorage 的完整调用图，附每层涉及的函数名和文件:行号。

---

### 📅 第三阶段：元数据与对象存储引擎（第 4-5 周，5 篇 Skill）

**目标**：深入理解三种元数据引擎的实现差异，掌握对象存储的适配模式。

| 天数 | Skill | 核心收获 |
|------|-------|---------|
| 第 19-22 天 | [Redis 引擎实现](metadata/redis-engine/SKILL.md) | Lua 事务脚本的设计模式、Sentinel 自动故障转移、Client Side Caching 加速 |
| 第 23-25 天 | [SQL 引擎实现](metadata/sql-engine/SKILL.md) | XORM 三层表设计（jfs_node/jfs_edge/jfs_symlink），MySQL/PG/SQLite 的事务差异处理 |
| 第 26-27 天 | [ObjectStorage 层](object-storage/SKILL.md) | ObjectStorage 接口的核心方法，S3 后端如何实现 MultipartUpload，Encrypt 装饰器模式 |
| 第 28-29 天 | [S3 Gateway](gateway/SKILL.md) | MinIO 如何将 S3 API 映射到 `pkg/fs/` 的文件操作，多用户隔离 |
| 第 30-31 天 | [Sync 同步引擎](sync-engine/SKILL.md) | 多 worker 并发同步、checkpoint 断点续传、inode 映射 |

**里程碑**：能对比 Redis/SQL/TiKV 三种引擎在 Lookup 实现上的代码差异，能说出每种引擎的适用场景。

---

### 📅 第四阶段：全链路调试与贡献（第 6 周，2 篇 Skill）

**目标**：掌握调试工具链，具备定位 Bug 和提交 PR 的能力。

| 天数 | Skill | 核心收获 |
|------|-------|---------|
| 第 32-35 天 | [Mount 全流程](cli/mount-flow/SKILL.md) | format 写入了什么、mount 初始化了哪些对象、session 注册流程、第一个 Read 请求的完整路径 |
| 第 36-40 天 | [调试工具与代码贡献](debug/SKILL.md) | 用 pprof/profile/stats/dump 诊断问题，看懂 debug 输出，按 CONTRIBUTING.md 提交 PR |

**里程碑**：能独立完成一个简单的 Bug fix：在 Issue 列表中找到你的第一个 Good First Issue，定位代码、修改、测试、提 PR。

---

## 每个 SKILL.md 的结构

每个技能文件遵循统一的组织方式：

```
〇、为什么需要这个模块？     — 问题起源，该模块解决的痛点
一、核心数据结构             — 关键接口/结构体定义（带源码行号）
二、核心流程                 — 完整调用链和关键函数签名
三、关键设计特性             — 设计哲学、边界条件处理、性能优化
四、关键代码位置             — 精确到文件:行号
五、调试与排查               — 如何用 debug/dump/stats 验证此模块
六、常见问题                 — 实战中的困惑与解答
七、改进思路 / 贡献方向      — 可能的优化点、开放的 Issue 方向
```

## 源码参考

本项目的分析基于以下代码库：

- **JuiceFS**: `https://github.com/juicedata/juicefs` (本机路径: `/home/lilingfeng/project/juicefs`)

## 项目结构

```
juicefs-skills/
├── README.md                      # 索引与学习路线
├── golang-basics/                  # Go 语言基础
│   └── SKILL.md
├── overview/                       # 架构概览
│   └── SKILL.md
├── cli/                            # CLI 入口
│   ├── command-system/
│   │   └── SKILL.md                # urfave/cli 命令框架
│   └── mount-flow/
│       └── SKILL.md                # Mount 完整流程
├── metadata/                       # 元数据引擎
│   ├── meta-interface/
│   │   └── SKILL.md                # Meta 接口与核心类型
│   ├── redis-engine/
│   │   └── SKILL.md                # Redis 引擎实现
│   └── sql-engine/
│       └── SKILL.md                # SQL 引擎实现
├── data-path/                      # 数据路径
│   ├── vfs-layer/
│   │   └── SKILL.md                # VFS 虚拟文件系统层
│   ├── chunk-cache/
│   │   └── SKILL.md                # Chunk 存储与缓存
│   └── slice-compaction/
│       └── SKILL.md                # Slice 压缩与碎片整理
├── fuse/                           # FUSE 接入层
│   └── SKILL.md
├── object-storage/                 # 对象存储层
│   └── SKILL.md
├── gateway/                        # S3 Gateway
│   └── SKILL.md
├── sync-engine/                    # 数据同步
│   └── SKILL.md
└── debug/                          # 调试与贡献
    └── SKILL.md
```

## License

本项目内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可。
