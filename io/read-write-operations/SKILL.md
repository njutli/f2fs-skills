# F2FS 读写操作原理

## 问题起源

F2FS 作为专为闪存设备设计的日志结构文件系统，其读写操作机制与传统文件系统（如 ext4）有显著不同。理解 F2FS 如何处理读写请求，对于优化存储性能、理解垃圾回收机制以及排查性能问题至关重要。

核心问题：
- F2FS 如何将文件写入请求转化为物理块写入？
- 日志结构设计如何影响写入模式？
- 节点（Node）和数据（Data）如何协同工作？
- 多头日志（Multi-head Logging）如何提升性能？
- 内联（Inline）机制如何优化小文件性能？

## 核心数据结构

### 1. 节点类型（Node Types）

F2FS 中所有元数据都通过 Node 表示，分为三类：

```
struct f2fs_node {
    union {
        struct f2fs_inode i;      // Inode 节点
        struct direct_node dn;     // 直接节点
        struct indirect_node in;   // 间接节点
    };
    struct node_footer footer;    // 节点尾部信息
};
```

**Inode 节点：**
- 包含文件/目录的元数据（权限、大小、时间戳等）
- 内嵌数据块指针（最多 923 个直接指针）
- 内嵌目录项（对于小目录）

**直接节点（Direct Node）：**
- 包含 1018 个数据块地址
- 直接指向数据块

**间接节点（Indirect Node）：**
- 包含 1018 个节点地址
- 指向其他节点

### 2. 文件数据组织

F2FS 使用多级节点树管理文件数据：

```
Level 0: Inode 内嵌指针（inline_data 或最多 923 个直接指针）
Level 1: 直接节点（每个节点 1018 个数据块指针，约 4MB）
Level 2: 一级间接节点（每个间接节点指向 1018 个直接节点，约 4GB）
Level 3: 二级间接节点（每个间接节点指向 1018 个一级间接节点，约 4TB）
Level 4: 三级间接节点（理论上可达 4PB）
```

### 3. 关键结构：Node Footer

每个节点都有一个 footer，记录关键信息：

```
struct node_footer {
    __le32 nid;          // 节点 ID
    __le32 ino;          // 所属 inode 号
    __le32 flag;         // 节点类型标志
    __le64 cp_ver;       // 检查点版本
    __le32 next_blkaddr; // 下一个块的逻辑地址（用于 WAL）
};
```

### 4. 多头日志（Multi-head Logging）

F2FS 将写入分为 6 种类型的日志流：

```
数据类型：
- COLD data: 冷数据（从不访问或不重要）
- WARM data: 温数据（正常温度）
- HOT data: 热数据（频繁访问，如元数据）

节点类型：
- COLD dnode: 冷节点（目录节点）
- WARM dnode: 温节点（普通文件节点）
- HOT dnode: 热节点（间接节点）
```

**测试结果验证：**
```
Main area: 502 segs, 502 secs 502 zones
    TYPE           blkoff    segno    secno   zoneno  dirty_seg   full_seg  valid_blk
  - COLD   data:        0      124      124      124          0          0          0
  - WARM   data:        0       53       53       53          0         50      25600
  - HOT    data:        5        3        3        3          1          0          1
  - Dir   dnode:       11        0        0        0          1          0          2
  - File  dnode:       64        1        1        1          1          0         48
  - Indir nodes:        1        2        2        2          1          0          1
```

可以看到：
- 大部分数据写入 WARM data 类型（25600 个有效块，50 个完整段）
- 文件元数据主要在 File dnode（48 个有效块）
- 目录元数据在 Dir dnode（2 个有效块）

## 核心流程

### 1. 文件写入流程

#### 传统文件系统（ext4）的写入：

```
应用 write()
  ↓
文件系统层：查找空闲块
  ↓
块层：映射逻辑块到物理块
  ↓
设备层：直接写入固定位置（覆盖写）
```

#### F2FS 的写入流程：

```
应用 write()
  ↓
F2FS 层：分配新块（从 Curseg）
  ↓
日志写入：追加到当前日志尾部
  ↓
更新 NAT：记录 <vblock, pblock> 映射
  ↓
更新 Node：修改 inode 或 direct node
  ↓
Checkpoint（定期）：持久化元数据
```

**实际测试观察：**

