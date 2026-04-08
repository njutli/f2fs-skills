---
name: f2fs-garbage-collection
description: F2FS垃圾回收(GC)机制详解。当用户询问F2FS的GC工作原理、前台/后台GC、Victim选择算法、Greedy/Cost-benefit算法、数据迁移流程、SSR机制时调用此技能。
---

# F2FS 垃圾回收机制详解

## 〇、为什么需要垃圾回收？

### 日志结构文件系统的宿命

F2FS 作为日志结构文件系统，采用异地更新（out-of-place update）策略：
```
传统文件系统（就地更新）：
文件A: [Block 100] → 更新 → [Block 100']（覆盖）

F2FS（异地更新）：
文件A: [Block 100] → 更新 → [Block 500]（追加到日志末尾）
        ↓
      Block 100 变为无效，等待垃圾回收
```

这种设计带来的问题：
1. **空间碎片化**：无效块散布在各个段中
2. **空间耗尽**：没有完整的空闲段可用
3. **写入阻塞**：必须先回收空间才能继续写入

### 垃圾回收的目标

- **回收空间**：将碎片化的段中的有效数据搬移，释放完整的段
- **性能优化**：减少回收开销，避免长时间阻塞
- **冷热分离**：回收时重新组织数据，提高后续回收效率

## 一、核心数据结构

### 1.1 Victim 选择结构

```c
// fs/f2fs/gc.h
struct victim_sel_policy {
    int alloc_mode;          /* LFS or SSR */
    int gc_mode;             /* GC_CB or GC_GREEDY */
    unsigned long *dirty_segmap;      /* dirty segment bitmap */
    unsigned int max_search;          /* maximum # of segments to search */
    unsigned int offset;              /* last victim segment offset */
    unsigned int ofs_unit;            /* bitmap search unit */
    unsigned int min_cost;            /* minimum cost */
    unsigned int min_segno;           /* segment number having min cost */
};

// GC 模式
enum {
    GC_CB,           /* Cost-Benefit */
    GC_GREEDY,       /* Greedy */
    ALLOC_NEXT,      /* Should allocate next free segment */
    FLUSH_DEVICE,    /* Flush device cache */
    MAX_GC_POLICY,
};

// fs/f2fs/f2fs.h
struct f2fs_gc_control {
    unsigned int victim_segno;   /* target victim segment number */
    int init_gc_type;           /* FG_GC or BG_GC */
    bool no_bg_gc;              /* check the space and stop bg_gc */
    bool should_migrate_blocks; /* should migrate blocks */
    bool err_gc_skipped;        /* return EAGAIN if GC skipped */
    unsigned int nr_free_secs;  /* # of free sections to do GC */
};
```

### 1.2 段信息管理

```c
// fs/f2fs/segment.h
struct seg_entry {
    unsigned int type:6;              /* segment type like CURSEG_XXX */
    unsigned int valid_blocks:10;     /* # of valid blocks */
    unsigned int ckpt_valid_blocks:10;/* # of valid blocks in checkpoint */
    unsigned int cur_valid_map_nr:6;  /* # of valid blocks in current bitmap */
    
    /*
     * # of valid blocks in the segment before last checkpoint
     * for victim selection algorithm
     */
    unsigned int ckpt_valid_map_nr:6;
    
    unsigned long long mtime;         /* modification time */
    
    /* bitmap of valid blocks */
    unsigned long *cur_valid_map;     /* validity bitmap of blocks */
    unsigned long *ckpt_valid_map;    /* validity bitmap in checkpoint */
    
    unsigned long *discard_map;       /* discard bitmap */
};
```

## 二、垃圾回收触发机制

### 2.1 前台 GC（Foreground GC）

```c
// fs/f2fs/segment.c
int f2fs_need_SSR(struct f2fs_sb_info *sbi)
{
    int node_secs = get_blocktype_secs(sbi, F2FS_DIRTY_NODES);
    int dent_secs = get_blocktype_secs(sbi, F2FS_DIRTY_DENTS);
    int imeta_secs = get_blocktype_secs(sbi, F2FS_DIRTY_IMETA);
    
    /* 检查是否有足够的空闲段 */
    if (free_sections(sbi) <= ovp_holes)
        return true;
    
    /* 检查各类型段的使用情况 */
    if (has_not_enough_free_secs(sbi, node_secs))
        return true;
    
    return false;
}

// 触发前台 GC
static int f2fs_gc(struct f2fs_sb_info *sbi, struct f2fs_gc_control *gc_control)
{
    int gc_type = gc_control->init_gc_type;
    
    /* 前台 GC 必须立即执行 */
    if (gc_type == FG_GC) {
        ret = do_garbage_collect(sbi, segno, &gc_list, gc_type);
        if (ret)
            goto stop;
    }
}
```

