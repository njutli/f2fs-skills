---
name: f2fs-on-disk-layout
description: F2FS磁盘布局详解。当用户询问F2FS的磁盘数据结构、Superblock格式、Checkpoint结构、SIT/NAT/SSA区域、Main Area组织、Segment/Section/Zone概念时调用此技能。
---

# F2FS 磁盘布局详解

## 〇、为什么需要了解磁盘布局？

理解 F2FS 的磁盘布局是深入掌握其工作原理的基础：
- **故障诊断**：通过磁盘结构分析文件系统损坏原因
- **性能优化**：了解数据组织方式，优化访问模式
- **数据恢复**：手工恢复关键数据
- **二次开发**：基于 F2FS 开发新特性

## 一、整体布局概览

F2FS 将整个分区划分为以下区域：

```
┌─────────────────────────────── F2FS Partition ───────────────────────────────┐
│                                                                               │
│  ┌────────┐  ┌──────────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────────────────┐  │
│  │   SB   │  │    CP    │  │ SIT │  │ NAT │  │ SSA │  │   Main Area     │  │
│  │ (1 blk)│  │ (2 segs) │  │     │  │     │  │     │  │                 │  │
│  └────────┘  └──────────┘  └─────┘  └─────┘  └─────┘  └─────────────────┘  │
│   0x0000      0x0400        Variable size areas        Start from zone 0     │
│                                                                               │
│  Superblock  Checkpoint    Segment   Node     Summary   User Data & Node     │
│              Area          Info      Address  Area      Storage Area          │
│                            Table     Table                                    │
└───────────────────────────────────────────────────────────────────────────────┘
```

关键概念：
- **Block**: 基本 I/O 单位，默认 4KB
- **Segment**: 2MB (512 blocks)，GC 的基本单位
- **Section**: 一个或多个连续的 segments
- **Zone**: 一个或多个连续的 sections，对应闪存擦除块

## 二、Superblock (SB)

### 2.1 位置与作用

Superblock 位于分区起始位置（block 0），包含文件系统的全局信息。

### 2.2 数据结构

```c
// fs/f2fs/f2fs.h:1419
struct f2fs_super_block {
    __le32 magic;                    /* 魔数 0xF2F52010 */
    __le16 major_ver;                /* 主版本号 */
    __le16 minor_ver;                /* 次版本号 */
    __le32 log_sectorsize;           /* log2(sector_size) */
    __le32 log_sectors_per_block;    /* log2(sectors_per_block) */
    __le32 log_blocksize;            /* log2(block_size) */
    __le32 log_blocks_per_seg;       /* log2(blocks_per_segment) */
    __le32 segs_per_sec;             /* segments per section */
    __le32 secs_per_zone;            /* sections per zone */
    __le32 checksum_offset;          /* checksum offset inside super block */
    __le64 block_count;              /* total block count */
    __le32 section_count;            /* total section count */
    __le32 segment_count;            /* total segment count */
    __le32 segment_count_ckpt;       /* # of segments for checkpoint */
    __le32 segment_count_sit;        /* # of segments for SIT */
    __le32 segment_count_nat;        /* # of segments for NAT */
    __le32 segment_count_ssa;        /* # of segments for SSA */
    __le32 segment_count_main;       /* # of segments for main area */
    __le32 segment0_blkaddr;         /* start block address of segment 0 */
    __le32 cp_blkaddr;               /* start block address of checkpoint */
    __le32 sit_blkaddr;              /* start block address of SIT */
    __le32 nat_blkaddr;              /* start block address of NAT */
    __le32 ssa_blkaddr;              /* start block address of SSA */
    __le32 main_blkaddr;             /* start block address of main area */
    __le32 root_ino;                 /* root inode number */
    __le32 node_ino;                 /* node inode number */
    __le32 meta_ino;                 /* meta inode number */
    __u8 uuid[16];                   /* 128-bit UUID */
    __le16 volume_name[MAX_VOLUME_NAME]; /* volume name */
    __le32 extension_count;          /* # of extensions */
    __u8 extension_list[F2FS_MAX_EXTENSION][8]; /* extension array */
    __le32 cp_payload;               /* checkpoint payload size */
    __u8 version[VERSION_LEN];       /* kernel version */
    __u8 init_version[VERSION_LEN];  /* initial kernel version */
    __le32 feature;                  /* defined features */
    __u8 encryption_level;           /* encryption level */
    __u8 encrypt_pw_salt[16];        /* encryption password salt */
    struct f2fs_device devs[MAX_DEVICES]; /* multi-device info */
    __le16 qf_ino[F2FS_MAX_QUOTAS]; /* quota inode numbers */
    __u8 hot_ext_count;              /* # of hot file extensions */
    __le16 s_encoding;               /* filename encoding */
    __le16 s_encoding_flags;         /* filename encoding flags */
    __u8 compress_algorithm;         /* compression algorithm */
    __u8 compress_log_size;          /* compression cluster log size */
    __u8 reserved[3006];             /* valid reserved region */
    __le32 crc;                      /* checksum of superblock */
} __packed;
```

