---
name: f2fs-node-address-table
description: F2FS节点地址表(NAT)机制详解。当用户询问F2FS的NAT工作原理、节点ID映射、NAT缓存机制、NAT日志、地址转换流程、NAT更新与持久化时调用此技能。
---

# F2FS NAT (Node Address Table) 机制详解

## 〇、为什么需要 NAT？

### 传统文件系统的问题

传统文件系统（如 ext4）使用直接块地址：
```
inode → 直接指向数据块地址
问题：
1. 数据移动困难：GC 需要更新所有引用
2. 磨损不均衡：热数据总在固定位置
3. 碎片整理复杂：需要更新大量指针
```

### NAT 的解决方案

F2FS 引入间接映射层：
```
inode → node ID → NAT → 物理块地址
优势：
1. GC 时只需更新 NAT，无需修改 inode
2. 支持透明的数据迁移
3. 实现 wandering tree 问题的解决
```

## 一、核心数据结构

### 1.1 NAT Entry

```c
// include/linux/f2fs_fs.h:199
struct f2fs_nat_entry {
    __u8 version;           /* 版本号，用于一致性检查 */
    __le32 ino;             /* 所属 inode 号 */
    __le32 block_addr;      /* 物理块地址 */
} __packed;

// 内存中的 NAT entry
// fs/f2fs/node.h:45
struct nat_entry {
    struct rb_node rb_node;  /* rb-tree node */
    struct list_head list;   /* lru list */
    nid_t nid;              /* node ID */
    union {
        struct {
            unsigned int version;    /* version number */
            unsigned int ino;        /* inode number */
            block_t blk_addr;        /* block address */
        };
        struct {
            /* for free nid list */
            struct list_head free_list;
        };
    };
};

// NAT entry 集合
// fs/f2fs/node.h:190
struct nat_entry_set {
    struct list_head set_list;   /* link with other nat sets */
    struct list_head entry_list; /* link with nat entries */
    nid_t start_nid;            /* start nid of set */
    unsigned int entry_cnt;     /* entry count in set */
};
```

### 1.2 NAT Block 结构

```c
// include/linux/f2fs_fs.h:205
struct f2fs_nat_block {
    struct f2fs_nat_entry entries[NAT_ENTRY_PER_BLOCK];
} __packed;

// NAT_ENTRY_PER_BLOCK = 455
// 每个 4KB block 可存储 455 个 NAT entries
```

### 1.3 NAT 管理结构

```c
// fs/f2fs/f2fs.h:1000
struct f2fs_nm_info {
    /* NAT 缓存管理 */
    struct radix_tree_root nat_root;      /* radix tree for NAT entries */
    rwlock_t nat_tree_lock;               /* protect nat tree */
    struct list_head nat_entries;         /* lru list for NAT entries */
    spinlock_t nat_list_lock;            /* protect lru list */
    unsigned int nat_cnt[MAX_NAT_STATE];  /* NAT entry count */
    unsigned int nat_blocks;             /* # of nat blocks */

    /* NAT 位图管理 */
    unsigned char *nat_bitmap;           /* NAT bitmap pointer */
    unsigned int nat_bitmap_size;        /* bitmap size */
    
    /* 空闲 NID 管理 */
    struct radix_tree_root free_nid_root; /* free nid tree */
    struct list_head free_nid_list;      /* free nid list */
    spinlock_t nid_list_lock;           /* protect nid list */
    unsigned int nid_cnt[MAX_NID_STATE]; /* nid count */
    
    /* NAT journal */
    struct f2fs_journal *nat_j;          /* NAT journal */
    
    /* 其他字段... */
};
```

## 二、NAT 查找流程

### 2.1 地址转换过程

