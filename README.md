# F2FS Skills

基于 F2FS 主线源码分析总结的结构化知识集合。每个 SKILL.md 文件深入讲解 F2FS 文件系统的一个子系统或机制，包含核心设计、数据结构、完整代码流程、关键代码位置（精确到行号）以及常见问题。

> 本项目的组织方式参考了 [ceph-skills](https://github.com/i_ingfeng/ceph-skills) 的模式，以源码为基础，从问题起源到代码实现逐层剖析。

## 快速导航

### 🚀 入门与部署

| Skill | 内容 | 行数 |
|-------|------|------|
| [F2FS 快速部署](deploy/quick-start/SKILL.md) | 内核编译、格式化、挂载、调优参数、测试工具 | 520 |
| [F2FS 调试环境](deploy/debug-env/SKILL.md) | QEMU环境、trace工具、debugfs、dump分析 | 450 |

### 📐 核心架构

| Skill | 内容 | 行数 |
|-------|------|------|
| [F2FS 整体概述](overview/f2fs-intro/SKILL.md) | 设计理念、LFS原理、架构、数据流、演进历史 | 680 |
| [F2FS 磁盘布局](layout/on-disk-layout/SKILL.md) | Superblock、Checkpoint、SIT/NAT/SSA、六大区域详解 | 750 |

### 🗺️ 地址映射

| Skill | 内容 | 行数 |
|-------|------|------|
| [NAT 节点地址表](nat/node-address-table/SKILL.md) | Node映射、NAT缓存、NAT日志、地址转换流程 | 620 |
| [Node 管理](node/node-manager/SKILL.md) | inode/dnode/indirect node、node分配回收、radix-tree | 580 |

### 📝 日志与检查点

| Skill | 内容 | 行数 |
|-------|------|------|
| [Checkpoint 机制](checkpoint/checkpoint/SKILL.md) | CP pack结构、flush流程、版本切换、一致性保证 | 690 |
| [Recovery 流程](recovery/recovery/SKILL.md) | 3阶段恢复、roll-forward、fsync恢复、orphan处理 | 520 |

### 🔄 FTL 交互

| Skill | 内容 | 行数 |
|-------|------|------|
| [FTL 交互机制](ftl/ftl-interaction/SKILL.md) | FTL地址映射、F2FS与FTL关系、混合映射、协同优化 | 650 |

### 🧹 垃圾回收

| Skill | 内容 | 行数 |
|-------|------|------|
| [GC 机制](gc/garbage-collection/SKILL.md) | Victim选择、Greedy/Cost-benefit算法、前后台GC、SSR | 820 |
| [Segment 管理](segment/segment-management/SKILL.md) | SIT管理、空闲段分配、多段并发写入、冷热分离 | 640 |

### ⚡ 性能优化

| Skill | 内容 | 行数 |
|-------|------|------|
| [Multi-head 日志](logging/multi-head-logging/SKILL.md) | 6种日志类型、冷热数据分离、append策略、游标管理 | 530 |
| [Inline 优化](inline/inline-data/SKILL.md) | Inline data/dentry/xattr、小文件优化、转换流程 | 420 |
| [Extent Cache](cache/extent-cache/SKILL.md) | Extent树、读缓存、预取机制、缓存管理 | 480 |

### 🛠️ 高级特性

| Skill | 内容 | 行数 |
|-------|------|------|
| [Atomic Write](atomic/atomic-write/SKILL.md) | 原子写语义、in-place更新、journal机制、数据库优化 | 560 |
| [Compression](compression/compression/SKILL.md) | 压缩集群、LZ4/Zstd算法、透明压缩、cluster管理 | 490 |
| [Encryption](encryption/encryption/SKILL.md) | 文件/文件名加密、fscrypt框架、密钥管理、策略继承 | 510 |

## 推荐学习路线

```
入门 (1周)
  │
  ├─ 1. F2FS 整体概述          → 理解设计理念和架构
  ├─ 2. F2FS 快速部署          → 动手搭建实验环境
  ├─ 3. F2FS 磁盘布局          → 理解存储组织方式
  └─ 4. FTL 交互机制           → 理解与闪存设备的关系
       │
       ▼
核心机制 (2-3周)
  │
  ├─ 5. NAT 节点地址表         → 理解地址映射
  ├─ 6. Node 管理              → 理解元数据组织
  ├─ 7. Checkpoint 机制        → 理解一致性保证
  └─ 8. Segment 管理           → 理解空间分配
       │
       ▼
性能优化 (2-3周)
  │
  ├─ 9. GC 机制                → 理解垃圾回收
  ├─ 10. Multi-head 日志       → 理解并发写入
  ├─ 11. Extent Cache          → 理解读优化
  └─ 12. Inline 优化           → 理解小文件优化
       │
       ▼
高级特性 (2-3周)
  │
  ├─ 13. Recovery 流程         → 理解崩溃恢复
  ├─ 14. Atomic Write          → 理解原子操作
  ├─ 15. Compression           → 理解压缩机制
  └─ 16. Encryption            → 理解加密特性
```

## 每个 SKILL.md 的结构

每个技能文件遵循统一的组织方式：

```
〇、为什么需要这个机制？     — 问题起源，传统方案的不足
一、核心数据结构             — 关键结构体定义（带源码行号）
二、核心流程                 — 完整的调用链和状态转换
三、关键设计特性             — 设计哲学和权衡
四、关键代码位置             — 精确到文件:行号
五、常见问题与陷阱           — 实战中遇到的问题和解答
```

## 源码参考

本项目的分析基于以下代码库：

- **Linux Kernel F2FS**: `https://github.com/torvalds/linux` (fs/f2fs/)
- **F2FS Tools**: `https://github.com/jaegeuk/f2fs-tools`
- **主线代码**: 基于 Linux 6.x 内核版本

## 项目结构

```
f2fs-skills/
├── overview/                  # F2FS 整体架构
│   └── f2fs-intro/           # F2FS 介绍与设计理念
├── layout/                    # 磁盘布局
│   └── on-disk-layout/       # 磁盘数据结构详解
├── nat/                       # 节点地址表
│   └── node-address-table/   # NAT 管理机制
├── node/                      # 节点管理
│   └── node-manager/         # Node 分配与管理
├── checkpoint/                # 检查点
│   └── checkpoint/           # Checkpoint 机制详解
├── recovery/                  # 恢复机制
│   └── recovery/             # Recovery 流程分析
├── ftl/                       # FTL 交互
│   └── ftl-interaction/      # F2FS 与 FTL 的关系
├── gc/                        # 垃圾回收
│   └── garbage-collection/   # GC 算法与实现
├── segment/                   # 段管理
│   └── segment-management/   # Segment 分配策略
├── logging/                   # 日志系统
│   └── multi-head-logging/   # 多头日志机制
├── inline/                    # 内联优化
│   └── inline-data/          # Inline 数据存储
├── cache/                     # 缓存机制
│   └── extent-cache/         # Extent 缓存优化
├── atomic/                    # 原子写
│   └── atomic-write/         # 原子写实现
├── compression/               # 压缩
│   └── compression/          # 透明压缩机制
├── encryption/                # 加密
│   └── encryption/           # 文件加密实现
└── deploy/                    # 部署指南
    ├── quick-start/          # 快速开始指南
    └── debug-env/            # 调试环境搭建
```

## 贡献

欢迎提交 Issue 或 Pull Request 来补充和修正内容。每个 SKILL.md 中的代码行号基于特定版本的 Linux 内核源码，如果源码更新导致行号变化，也欢迎提交修正。

## License

本项目内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 许可。