写入 100MB 文件：
```bash
dd if=/dev/urandom of=large_file.bin bs=1M count=100
sync
```

状态变化：
```
- Data blocks: 25601
- Node blocks: 51 (Inode: 25, Other: 26)
CP calls: 4
```

关键观察：数据块连续分配在 WARM data 段中。

### 2. 文件追加写入流程

F2FS 对追加写入有特殊优化：

```
文件大小 < inline_size (通常 4KB)
  ↓
数据存储在 inode 内部
  ↓
不需要分配额外数据块
```

**测试验证：**
```bash
echo "First line" > append_test.txt
for i in {1..100}; do echo "Line $i" >> append_test.txt; done
sync
```

状态：
```
- Inline_data Inode: 22
- 文件大小: 803 字节（可能已转为常规数据存储）
```

### 3. 目录创建流程

创建目录时，F2FS 使用 inline_dentry 优化：

```
mkdir subdirectory
  ↓
分配新 inode（Dir dnode）
  ↓
如果目录项数量 < threshold
  ↓
目录项存储在 inode 内部（inline_dentry）
```

**测试验证：**
```
mkdir subdirectory
for i in {1..20}; do touch subdirectory/file_$i.txt; done
```

状态：
```
- Inline_dentry Inode: 1
- Dir dnode: 2 个有效块
```

### 4. 小文件写入（Inline 机制）

F2FS 对小文件有两种内联优化：

**inline_data：**
```
文件大小< inline_size（默认 4KB）
  ↓
数据存储在 inode 结构中
  ↓
优点：
  - 零额外数据块分配
  - 减少元数据开销
  - 加速元数据读写
```

**inline_dentry：**
```
目录文件数量 < max_inline_dentry（默认 8-64）
  ↓
目录项存储在 inode 结构中
  ↓
优点：
  - 小目录无需额外数据块
  - 加速目录查找
```

**测试统计：**
```
- Inline_xattr Inode: 24
- Inline_data Inode: 22
- Inline_dentry Inode: 1
```

说明约 88% 的文件使用内联数据优化。

### 5. 大文件写入流程

对于大文件（> inline_size），F2FS 分配独立数据块：

```
大文件写入请求
  ↓
检查文件当前大小
  ↓
分配新数据块（从 Curseg[WARM data]）
  ↓
写入数据到新块
  ↓
更新 NAT：<vblock, pblock>
  ↓
更新 Node：记录新块地址
  ↓
如果 Node 需要新增：分配 Node 块
```

### 6. 读取流程

F2FS 读取需要通过 NAT 转换地址：

```
应用 read()
  ↓
文件系统层：计算逻辑块号
  ↓
查找 Node：获取 <vblock> 地址
  ↓
查询 NAT：<vblock> → <pblock>
  ↓
读取物理块 <pblock>
```

**关键点：**
- 读取需要额外的 NAT 查找
- F2FS 维护范围缓存（Extent Cache）加速查找
- inline_data/inline_dentry 可以避免块读取

**范围缓存测试：**
```
Extent Cache (Read):
  - Hit Count: L1-1:0 L1-2:0 L2:0
  - Hit Ratio: 0% (0 / 25600)
  - Inner Struct Count: tree: 23(0), node: 2
```

此时缓存命中率较低，因为这是刚写入的数据。

## 关键设计特性

### 1. WAL（Write-Ahead Logging）机制

F2FS 所有写入都是追加（Append-only）：

```
传统文件系统：覆盖写
  Block 100: [数据] → Block 100: [新数据] (原地覆盖)

F2FS：追加写
  Block 100: [数据] → Block 200: [新数据] (新位置)
  更新映射：vblock=100 → pblock=200
```

**优点：**
- 避免随机写入
- 充分利用闪存顺序写性能
- 简化错误恢复（通过 Checkpoint）

**代价：**
- 需要 NAT 维护地址映射
- 需要垃圾回收（Garbage Collection）

### 2. Curseg（Current Segment）机制

F2FS 为每种日志类型维护一个当前段：

```
发生。Curseg[WARM data] = Segment 53
写入数据 → Block 53:0, 53:1, 53:2, ...
Segment满后→分配新 Segment
```

**测试观察：**
```
WARM   data: segno=53, full_seg=50
```