```c
// fs/f2fs/node.c:530
static int get_node_info(struct f2fs_sb_info *sbi, nid_t nid,
                        struct node_info *ni)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry *ne;
    struct page *page;
    int i;

    /* 1. 先查找内存缓存 */
    down_read(&nm_i->nat_tree_lock);
    ne = radix_tree_lookup(&nm_i->nat_root, nid);
    if (ne) {
        /* 缓存命中 */
        ni->ino = ne->ino;
        ni->blk_addr = ne->blk_addr;
        ni->version = ne->version;
        up_read(&nm_i->nat_tree_lock);
        return 0;
    }
    up_read(&nm_i->nat_tree_lock);

    /* 2. 缓存未命中，需要从磁盘读取 */
    /* 计算 NAT block 位置 */
    pgoff_t nat_blk_no = nid / NAT_ENTRY_PER_BLOCK;
    int entry_off = nid % NAT_ENTRY_PER_BLOCK;
    
    /* 读取 NAT block */
    page = f2fs_get_meta_page(sbi, nat_blk_no);
    if (IS_ERR(page))
        return PTR_ERR(page);
        
    /* 3. 解析 NAT entry */
    struct f2fs_nat_block *nat_blk = page_address(page);
    struct f2fs_nat_entry *ne_disk = &nat_blk->entries[entry_off];
    
    ni->ino = le32_to_cpu(ne_disk->ino);
    ni->blk_addr = le32_to_cpu(ne_disk->block_addr);
    ni->version = ne_disk->version;
    
    /* 4. 加入缓存 */
    cache_nat_entry(sbi, nid, ne_disk);
    
    f2fs_put_page(page, 1);
    return 0;
}
```

### 2.2 NAT 缓存机制

```c
// fs/f2fs/node.c:450
static struct nat_entry *cache_nat_entry(struct f2fs_sb_info *sbi,
                    nid_t nid, struct f2fs_nat_entry *ne)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry *new_ne, *e;

    /* 分配新的 NAT entry */
    new_ne = kmem_cache_alloc(nat_entry_slab, GFP_F2FS_ZERO);
    if (!new_ne)
        return NULL;

    /* 初始化 */
    nat_set_nid(new_ne, nid);
    nat_set_version(new_ne, ne->version);
    nat_set_ino(new_ne, le32_to_cpu(ne->ino));
    nat_set_blkaddr(new_ne, le32_to_cpu(ne->block_addr));
    
    /* 加入 radix tree */
    down_write(&nm_i->nat_tree_lock);
    e = radix_tree_insert(&nm_i->nat_root, nid, new_ne);
    if (!e) {
        /* 成功插入，加入 LRU 链表 */
        list_add_tail(&new_ne->list, &nm_i->nat_entries);
        nm_i->nat_cnt[TOTAL_NAT]++;
        
        /* 检查是否需要收缩缓存 */
        if (nm_i->nat_cnt[TOTAL_NAT] > NM_WOUT_THRESHOLD)
            try_to_free_nats(sbi, NM_WOUT_THRESHOLD);
    }
    up_write(&nm_i->nat_tree_lock);
    
    return new_ne;
}
```

## 三、NAT 更新流程

### 3.1 更新 NAT Entry

```c
// fs/f2fs/node.c:590
static void set_node_addr(struct f2fs_sb_info *sbi, struct node_info *ni,
                         block_t new_blkaddr, bool fsync_done)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry *e;
    
    down_write(&nm_i->nat_tree_lock);
    
    /* 查找或创建 NAT entry */
    e = __lookup_nat_cache(nm_i, ni->nid);
    if (!e) {
        e = __init_nat_entry(nm_i, ni->nid, true);
        if (!e) {
            up_write(&nm_i->nat_tree_lock);
            return;
        }
    }
    
    /* 更新地址 */
    nat_set_blkaddr(e, new_blkaddr);
    nat_set_ino(e, ni->ino);
    nat_set_version(e, ni->version);
    
    /* 标记为脏 */
    if (!__is_set_nat_flag(e, IS_DIRTY)) {
        __set_nat_flag(e, IS_DIRTY, true);
        __add_dirty_nat_entry(nm_i, e);
    }
    
    /* 如果是 fsync 操作，需要立即持久化 */
    if (fsync_done)
        __set_nat_flag(e, HAS_FSYNCED, true);
        
    up_write(&nm_i->nat_tree_lock);
}
```