### 2.3 关键字段解析

```
Magic Number: 0xF2F52010
  用于识别 F2FS 文件系统

布局计算相关：
  log_blocksize = 12        → block_size = 4KB
  log_blocks_per_seg = 9    → blocks_per_segment = 512
  segment_size = 512 * 4KB = 2MB

区域地址：
  cp_blkaddr:   Checkpoint 起始地址
  sit_blkaddr:  SIT 起始地址  
  nat_blkaddr:  NAT 起始地址
  ssa_blkaddr:  SSA 起始地址
  main_blkaddr: Main Area 起始地址

特殊 inode：
  root_ino = 3: 根目录 inode
  node_ino = 1: 用于 node 管理的特殊 inode
  meta_ino = 2: 用于元数据管理的特殊 inode
```

## 三、Checkpoint Area (CP)

### 3.1 Checkpoint 设计

F2FS 使用双 checkpoint 机制保证原子性：
- 两个 checkpoint 交替更新
- 通过版本号判断哪个是最新的
- 崩溃恢复时使用最新的有效 checkpoint

### 3.2 Checkpoint 结构

```c
// fs/f2fs/f2fs.h:1245
struct f2fs_checkpoint {
    __le64 checkpoint_ver;           /* checkpoint block version number */
    __le64 user_block_count;         /* # of user blocks */
    __le64 valid_block_count;        /* # of valid blocks */
    __le32 rsvd_segment_count;       /* # of reserved segments */
    __le32 overprov_segment_count;   /* # of overprovision segments */
    __le32 free_segment_count;       /* # of free segments */

    /* information of current node segments */
    __le32 cur_node_segno[MAX_ACTIVE_NODE_LOGS];
    __le16 cur_node_blkoff[MAX_ACTIVE_NODE_LOGS];
    
    /* information of current data segments */
    __le32 cur_data_segno[MAX_ACTIVE_DATA_LOGS];
    __le16 cur_data_blkoff[MAX_ACTIVE_DATA_LOGS];
    
    __le32 ckpt_flags;               /* checkpoint flags */
    __le32 cp_pack_total_block_count;/* # of blocks in checkpoint pack */
    __le32 cp_pack_start_sum;        /* start block number of summary */
    __le32 valid_node_count;         /* # of valid nodes */
    __le32 valid_inode_count;        /* # of valid inodes */
    __le32 next_free_nid;            /* next free node id */
    __le32 sit_ver_bitmap_bytesize;  /* version bitmap byte size */
    __le32 nat_ver_bitmap_bytesize;  /* version bitmap byte size */
    __le32 checksum_offset;          /* checksum offset inside cp block */
    __le64 elapsed_time;             /* elapsed time while partition is mounted */
    
    /* allocation type of current segment */
    unsigned char alloc_type[MAX_ACTIVE_LOGS];
    
    /* SIT and NAT version bitmap */
    unsigned char sit_nat_version_bitmap[];
} __packed;
```

### 3.3 Checkpoint Pack 布局

```
Checkpoint Pack 结构：
┌──────────────────────┐
│  Checkpoint Header   │ ← checkpoint 结构体
├──────────────────────┤
│  Orphan Blocks       │ ← 孤儿 inode 列表
├──────────────────────┤
│  Data Summary        │ ← 当前 data segment 的 summary
├──────────────────────┤
│  Node Summary        │ ← 当前 node segment 的 summary
├──────────────────────┤
│  Footer (CP copy)    │ ← checkpoint 结构体副本（用于校验）
└──────────────────────┘
```

### 3.4 关键字段说明