### 2.2 后台 GC（Background GC）

```c
// fs/f2fs/gc.c
static int gc_thread_func(void *data)
{
    struct f2fs_sb_info *sbi = data;
    struct f2fs_gc_kthread *gc_th = sbi->gc_thread;
    wait_queue_head_t *wq = &gc_th->gc_wait_queue;
    
    do {
        wait_event_interruptible_timeout(*wq,
            kthread_should_stop() || gc_th->gc_wake,
            msecs_to_jiffies(gc_th->min_sleep_time));
            
        /* 检查是否需要 GC */
        if (has_enough_free_secs(sbi, 0))
            continue;
            
        /* 执行后台 GC */
        gc_control.init_gc_type = BG_GC;
        f2fs_gc(sbi, &gc_control);
        
    } while (!kthread_should_stop());
}

// GC 触发条件
static inline bool has_not_enough_free_secs(struct f2fs_sb_info *sbi,
                                           int freed)
{
    int node_secs = get_blocktype_secs(sbi, F2FS_DIRTY_NODES);
    int dent_secs = get_blocktype_secs(sbi, F2FS_DIRTY_DENTS);
    
    if (free_sections(sbi) + freed <= 
        reserved_sections(sbi) + needed_for_cp)
        return true;
        
    return false;
}
```

## 三、Victim 选择算法

### 3.1 Greedy 算法

```c
// fs/f2fs/gc.c
static unsigned int get_max_cost(struct f2fs_sb_info *sbi,
                                struct victim_sel_policy *p)
{
    /* Greedy 算法：选择有效块最少的段 */
    if (p->gc_mode == GC_GREEDY)
        return BLKS_PER_SEC(sbi);  /* 最大代价 = 段大小 */
    
    /* Cost-Benefit 算法的最大代价 */
    return UINT_MAX;
}

// Greedy victim 选择
static int get_victim_by_default(struct f2fs_sb_info *sbi,
        unsigned int *result, int gc_type, int type, 
        char alloc_mode)
{
    struct victim_sel_policy p;
    unsigned int secno, last_victim;
    
    /* 贪心策略：寻找有效块最少的段 */
    p.min_cost = get_max_cost(sbi, &p);
    
    for_each_set_bit(secno, dirty_i->dirty_segmap[DIRTY], MAIN_SECS(sbi)) {
        struct seg_entry *se = get_seg_entry(sbi, segno);
        
        /* 跳过正在使用的段 */
        if (sec_usage_check(sbi, secno))
            continue;
            
        /* 计算代价：有效块数 */
        cost = se->valid_blocks;
        
        if (cost < p.min_cost) {
            p.min_cost = cost;
            p.min_segno = segno;
        }
        
        /* 找到空段，立即返回 */
        if (cost == 0)
            break;
    }
    
    *result = p.min_segno;
    return 0;
}
```

### 3.2 Cost-Benefit 算法

```c
// fs/f2fs/gc.c
static unsigned int get_cb_cost(struct f2fs_sb_info *sbi, 
                               unsigned int segno)
{
    struct sit_info *sit_i = SIT_I(sbi);
    struct seg_entry *se = get_seg_entry(sbi, segno);
    unsigned int valid_blocks = se->valid_blocks;
    unsigned int usable_blocks = sbi->blocks_per_seg;
    unsigned long long mtime = se->mtime;
    unsigned long long age;
    unsigned int cost;
    
    /* 计算段年龄 */
    age = get_mtime(sbi) - mtime;
    
    /* 
     * Cost-Benefit 公式：
     * cost = (1 - u) * age / (1 + u)
     * 其中：
     * - u: 段利用率（valid_blocks / total_blocks）
     * - age: 段年龄
     * 
     * 目标：选择年龄大但有效块少的段
     */
    if (valid_blocks == 0) {
        /* 空段，代价最小 */
        return UINT_MAX;
    } else if (valid_blocks == usable_blocks) {
        /* 满段，不选择 */
        return 0;
    }
    
    /* 归一化处理，避免溢出 */
    age = div64_u64(age * FACTOR, sbi->max_age);
    
    /* 计算最终代价 */
    cost = UINT_MAX - ((usable_blocks - valid_blocks) * age) / valid_blocks;
    
    return cost;
}

// 基于 Cost-Benefit 的 victim 选择
static void select_victim_cb(struct f2fs_sb_info *sbi,
                           struct victim_sel_policy *p)
{
    struct sit_info *sit_i = SIT_I(sbi);
    unsigned int segno, nsearched = 0;
    
    for_each_set_bit(segno, dirty_i->dirty_segmap[DIRTY], MAIN_SEGS(sbi)) {
        unsigned int cost = get_cb_cost(sbi, segno);
        
        /* 记录最大代价（即最适合回收）的段 */
        if (cost > p->min_cost) {
            p->min_cost = cost;
            p->min_segno = segno;
        }
        
        /* 限制搜索范围 */
        if (++nsearched >= p->max_search)
            break;
    }
}
```