### 3.2 NAT Journal 机制

为了减少频繁的 NAT block 更新，F2FS 使用 journal 缓存少量 NAT 更新：

```c
// fs/f2fs/node.c:2890
static int __update_nat_bits(struct f2fs_sb_info *sbi, nid_t nid,
                            struct f2fs_nat_entry *ne)
{
    struct f2fs_journal *journal = &curseg->sum_blk->journal;
    int i;

    /* 检查 journal 是否已满 */
    if (journal->n_nats >= NAT_JOURNAL_ENTRIES)
        return -ENOSPC;

    /* 查找是否已在 journal 中 */
    for (i = 0; i < journal->n_nats; i++) {
        if (journal->nat_j.entries[i].nid == cpu_to_le32(nid)) {
            /* 更新已存在的 entry */
            journal->nat_j.entries[i] = *ne;
            return 0;
        }
    }

    /* 添加新 entry */
    journal->nat_j.entries[journal->n_nats] = *ne;
    journal->nat_j.entries[journal->n_nats].nid = cpu_to_le32(nid);
    journal->n_nats++;
    
    return 0;
}
```

## 四、NAT 持久化

### 4.1 Checkpoint 时 NAT 写回

```c
// fs/f2fs/node.c:2950
int f2fs_flush_nat_entries(struct f2fs_sb_info *sbi)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry_set *set, *tmp;
    LIST_HEAD(sets);
    
    /* 1. 收集所有脏 NAT entries */
    down_write(&nm_i->nat_tree_lock);
    list_for_each_entry_safe(set, tmp, &nm_i->dirty_nat_sets, set_list) {
        /* 每个 set 包含一个 NAT block 中的脏 entries */
        list_move_tail(&set->set_list, &sets);
    }
    up_write(&nm_i->nat_tree_lock);

    /* 2. 写回每个 NAT block */
    list_for_each_entry(set, &sets, set_list) {
        struct page *page;
        nid_t start_nid = set->start_nid;
        pgoff_t nat_blk = start_nid / NAT_ENTRY_PER_BLOCK;
        
        /* 读取 NAT block */
        page = f2fs_get_meta_page(sbi, nat_blk);
        
        /* 更新脏 entries */
        __flush_nat_entries(sbi, set, page);
        
        /* 写回 */
        set_page_dirty(page);
        f2fs_put_page(page, 1);
    }
    
    return 0;
}

// 实际写入 NAT entries
static void __flush_nat_entries(struct f2fs_sb_info *sbi,
            struct nat_entry_set *set, struct page *page)
{
    struct f2fs_nat_block *nat_blk = page_address(page);
    struct nat_entry *ne, *tmp;
    
    /* 遍历 set 中的所有脏 entries */
    list_for_each_entry_safe(ne, tmp, &set->entry_list, list) {
        int offset = ne->nid % NAT_ENTRY_PER_BLOCK;
        struct f2fs_nat_entry *raw_ne = &nat_blk->entries[offset];
        
        /* 更新磁盘上的 NAT entry */
        raw_ne->version = ne->version;
        raw_ne->ino = cpu_to_le32(ne->ino);
        raw_ne->block_addr = cpu_to_le32(ne->blk_addr);
        
        /* 清除脏标记 */
        __clear_nat_flag(ne, IS_DIRTY);
        
        /* 如果地址为 NULL，可以释放这个 entry */
        if (ne->blk_addr == NULL_ADDR) {
            list_del(&ne->list);
            kmem_cache_free(nat_entry_slab, ne);
        }
    }
}
```

### 4.2 版本位图机制

F2FS 使用版本位图快速判断哪些 NAT entries 被修改：