说明当前 WARM data 段从 Segment 53 开始，已填满 50 个段。

### 3. Checkpoint 机制

Checkpoint 持久化文件系统状态：

```
触发条件：
1. sync()/fsync() 系统调用
2. 定期后台检查点（默认 5 秒）
3. 内存压力
4. 文件系统卸载

写入内容：
- CP 块：元数据摘要
- SIT 块：段信息表（有效块位图）
- NAT 块：节点地址表
- SSA 块：段摘要区（可选）

写入流程：
1. 冻结所有更新
2. 刷出所有脏节点和数据
3. 写入 CP、SIT、NAT
4. 更新 Superblock
```

**测试统计：**
```
CP calls: 4 (BG: 0)
CP count: 4
  - cp blocks : 12
  - sit blocks : 1
  - nat blocks : 1
  - ssa blocks : 50

CP merge:
  - Issued : 4
  - Cur time : 5(ms)
  - Peak time : 146(ms)
```

说明：
- 共执行 4 次 checkpoint
- 全部是前台 checkpoint（BG: 0）
- SSA 块数较多（50），提高恢复效率

### 4. 延迟分配（Delayed Allocation）

F2FS 支持类似 ext4 的延迟分配：

```
写入请求 → 缓存到 Page Cache
  ↓
sync/磁盘空间紧张 → 分配块并写入
```

**优点：**
- 减少碎片
- 提高写入聚合度

### 5. 范围缓存（Extent Cache）

F2FS 维护文件物理地址范围缓存：

```
文件块 < 100-110> → 物理块 < 1000-1010>
文件块 < 200-210> → 物理块 < 2000-2010>
```

**缓存层次：**
- L1-1: 最近访问的范围
- L1-2: 热点范围
- L2: 全局范围缓存

**测试数据：**
```
Inner Struct Count: tree: 23(0), node: 2
```

说明已建立 23 个范围缓存树，2 个缓存节点。

### 6. 数据放置策略

F2FS 根据数据温度分配不同的段：

```
Hot Data: 日志文件、临时文件
  ↓
Warm Data: 普通文件
  ↓
Cold Data: 末尾块、稀疏文件

Hot Node: 间接节点（频繁被间接节点访问）
  ↓
Warm Node: 文件节点（File dnode）
  ↓
Cold Node: 目录节点（Dir dnode）
```

**策略原因：**
- 冷热分离：减少 GC 迁移成本
- 热数据聚集：提高缓存命中率
- 冷数据隔离：避免影响热数据性能

### 7. Fsync 优化

F2FS 提供多种 fsync 模式：

```
fsync_mode=posix:
  - fsync() 先写出所有脏节点
  - 再写出数据
  - 最后写出 CP
  - 最安全，性能最低

fsync_mode=strict:
  - 类似 posix
  - 但允许少量数据延迟

fsync_mode=no-barrier:
  - 关闭写屏障
  - 性能最高，崩溃恢复可能丢失数据
  - 仅适用于有电池保护的设备
```

**测试配置：**
```
fsync_mode=posix
```

### 8. SIT（Segment Information Table）机制

SIT 记录每个段的有效块信息：

```
每个 SIT entry (4字节) 包含：
- valid_map: 位图（512 bits，每个 bit 代表一个块）
- vblocks: 有效块计数

作用：
1. 垃圾回收选择 victim 段
2. 空间管理
3. 碎片整理
```

### 9. NAT（Node Address Table）机制

NAT 记录节点地址映射：

```
<vnode_id (nid), pblock_physical_address>

作用：
1. 支持 WAL 机制
2. 节点可以移动（GC）
3. 地址重定向
```

## 关键代码位置

### 写入路径

```
文件写入：
mm/filemap.c: generic_file_buffered_write()
 ↓
fs/f2fs/file.c: f2fs_write_begin() / f2fs_write_end()
  ↓
fs/f2fs/data.c: f2fs_write_data_pages()
  ↓
fs/f2fs/segment.c: allocate_data_block()
  ↓
fs/f2fs/segment.c: write_data_page()

节点更新：
fs/f2fs/node.c: f2fs_write_node_pages()
  ↓
fs/f2fs/node.c: sync_node_pages()
```

### 追加写入