## 四、数据迁移流程

### 4.1 识别有效块

```c
// fs/f2fs/gc.c
static int gc_data_segment(struct f2fs_sb_info *sbi, struct f2fs_summary *sum,
                          struct gc_inode_list *gc_list, unsigned int segno,
                          int gc_type)
{
    struct super_block *sb = sbi->sb;
    struct f2fs_summary *entry;
    block_t start_addr;
    int off;
    int phase = 0;
    
    start_addr = START_BLOCK(sbi, segno);
    
next_step:
    entry = sum;
    
    /* 遍历段中的所有块 */
    for (off = 0; off < sbi->blocks_per_seg; off++, entry++) {
        struct page *data_page;
        struct inode *inode;
        struct node_info dni;
        unsigned int ofs_in_node;
        nid_t nid = le32_to_cpu(entry->nid);
        
        /* 通过 SIT 检查块是否有效 */
        if (!is_valid_data_blkaddr(start_addr + off))
            continue;
            
        /* 获取块的所有者信息 */
        inode = f2fs_gc_get_inode(sb, nid, gc_list);
        if (!inode)
            continue;
            
        /* 读取数据页 */
        data_page = f2fs_get_read_data_page(inode, 
                        start_addr + off, REQ_RAHEAD);
        
        /* 移动数据 */
        move_data_page(inode, data_page, gc_type);
    }
}
```

### 4.2 数据搬移实现

```c
// fs/f2fs/gc.c
static int move_data_page(struct inode *inode, struct page *page,
                         int gc_type)
{
    struct f2fs_io_info fio = {
        .sbi = F2FS_I_SB(inode),
        .type = DATA,
        .temp = COLD,
        .op = REQ_OP_WRITE,
        .op_flags = REQ_SYNC,
        .old_blkaddr = page->index,
        .page = page,
        .encrypted_page = NULL,
    };
    
    if (gc_type == BG_GC) {
        /* 
         * 后台 GC：标记页面为脏，交给后台进程处理
         * 优点：不阻塞，可以聚合写入
         */
        if (PageDirty(page)) {
            /* 页面已经是脏的，无需重复标记 */
            goto out;
        }
        set_page_dirty(page);
        set_cold_data(page);
    } else {
        /*
         * 前台 GC：立即写入
         * 需要等待 IO 完成
         */
        f2fs_wait_on_page_writeback(page, DATA, true, true);
        
        /* 分配新位置并写入 */
        err = f2fs_do_write_data_page(&fio);
        if (err)
            goto out;
            
        /* 清理原位置 */
        clear_page_dirty_for_io(page);
    }
}
```

### 4.3 Node 搬移

```c
// fs/f2fs/gc.c  
static int gc_node_segment(struct f2fs_sb_info *sbi,
                struct f2fs_summary *sum, unsigned int segno, 
                int gc_type)
{
    struct f2fs_summary *entry;
    int off;
    
    entry = sum;
    
    for (off = 0; off < sbi->blocks_per_seg; off++, entry++) {
        nid_t nid = le32_to_cpu(entry->nid);
        struct page *node_page;
        
        /* 检查 node 是否有效 */
        if (!is_alive_node(sbi, entry, segno, off))
            continue;
            
        /* 读取 node page */
        node_page = f2fs_get_node_page(sbi, nid);
        if (IS_ERR(node_page))
            continue;
            
        /* 搬移 node */
        move_node_page(node_page, gc_type);
    }
}
```

## 五、SSR（Slack Space Recycling）

### 5.1 SSR 触发条件