```c
// fs/f2fs/node.c:2800
static void update_nat_bits(struct f2fs_sb_info *sbi, nid_t nid, int set)
{
    struct f2fs_checkpoint *ckpt = F2FS_CKPT(sbi);
    int nat_bit_offset = nid / NAT_ENTRY_PER_BLOCK;
    
    if (set) {
        /* 设置位图，表示这个 NAT block 有更新 */
        __set_bit(nat_bit_offset, nm_i->nat_bitmap);
    } else {
        /* 清除位图 */
        __clear_bit(nat_bit_offset, nm_i->nat_bitmap);
    }
    
    /* 更新 checkpoint 中的版本位图 */
    memcpy(ckpt->sit_nat_version_bitmap + ckpt->sit_ver_bitmap_bytesize,
           nm_i->nat_bitmap, nm_i->nat_bitmap_size);
}
```

## 五、空闲 Node ID 管理

### 5.1 Free NID 结构

```c
// fs/f2fs/node.h:200
struct free_nid {
    struct list_head list;   /* for free nid list */
    nid_t nid;              /* node id */
    int state;              /* FREE_NID or PREALLOC_NID */
};

// Free NID 状态
enum {
    FREE_NID,       /* 完全空闲，可分配 */
    PREALLOC_NID,   /* 预分配，等待使用 */
    MAX_NID_STATE,
};
```

### 5.2 分配 Node ID

```c
// fs/f2fs/node.c:2400
bool f2fs_alloc_nid(struct f2fs_sb_info *sbi, nid_t *nid)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct free_nid *i = NULL;
    
retry:
    /* 如果空闲 NID 不足，触发构建 */
    if (nm_i->nid_cnt[FREE_NID] < NAT_ENTRY_PER_BLOCK) {
        /* 扫描 NAT 构建空闲列表 */
        f2fs_build_free_nids(sbi, true, false);
    }

    /* 从空闲列表分配 */
    spin_lock(&nm_i->nid_list_lock);
    if (list_empty(&nm_i->free_nid_list)) {
        spin_unlock(&nm_i->nid_list_lock);
        return false;
    }
    
    /* 取第一个空闲 NID */
    i = list_first_entry(&nm_i->free_nid_list, struct free_nid, list);
    *nid = i->nid;
    
    /* 移动到预分配状态 */
    i->state = PREALLOC_NID;
    __move_free_nid(nm_i, i, FREE_NID, PREALLOC_NID);
    
    spin_unlock(&nm_i->nid_list_lock);
    return true;
}
```

### 5.3 构建空闲 NID 列表

```c
// fs/f2fs/node.c:2200
static int __f2fs_build_free_nids(struct f2fs_sb_info *sbi,
                                bool sync, bool mount)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    nid_t nid = nm_i->next_scan_nid;
    
    /* 扫描 NAT blocks */
    for (; nid < nm_i->max_nid; nid += NAT_ENTRY_PER_BLOCK) {
        struct page *page;
        pgoff_t nat_blk = nid / NAT_ENTRY_PER_BLOCK;
        
        /* 检查版本位图，跳过未修改的块 */
        if (!__test_bit(nat_blk, nm_i->nat_bitmap))
            continue;
            
        /* 读取 NAT block */
        page = f2fs_get_meta_page(sbi, nat_blk);
        scan_nat_block(sbi, page, nid);
        f2fs_put_page(page, 1);
        
        /* 如果收集够了，可以停止 */
        if (nm_i->nid_cnt[FREE_NID] >= NAT_ENTRY_PER_BLOCK)
            break;
    }
    
    nm_i->next_scan_nid = nid;
    return 0;
}
```

## 六、NAT 优化技术

### 6.1 NAT 缓存收缩