```
fs/f2fs/file.c: f2fs_file_write_iter()
  ↓
fs/f2fs/data.c: f2fs_write_end_io()
  ↓
fs/f2fs/segment.h: CURSEG_WARM_DATA
```

### Inline 数据

```
fs/f2fs/inline.c:
  - f2fs_write_inline_data()
  - f2fs_read_inline_data()
  - f2fs_convert_inline_page()

fs/f2fs/namei.c:
  - f2fs_create() → f2fs_init_inode_metadata()
```

### 目录操作

```
fs/f2fs/namei.c: f2fs_mkdir()
  ↓
fs/f2fs/dir.c: f2fs_add_link()
  ↓
fs/f2fs/inline.c: f2fs_add_inline_entry()
```

### Checkpoint

```
fs/f2fs/checkpoint.c:
  - block_operations()
  - do_checkpoint()
  - f2fs_write_checkpoint()

fs/f2fs/super.c: f2fs_sync_fs()
  ↓
fs/f2fs/checkpoint.c: f2fs_sync_fs()
```

### NAT 管理优先的作用

```
fs/f2fs/node.h: struct nat_entry
fs/f2fs/node.c:
  - get_node_addr()
  - set_node_addr()
  - f2fs_get_node_page()
```

### 垃圾回收（写入触发）

```
fs/f2fs/gc.c:
  - f2fs_gc()
  - gc_data_segment()
  - move_data_page()
```

## 常见问题

### 1. 为什么 F2FS 写入性能优于 ext4？

**答案：**
- F2FS 使用追加写，避免覆盖写的随机 I/O
- 多头日志将热数据、温数据、冷数据分离，减少 GC 开销
- Curseg 机制预分配段，减少寻址开销
- Inline 机制减少小文件的元数据开销

**测试验证：**
```
写入 100MB 文件：
dd if=/dev/urandom of=large_file.bin bs=1M count=100
→ 104857600 bytes copied in 0.158528 s (661 MB/s)
```

### 2. F2FS 读取性能如何？

**答案：**
- 读取需要额外的 NAT 查找（一次额外内存查找）
- 范围缓存（Extent Cache）加速连续块读取
- inline_data 文件可以零块读取（数据在 inode 内）

**优化建议：**
- 启用 extent_cache（默认已启用）
- 适当增加 active_logs（默认 6）
- 对大文件启用 read-ahead

### 3. 小文件性能为什么更好？

**答案：**
- inline_data 避免分配数据块
- inline_dentry 避免目录块分配
- 数据和元数据集中，减少 I/O 次数

**测试统计：**
```
Inodes IUsed: 25652
Inline_data Inode: 22 (88% 的文件)
```

### 4. 追加写如何保证数据一致性？

**答案：**
- WAL 机制：所有写入先记录
- Checkpoint 定期持久化状态
- fsync 保证立即 checkpoint
- 日志结构天然支持崩溃恢复

### 5. 文件删除会发生什么？

**答案：**
```
文件删除：
  ↓
标记 inode 为 orphan
  ↓
释放数据块（更新 SIT valid_map）
  ↓
释放节点块（更新 NAT）
  ↓
等待 Checkpoint 持久化
  ↓
（可选）后台 GC 回收空间
```

**测试观察：**
```
删除 100MB 文件后：
GC calls: 0 (gc_thread: 0)
→ 空间标记为无效，等待后台 GC 回收
```

### 6. 性能调优建议？

**挂载选项：**
```
# 平衡性能和安全
mount -o mode=adaptive,background_gc=on,discard

# 性能优先
mount -o mode=lfs,background_gc=on,no_heap,discard

# SSD 优化
mount -o ssd_atom_write=on

# 大文件优化
mount -o inline_data=off,extent_cache=on
```

**内核参数：**
```
/sys/fs/f2fs/<dev>/gc_urgent_sleep_time
/sys/fs/f2fs/<dev>/gc_idle_time
/sys/fs/f2fs/<dev>/cp_interval
```

### 7. 如何诊断写入性能问题？

**方法：**
```bash
# 查看 F2FS 状态
cat /sys/kernel/debug/f2fs/status

# 查看 CP 频率
grep "CP calls" /sys/kernel/debug/f2fs/status

# 查看 GC 活动
grep "GC calls" /sys/kernel/debug/f2fs/status

# 查看段使用情况
grep "valid_blk" /sys/kernel/debug/f2fs/status

# 查看写入带宽
f2fs_io get_cprecom  # F2FS 工具
```