```c
// fs/f2fs/segment.c
static int get_ssr_segment(struct f2fs_sb_info *sbi, int type,
                          int alloc_mode, unsigned int *segno)
{
    struct curseg_info *curseg = CURSEG_I(sbi, type);
    
    /*
     * SSR：在碎片化的段中寻找空闲块
     * 优先级：
     * 1. 找相同温度的段
     * 2. 找有效块最少的段
     */
    if (IS_NODESEG(type)) {
        /* Node 段的 SSR */
        for (i = CURSEG_HOT_NODE; i <= CURSEG_COLD_NODE; i++)
            cnt += get_valid_blocks(sbi, curseg->segno, false);
    } else {
        /* Data 段的 SSR */  
        for (i = CURSEG_HOT_DATA; i <= CURSEG_COLD_DATA; i++)
            cnt += get_valid_blocks(sbi, curseg->segno, false);
    }
    
    /* 选择碎片最多的段 */
    return get_victim(sbi, segno, BG_GC, alloc_mode, SSR);
}
```

### 5.2 SSR vs Normal Allocation

```c
// fs/f2fs/segment.c
void f2fs_allocate_segment_for_resize(struct f2fs_sb_info *sbi,
                                     int type, unsigned int start,
                                     unsigned int end)
{
    struct curseg_info *curseg = CURSEG_I(sbi, type);
    unsigned int segno;
    
    if (f2fs_need_SSR(sbi) && get_ssr_segment(sbi, type, SSR, &segno)) {
        /* 使用 SSR：在已有段中分配 */
        change_curseg(sbi, type, true);
        return;
    }
    
    /* Normal allocation：分配新段 */
    segno = get_new_segment(sbi, &curseg->next_segno, true, false);
    curseg->next_blkoff = 0;
}
```

## 六、GC 优化技术

### 6.1 Adaptive GC

```c
// fs/f2fs/gc.c
static int f2fs_gc_range(struct f2fs_sb_info *sbi,
                        unsigned int start_seg, unsigned int end_seg,
                        bool dry_run, unsigned int dry_run_sections)
{
    /* 
     * 自适应 GC：根据空闲空间动态调整策略
     * - 空间充足：使用 Cost-Benefit，优化长期性能
     * - 空间紧张：使用 Greedy，快速回收
     */
    if (free_sections(sbi) <= NEED_SSR_SECTIONS) {
        p.gc_mode = GC_GREEDY;
        p.max_search = sbi->max_victim_search;
    } else {
        p.gc_mode = GC_CB;
        p.max_search = DEF_MAX_VICTIM_SEARCH;
    }
}
```

### 6.2 Section 对齐

```c
// 确保 GC 单位与闪存擦除块对齐
static inline bool sec_usage_check(struct f2fs_sb_info *sbi, 
                                  unsigned int secno)
{
    /* 检查整个 section 的使用情况 */
    if (IS_CURSEC(sbi, secno))
        return true;
        
    /* 
     * 只有整个 section 都可以回收时才选择
     * 避免 FTL 层的数据迁移
     */
    for (i = 0; i < sbi->segs_per_sec; i++) {
        segno = secno * sbi->segs_per_sec + i;
        if (get_valid_blocks(sbi, segno, true))
            return true;
    }
    return false;
}
```

### 6.3 GC 限流

```c
// fs/f2fs/gc.h
struct f2fs_gc_kthread {
    struct task_struct *f2fs_gc_task;
    wait_queue_head_t gc_wait_queue;
    
    /* GC 线程休眠时间 */
    unsigned int min_sleep_time;
    unsigned int max_sleep_time;
    unsigned int no_gc_sleep_time;
    
    /* GC 触发控制 */
    unsigned int gc_idle;        /* 空闲时触发 */
    unsigned int gc_urgent;      /* 紧急模式 */
    unsigned int gc_wake;        /* 唤醒标志 */
};

// 动态调整 GC 频率
static inline void increase_sleep_time(struct f2fs_sb_info *sbi)
{
    struct f2fs_gc_kthread *gc_th = sbi->gc_thread;
    
    /* 空间充足，减少 GC 频率 */
    if (gc_th->min_sleep_time < gc_th->max_sleep_time)
        gc_th->min_sleep_time += SLEEP_TIME_STEP;
}
```

## 七、调试与监控

### 7.1 GC 统计信息

```bash
# 查看 GC 状态
$ cat /sys/kernel/debug/f2fs/status | grep GC
GC calls: 1234 (BG: 1000, FG: 234)
  - data segments : 800 (BG: 700, FG: 100)
  - node segments : 434 (BG: 300, FG: 134)
Valid: 1234567 blocks
Dirty: 234 segments, 456 sections

# 查看 Victim 分布
$ cat /sys/fs/f2fs/sda1/victim_bits
Victim bitmap: 0-10%, 10-20%, ..., 90-100%
Distribution: [1234, 567, ..., 89]
```