```c
// fs/f2fs/node.c:480
static int try_to_free_nats(struct f2fs_sb_info *sbi, int nr_shrink)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry *ne, *tmp;
    LIST_HEAD(victims);
    int nr_freed = 0;

    down_write(&nm_i->nat_tree_lock);
    
    /* 从 LRU 尾部选择受害者 */
    list_for_each_entry_safe_reverse(ne, tmp, &nm_i->nat_entries, list) {
        /* 跳过脏 entry */
        if (__is_set_nat_flag(ne, IS_DIRTY))
            continue;
            
        /* 移到受害者列表 */
        list_move(&ne->list, &victims);
        if (++nr_freed >= nr_shrink)
            break;
    }
    
    /* 释放受害者 */
    list_for_each_entry_safe(ne, tmp, &victims, list) {
        radix_tree_delete(&nm_i->nat_root, ne->nid);
        list_del(&ne->list);
        kmem_cache_free(nat_entry_slab, ne);
        nm_i->nat_cnt[TOTAL_NAT]--;
    }
    
    up_write(&nm_i->nat_tree_lock);
    return nr_freed;
}
```

### 6.2 批量 NAT 更新

```c
// fs/f2fs/node.c:2850
static void __update_nat_bits(struct f2fs_sb_info *sbi, 
                             struct nat_entry_set *set)
{
    struct f2fs_nm_info *nm_i = NM_I(sbi);
    struct nat_entry *ne;
    
    /* 批量处理一个 set 中的所有 entries */
    list_for_each_entry(ne, &set->entry_list, list) {
        if (ne->blk_addr == NULL_ADDR) {
            /* 空闲 node，更新空闲位图 */
            __set_bit(ne->nid, nm_i->free_nid_bitmap);
        } else {
            /* 已用 node，清除空闲位图 */
            __clear_bit(ne->nid, nm_i->free_nid_bitmap);
        }
    }
}
```

## 七、关键代码路径

### 7.1 读取文件时的 NAT 查找

```
f2fs_file_read_iter()
  → generic_file_read_iter()
    → f2fs_read_data_pages()
      → f2fs_get_dnode_of_data()
        → f2fs_get_node_page()
          → __get_node_page()
            → get_node_info()  // 查询 NAT
              → radix_tree_lookup() // 缓存查找
              → f2fs_get_meta_page() // 磁盘读取
```

### 7.2 写入文件时的 NAT 更新

```
f2fs_file_write_iter()
  → generic_file_write_iter()
    → f2fs_write_begin()
      → f2fs_get_block()
        → f2fs_map_blocks()
          → f2fs_allocate_data_block()
            → f2fs_update_data_blkaddr()
              → set_data_blkaddr()
                → set_node_addr() // 更新 NAT
```

### 7.3 Checkpoint 时的 NAT 持久化

```
f2fs_write_checkpoint()
  → do_checkpoint()
    → f2fs_flush_nat_entries()
      → __flush_nat_entries()
        → set_page_dirty() // 标记 NAT page 为脏
    → f2fs_submit_merged_write() // 提交写入
```

## 八、调试方法

### 8.1 查看 NAT 统计

```bash
# 查看 NAT 缓存统计
$ cat /sys/kernel/debug/f2fs/status | grep NAT
  - NAT entry count: 1234 (total: 2000)
  - NAT cache hit ratio: 95.2%
  - Dirty NAT entries: 56

# 查看具体的 NAT 信息
$ dump.f2fs -n /dev/sda1
[NAT info]
nid:      0 ino:      0 block_addr:     0x2 version: 0
nid:      1 ino:      1 block_addr:     0x3 version: 0
nid:      2 ino:      2 block_addr:     0x4 version: 0
...
```

### 8.2 Trace NAT 操作

```bash
# 开启 NAT 相关 tracepoint
$ echo 1 > /sys/kernel/debug/tracing/events/f2fs/f2fs_get_node_info/enable
$ echo 1 > /sys/kernel/debug/tracing/events/f2fs/f2fs_set_node_addr/enable

# 查看 trace
$ cat /sys/kernel/debug/tracing/trace_pipe
 <...>-1234  [001] .... 12345.678: f2fs_get_node_info: dev = 8,1 nid = 100 blkaddr = 0x1234
 <...>-1234  [001] .... 12345.679: f2fs_set_node_addr: dev = 8,1 nid = 100 blkaddr = 0x5678
```