```
六个活跃段信息：
  cur_node_segno[3]: HOT/WARM/COLD NODE 当前段号
  cur_data_segno[3]: HOT/WARM/COLD DATA 当前段号
  cur_*_blkoff: 各段内的当前写入偏移

版本位图：
  sit_nat_version_bitmap: 标记 SIT/NAT 项的更新版本
  用于快速判断哪些元数据在上次 checkpoint 后被修改

检查点标志：
  CP_DISABLED_FLAG: checkpoint 被禁用
  CP_UMOUNT_FLAG: 干净卸载标志
  CP_COMPACT_SUM_FLAG: 压缩 summary
```

## 四、SIT (Segment Information Table)

### 4.1 SIT 作用

SIT 记录每个 segment 的使用情况：
- 有效块数量
- 有效块位图
- 修改时间

### 4.2 SIT Entry 结构

```c
// include/linux/f2fs_fs.h:213
struct f2fs_sit_entry {
    __le16 vblocks;              /* valid block count */
    __u8 valid_map[SIT_VBLOCK_MAP_SIZE]; /* valid bitmap */
    __le64 mtime;                /* modification time */
} __packed;

// SIT_VBLOCK_MAP_SIZE = 64 bytes
// 可以表示 512 个 blocks (1 segment) 的有效性
```

### 4.3 SIT 区域组织

```
SIT Area Layout:
┌────────────────┐
│   SIT Block 0  │ → 包含 segment 0-n 的 SIT entries
├────────────────┤
│   SIT Block 1  │
├────────────────┤
│      ...       │
├────────────────┤
│ SIT Block n-1  │
└────────────────┘

每个 SIT block 包含多个 SIT entries
每个 entry 74 bytes，一个 4KB block 可存 55 个 entries
```

## 五、NAT (Node Address Table)

### 5.1 NAT 作用

NAT 实现 node ID 到物理地址的映射，是 F2FS 间接寻址的核心。

### 5.2 NAT Entry 结构

```c
// include/linux/f2fs_fs.h:199
struct f2fs_nat_entry {
    __u8 version;                /* latest version of cached nat entry */
    __le32 ino;                  /* inode number */
    __le32 block_addr;           /* block address */
} __packed;

// NAT block 结构
struct f2fs_nat_block {
    struct f2fs_nat_entry entries[NAT_ENTRY_PER_BLOCK];
} __packed;

// NAT_ENTRY_PER_BLOCK = 455 (4096 / 9)
```

### 5.3 NAT 查找流程

```
查找 node_id = 1000 的物理地址：
1. nat_block_no = node_id / NAT_ENTRY_PER_BLOCK = 1000 / 455 = 2
2. nat_block_offset = node_id % NAT_ENTRY_PER_BLOCK = 1000 % 455 = 90
3. 读取 NAT block 2
4. 获取 entries[90].block_addr
```

## 六、SSA (Segment Summary Area)

### 6.1 SSA 作用

SSA 记录每个 block 的反向映射信息，用于：
- 垃圾回收时快速找到 block 的所有者
- 恢复时重建索引

### 6.2 Summary Entry 结构

```c
// include/linux/f2fs_fs.h:175
struct f2fs_summary {
    __le32 nid;                  /* node id */
    union {
        __u8 reserved[3];
        struct {
            __u8 version;        /* node version number */
            __le16 ofs_in_node;  /* offset in parent node */
        } __packed;
    };
} __packed;

// Summary block
struct f2fs_summary_block {
    struct f2fs_summary entries[ENTRIES_IN_SUM];
    struct f2fs_journal journal;  /* NAT/SIT journal */
    struct f2fs_summary_footer footer;
} __packed;
```

### 6.3 Journal 机制

为了减少元数据更新开销，F2FS 在 summary block 中内嵌 journal：

```c
// fs/f2fs/segment.h:259
struct f2fs_journal {
    union {
        struct nat_journal {
            struct nat_journal_entry entries[NAT_JOURNAL_ENTRIES];
            __u8 reserved[NAT_JOURNAL_RESERVED];
        } nat_j;
        struct sit_journal {
            struct sit_journal_entry entries[SIT_JOURNAL_ENTRIES];
            __u8 reserved[SIT_JOURNAL_RESERVED];
        } sit_j;
    };
} __packed;
```

## 七、Main Area

### 7.1 Main Area 组织

Main Area 是实际存储用户数据和 node 的区域：