### 7.2 跟踪 GC 行为

```bash
# 启用 GC tracepoint
$ echo 1 > /sys/kernel/debug/tracing/events/f2fs/f2fs_gc_begin/enable
$ echo 1 > /sys/kernel/debug/tracing/events/f2fs/f2fs_gc_end/enable

# 查看 trace
$ cat /sys/kernel/debug/tracing/trace_pipe
f2fs_gc-1234 [001] .... 12345.678: f2fs_gc_begin: dev=8,0 type=BG_GC
f2fs_gc-1234 [001] .... 12345.789: f2fs_gc_end: dev=8,0 ret=0 freed=512
```

### 7.3 GC 性能分析

```python
#!/usr/bin/env python3
# 分析 GC 效率

import re
import sys

def analyze_gc_log(logfile):
    gc_stats = {
        'total_gc': 0,
        'fg_gc': 0,
        'bg_gc': 0,
        'data_moved': 0,
        'time_spent': 0
    }
    
    with open(logfile, 'r') as f:
        for line in f:
            if 'f2fs_gc_begin' in line:
                gc_stats['total_gc'] += 1
                if 'FG_GC' in line:
                    gc_stats['fg_gc'] += 1
                else:
                    gc_stats['bg_gc'] += 1
            elif 'f2fs_gc_end' in line:
                m = re.search(r'freed=(\d+)', line)
                if m:
                    gc_stats['data_moved'] += int(m.group(1))
    
    print(f"Total GC: {gc_stats['total_gc']}")
    print(f"FG GC: {gc_stats['fg_gc']} ({gc_stats['fg_gc']/gc_stats['total_gc']*100:.1f}%)")
    print(f"BG GC: {gc_stats['bg_gc']} ({gc_stats['bg_gc']/gc_stats['total_gc']*100:.1f}%)")
    print(f"Data moved: {gc_stats['data_moved']} blocks")
    print(f"Average moved per GC: {gc_stats['data_moved']/gc_stats['total_gc']:.1f} blocks")

if __name__ == "__main__":
    analyze_gc_log(sys.argv[1])
```

## 八、GC 调优建议

### 8.1 参数调优

```bash
# 1. 调整 GC 线程休眠时间
$ echo 300 > /sys/fs/f2fs/sda1/gc_min_sleep_time   # 最小 300ms
$ echo 60000 > /sys/fs/f2fs/sda1/gc_max_sleep_time # 最大 60s

# 2. 设置 GC 空闲触发
$ echo 1 > /sys/fs/f2fs/sda1/gc_idle    # 只在系统空闲时 GC

# 3. 调整 Victim 搜索范围
$ echo 4096 > /sys/fs/f2fs/sda1/max_victim_search

# 4. 预留空间配置
$ mkfs.f2fs -o 10 /dev/sda1  # 预留 10% 空间
```

### 8.2 应用场景优化

```
1. 写密集场景：
   - 增加预留空间（-o 15）
   - 使用 Greedy 算法（更快）
   - 减少 gc_min_sleep_time

2. 读密集场景：
   - 减少预留空间（-o 5）
   - 使用 Cost-Benefit 算法
   - 增加 gc_max_sleep_time

3. 混合负载：
   - 默认配置
   - 启用 adaptive GC
   - 监控并动态调整
```

## 九、常见问题

1. **Q: 为什么 GC 后空间没有立即释放？**
   A: 需要等待下一个 checkpoint 完成，确保崩溃恢复

2. **Q: 如何减少前台 GC？**
   A: 增加预留空间，优化数据冷热分离，及时执行后台 GC

3. **Q: SSR 会影响性能吗？**
   A: 会产生随机写，但避免了前台 GC 的长时间阻塞

4. **Q: 如何判断 GC 效率？**
   A: 查看平均每次 GC 搬移的数据量，越少越好

5. **Q: GC 与 FTL 的关系？**
   A: F2FS GC 尽量按 Section 对齐，减少 FTL 层的 GC

## 十、总结

F2FS 的垃圾回收机制体现了以下设计思想：
- **分级回收**：后台 GC 预防，前台 GC 应急
- **智能选择**：Greedy 快速回收，Cost-Benefit 优化分布
- **灵活策略**：Normal logging 与 SSR 结合
- **性能优先**：后台 GC 使用脏页机制，减少影响
- **FTL 友好**：Section 对齐，减少设备层开销

理解 GC 机制对于 F2FS 性能调优至关重要。