### 8.3 手工解析 NAT

```python
#!/usr/bin/env python3
# 解析 F2FS NAT block

def parse_nat_block(device, nat_blkno):
    NAT_ENTRY_SIZE = 9  # bytes
    NAT_ENTRY_PER_BLOCK = 455
    BLOCK_SIZE = 4096
    
    with open(device, 'rb') as f:
        # Seek to NAT block
        f.seek(nat_blkno * BLOCK_SIZE)
        data = f.read(BLOCK_SIZE)
        
        for i in range(NAT_ENTRY_PER_BLOCK):
            offset = i * NAT_ENTRY_SIZE
            if offset + NAT_ENTRY_SIZE > len(data):
                break
                
            # Parse entry
            version = data[offset]
            ino = int.from_bytes(data[offset+1:offset+5], 'little')
            blk_addr = int.from_bytes(data[offset+5:offset+9], 'little')
            
            if blk_addr != 0:  # Skip empty entries
                nid = nat_blkno * NAT_ENTRY_PER_BLOCK + i
                print(f"NID: {nid:6d} -> Block: 0x{blk_addr:08x}, "
                      f"inode: {ino}, version: {version}")

if __name__ == "__main__":
    # 假设 NAT 从 block 0x1000 开始
    parse_nat_block("/dev/sda1", 0x1000)
```

## 九、性能影响

### 9.1 NAT 缓存大小

```c
// fs/f2fs/node.h:30
/* # of NAT entries to be cached */
#define DEF_NAT_CACHE_SIZE      (1 << 15)  /* 32K entries */
#define NM_WOUT_THRESHOLD       (64 * NAT_ENTRY_PER_BLOCK)

// 内存占用计算：
// 每个 nat_entry 约 64 bytes
// 32K entries = 2MB 内存
```

### 9.2 性能调优建议

1. **增大 NAT 缓存**
```bash
# 通过 mount option 调整（需要内核支持）
mount -o nat_cache_size=65536 /dev/sda1 /mnt
```

2. **监控缓存命中率**
```bash
# 定期检查命中率
while true; do
    hit=$(cat /sys/kernel/debug/f2fs/sda1/nat_hit)
    total=$(cat /sys/kernel/debug/f2fs/sda1/nat_total)
    ratio=$(echo "scale=2; $hit * 100 / $total" | bc)
    echo "NAT cache hit ratio: $ratio%"
    sleep 10
done
```

3. **减少 NAT 更新**
   - 使用更大的写入单位
   - 批量操作而非频繁小写入
   - 适当延长 checkpoint 间隔

## 十、常见问题

1. **Q: NAT 与 inode table 的区别？**
   A: inode table 存储文件元数据，NAT 存储 node 的地址映射

2. **Q: 为什么不直接用物理地址？**
   A: 间接映射支持透明的数据迁移，便于 GC 和磨损均衡

3. **Q: NAT entry 为什么是 9 字节？**
   A: 1字节版本 + 4字节inode + 4字节地址，紧凑设计

4. **Q: NAT 损坏会怎样？**
   A: 导致 node 无法定位，文件系统需要 fsck 修复

5. **Q: Journal 中的 NAT 与磁盘 NAT 的关系？**
   A: Journal 是缓存，checkpoint 时合并到磁盘 NAT

## 十一、总结

NAT 是 F2FS 实现其核心特性的基础：
- **间接寻址**：解耦逻辑地址与物理地址
- **高效缓存**：radix tree + LRU 管理
- **批量更新**：journal + set 机制减少写入
- **快速分配**：预构建的空闲 NID 列表
- **崩溃一致性**：版本号 + checkpoint 保证

理解 NAT 机制对于深入掌握 F2FS 的工作原理至关重要。