```
Main Area 分区：
┌────────────────────────────────────────────────────┐
│  HOT_NODE  │ WARM_NODE │ COLD_NODE │ HOT_DATA ... │
└────────────────────────────────────────────────────┘
     ↑            ↑           ↑           ↑
   目录项      直接节点    间接节点    目录数据

6 个并发写入流（Curseg）：
- CURSEG_HOT_NODE:  频繁更新的 node（目录）
- CURSEG_WARM_NODE: 一般 node（直接节点）
- CURSEG_COLD_NODE: 很少更新的 node（间接节点）
- CURSEG_HOT_DATA:  频繁更新的数据（目录数据）
- CURSEG_WARM_DATA: 一般用户数据
- CURSEG_COLD_DATA: 冷数据（多媒体文件）
```

### 7.2 Node 块结构

```c
// include/linux/f2fs_fs.h:239
struct f2fs_node {
    union {
        struct f2fs_inode i;     /* inode */
        struct direct_node dn;   /* direct node */
        struct indirect_node in; /* indirect node */
    };
    struct node_footer footer;
} __packed;

// Node footer
struct node_footer {
    __le32 nid;                  /* node id */
    __le32 ino;                  /* inode number */
    __le32 flag;                 /* node flags */
    __le64 cp_ver;               /* checkpoint version */
    __le32 next_blkaddr;         /* next node block address */
} __packed;
```

### 7.3 Data 块组织

数据块直接存储文件内容，通过 node 中的地址数组索引：

```
文件数据寻址：
inode (in node block)
  → i_addr[0-922]: 直接数据块地址
  → i_nid[0]: direct node ID
      → direct_node.addr[0-1017]: 数据块地址
  → i_nid[1]: direct node ID  
  → i_nid[2]: indirect node ID
      → indirect_node.nid[0-1017]: direct node ID
  → i_nid[3]: double indirect node ID
```

## 八、磁盘布局示例

### 8.1 创建 10GB F2FS 分区

```bash
# 格式化
$ mkfs.f2fs -l test_f2fs /dev/loop0

# 查看布局信息
$ dump.f2fs /dev/loop0

Info: Segments per section = 1
Info: Sections per zone = 1  
Info: sector size = 512
Info: total sectors = 20971520 (10240 MB)
Info: zone aligned segment0 blkaddr: 512
Info: format version with
  "Linux version 5.15.0 (gcc 9.3.0)"

Info: [/dev/loop0] Disk Model: 
Info: Segments per section = 1
Info: Sections per zone = 1
Info: total FS sectors = 20971520 (10240 MB)
Info: TOTAL          segments: 5115
Info: MAIN AREA      segments: 5115 (in 5115 sections)
  - Data segments: 5056 (11008 MB)
  - Node segments: 59 (118 MB)
Info: SIT area: 3 segments (6 MB)
Info: NAT area: 5 segments (10 MB)  
Info: CP area: 2 segments (4 MB)
Info: SSA area: 3 segments (6 MB)

Super block:
  magic                  [0xf2f52010]
  major_version          [1]
  minor_version          [13]
  block_count            [2621440]
  segment_count          [5120]
  segment_count_main     [5115]
  start of main          [0x2800] (block 10240)
```

### 8.2 地址计算示例

```
假设：
- Block size = 4KB
- Segment size = 2MB = 512 blocks
- cp_blkaddr = 0x400 (block 1024)
- main_blkaddr = 0x2800 (block 10240)

计算 segment 100 的起始地址：
segment_start = main_blkaddr + segment_no * blocks_per_seg
              = 10240 + 100 * 512
              = 61440 (block)
              = 240 MB (offset)

计算 node ID 1000 的 NAT 位置：
nat_seg_no = node_id / (NAT_ENTRY_PER_BLOCK * blocks_per_seg)
           = 1000 / (455 * 512)
           = 0
nat_block_no = (node_id % (455 * 512)) / 455
             = 1000 / 455
             = 2
nat_entry_no = node_id % 455
             = 90
```

## 九、高级特性布局

### 9.1 多设备支持

```c
struct f2fs_device {
    __u8 path[MAX_PATH_LEN];
    __le32 total_segments;
} __packed;

// Superblock 中最多支持 16 个设备
// 设备间使用 RAID0 模式分布数据
```

### 9.2 压缩集群布局