### 8. F2FS 与 FTL 的交互？

**关键点：**
- F2FS 的段（Segment）与 FTL 的块（Block）对齐
- discard 挂载选项让 F2FS 主动释放块（TRIM 命令）
- FTL 的 GC 与 F2FS 的 GC 可能冲突
- 最佳配置：F2FS 段大小 = FTL 块大小

### 9. 为什么有多个日志头？

**原因：**
- 分离数据和元数据写入
- 按温度分离数据（冷、温、热）
- 减少空间回收成本
- 提高顺序写入性能

**观察：**
```
写入 100MB 数据：
- WARM data: 25600 blocks
- File dnode: 48 blocks
- HOT data: 1 block

→ 大部分数据块写入 WARM 区
→ 元数据块写入对应的 Node 区
```

### 10. Checkpoint 性能影响？

**影响：**
```
CP开销：
- 冻结文件系统 ：~1-10ms
- 写入脏数据：数据量相关
- 写入 CP/SIT/NAT：固定开销 ~5-10ms
- 更新 Superblock：~1-2ms

测试数据：
CP calls: 4
  - Cur time : 5(ms)
  - Peak time : 146(ms)
```

**优化：**
- 减少不必要的 fsync
- 增加 cp_interval
- 使用 checkpoint_merge 合并多个 CP 请求

## 实际测试结果总结

### 测试环境
```
文件系统: F2FS (mkfs.f2fs Ver: 1.16.0)
镜像大小: 1GB (1024 MB)
段数量: 502 segments
预留空间: 5.370% (27 segments for GC)
```

### 写入测试

**小文件：**
```
echo "F2FS test content" > test_file.txt
文件大小: 18 字节
状态: Inline_data Inode
```

**大文件：**
```
dd if=/dev/urandom of=large_file.bin bs=1M count=100
文件大小: 100 MB
数据块: 25600 (WARM data)
节点块: 48 (File dnode)
写入速度: 661 MB/s
```

**追加写入：**
```
echo "First line" > append_test.txt
for i in {1..100}; do echo "Line $i" >> append_test.txt; done
文件大小: 803 字节
CP calls: 4 (每次 sync 触发 CP)
```

**目录创建：**
```
mkdir subdirectory
for i in {1..20}; do echo "File $i data" > subdirectory/file_$i.txt; done
目录项: 20 个文件
状态: Inline_dentry Inode: 1
```

### 状态分析

**空间使用：**
```
Utilization: 10%
有效块: 25652
- Node: 51 (Inode: 25, Other: 26)
- Data: 25601
```

**块分布：**
```
WARM data: 50 full_seg (25600 valid_blk)
HOT data: 1 dirty_seg (1 valid_blk)
File dnode: 48 valid_blk
Dir dnode: 2 valid_blk
```

**元数据统计：**
```
Inline_xattr Inode: 24
Inline_data Inode: 22
Inline_dentry Inode: 1
```

### 关键观察

1. **写入模式：**
   - 大文件写入 WARM data 段，连续分配
   - 元数据写入对应的 Node 段
   - 小文件使用 inline_data，零数据块分配

2. **空间效率：**
   - 10% 的空间使用率
   - 50 个完整段用于 WARM data（约 200MB）
   - 预留 27 个段用于 GC 和 Overprovision

3. **CP 活动：**
   - 测试期间 4 次 CP（sync 触发）
   - 无后台 CP（BG: 0）
   - CP 延迟较低（5ms 平均）

4. **GC 活动：**
   - 测试期间未触发 GC（GC calls: 0）
   - 空间充足，无需立即回收
   - 后台 GC 已启用（background_gc=on）

5. **性能特征：**
   - 大文件写入：661 MB/s（虚拟环境）
   - 元数据操作：快速（inline 优化）
   - CP 延迟：低（5ms平均，146ms 峰值）

## 参考资源

- F2FS 源码：`fs/f2fs/` (Linux 内核)
- F2FS 文档：`Documentation/filesystems/f2fs.rst`
- F2FS 工具：`f2fs-tools` 包
- 测试博客：https://www.cnblogs.com/Linux-tech/p/12961293.html