```
压缩集群（假设 log_cluster_size = 2）：
┌────────────┬────────────┬────────────┬────────────┐
│ Compressed │ Compressed │   Dummy    │   Dummy    │
│   Data 0   │   Data 1   │  Block 2   │  Block 3   │
└────────────┴────────────┴────────────┴────────────┘
     ↑             ↑            ↑            ↑
   有效数据    有效数据      填充块       填充块

地址数组中，dummy block 地址设为 NULL_ADDR
```

### 9.3 Zone 支持（SMR/ZNS）

```
Zone-aware F2FS:
┌──────────────┬──────────────┬──────────────┐
│ Random Zone  │ Sequential   │ Sequential   │
│ (元数据)      │ Zone 1       │ Zone 2       │
└──────────────┴──────────────┴──────────────┘
     ↑              ↑              ↑
  随机写区域     顺序写区域     顺序写区域
```

## 十、调试与分析

### 10.1 使用 hexdump 查看结构

```bash
# 查看 superblock
$ sudo hexdump -C /dev/loop0 -n 4096

# 查看 checkpoint
$ sudo hexdump -C /dev/loop0 -s $((0x400*4096)) -n 4096

# 查看特定 segment
$ sudo hexdump -C /dev/loop0 -s $((segment_no*2*1024*1024)) -n $((2*1024*1024))
```

### 10.2 使用 parse.f2fs 分析

```bash
# 解析 superblock
$ sudo parse.f2fs -s /dev/loop0

# 解析 checkpoint  
$ sudo parse.f2fs -c /dev/loop0

# 解析 NAT
$ sudo parse.f2fs -n /dev/loop0

# 解析 SIT
$ sudo parse.f2fs -S /dev/loop0
```

### 10.3 手工恢复关键数据

```python
#!/usr/bin/env python3
# 读取 F2FS superblock
import struct

def read_superblock(device):
    with open(device, 'rb') as f:
        # Read first 4KB
        data = f.read(4096)
        
        # Parse magic number
        magic = struct.unpack('<I', data[0:4])[0]
        if magic != 0xF2F52010:
            print("Not a F2FS filesystem")
            return
            
        # Parse version
        major = struct.unpack('<H', data[4:6])[0]
        minor = struct.unpack('<H', data[6:8])[0]
        print(f"F2FS version: {major}.{minor}")
        
        # Parse layout info
        log_blocksize = struct.unpack('<I', data[12:16])[0]
        print(f"Block size: {1 << log_blocksize} bytes")
        
        # Parse areas
        cp_blkaddr = struct.unpack('<I', data[48:52])[0]
        sit_blkaddr = struct.unpack('<I', data[52:56])[0]
        nat_blkaddr = struct.unpack('<I', data[56:60])[0]
        ssa_blkaddr = struct.unpack('<I', data[60:64])[0]
        main_blkaddr = struct.unpack('<I', data[64:68])[0]
        
        print(f"CP starts at block: {cp_blkaddr}")
        print(f"SIT starts at block: {sit_blkaddr}")
        print(f"NAT starts at block: {nat_blkaddr}")
        print(f"SSA starts at block: {ssa_blkaddr}")
        print(f"Main starts at block: {main_blkaddr}")

if __name__ == "__main__":
    read_superblock("/dev/loop0")
```

## 十一、常见问题

1. **Q: 为什么 F2FS 元数据区域这么大？**
   A: 为了支持快速查找和减少更新开销，用空间换时间

2. **Q: SIT 和 SSA 的区别？**
   A: SIT 记录段的有效块信息，SSA 记录块的反向映射

3. **Q: 为什么需要双 checkpoint？**
   A: 保证原子更新，崩溃时总有一个有效的 checkpoint

4. **Q: Journal 的作用？**
   A: 减少 NAT/SIT 更新的写入开销，批量提交

5. **Q: 如何计算文件的物理位置？**
   A: inode → node → NAT → 物理地址

## 十二、总结

F2FS 的磁盘布局体现了其设计理念：
- **间接索引**：通过 NAT 实现灵活的地址映射
- **元数据分离**：SIT/NAT/SSA 各司其职
- **日志结构**：Main Area 的追加写入
- **冷热分离**：6 个并发日志流
- **快速恢复**：Checkpoint + Journal + Summary

理解这些布局是深入研究 F2FS 的基础。