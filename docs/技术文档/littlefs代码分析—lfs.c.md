# littlefs代码分析—lfs.c

## 宏定义

```c
#define LFS_BLOCK_NULL ((lfs_block_t)-1)
#define LFS_BLOCK_INLINE ((lfs_block_t)-2)
```

约定无效块和内联数据块的地址的`id`是-1和-2

---

```c
#define LFS_MKTAG(type, id, size) \
    (((lfs_tag_t)(type) << 20) | ((lfs_tag_t)(id) << 10) | (lfs_tag_t)(size))

#define LFS_MKTAG_IF(cond, type, id, size) \
    ((cond) ? LFS_MKTAG(type, id, size) : LFS_MKTAG(LFS_FROM_NOOP, 0, 0))

#define LFS_MKTAG_IF_ELSE(cond, type1, id1, size1, type2, id2, size2) \
    ((cond) ? LFS_MKTAG(type1, id1, size1) : LFS_MKTAG(type2, id2, size2))
```

`littlefs`中tag构成如下：

```
        tag
[--      32      --]
[1| 3| 8 | 10 | 10 ]
 ^  ^  ^    ^    ^- size
 |  |  |    '------ id
 |  |  '----------- file type
 |  '-------------- type1 (0x0)
 '----------------- valid bit
```

这些宏（根据传入的条件）使用参数构建tag

---

```c
#define LFS_MKATTRS(...) \
    (struct lfs_mattr[]){__VA_ARGS__}, \
    sizeof((struct lfs_mattr[]){__VA_ARGS__}) / sizeof(struct lfs_mattr)
```

快速生成 `struct lfs_mattr` 类型的数组，同时计算数组长度。主要与前述生成tag的宏组合使用。

---

```c
#define LFS_DIR_TRAVERSE_DEPTH 3
```

最大目录遍历深度

---

```c
#ifdef LFS_THREADSAFE
#define LFS_LOCK(cfg)   cfg->lock(cfg)
#define LFS_UNLOCK(cfg) cfg->unlock(cfg)
#else
#define LFS_LOCK(cfg)   ((void)cfg, 0)
#define LFS_UNLOCK(cfg) ((void)cfg)
#endif
```

有线程安全需求时提供锁

## 别名

```c
typedef uint32_t lfs_tag_t;
typedef int32_t lfs_stag_t;
```

32位整数的tag

## 枚举

```c
enum {
    LFS_OK_RELOCATED = 1,
    LFS_OK_DROPPED   = 2,
    LFS_OK_ORPHANED  = 3,
};
```

提供移动成功、被删除和成为孤儿的状态码

---

```c
enum {
    LFS_CMP_EQ = 0,
    LFS_CMP_LT = 1,
    LFS_CMP_GT = 2,
};
```

提供比较结果的状态码

## 结构体

### `lfs_mattr`

```c
struct lfs_mattr {
    lfs_tag_t tag;
    const void *buffer;
};
```

包含tag和属性本身。

---

### `lfs_disk_off`

```c
struct lfs_diskoff {
    lfs_block_t block;
    lfs_off_t off;
};
```

包含块ID和块内偏移量

---

### `lfs_fcrc`

```c
struct lfs_fcrc {
    lfs_size_t size;
    uint32_t crc;
};
```

包含CRC值和待校验段的大小

---

### `lfs_dir_traverse`

```c
struct lfs_dir_traverse {
    const lfs_mdir_t *dir;
    lfs_off_t off;
    lfs_tag_t ptag;
    const struct lfs_mattr *attrs;
    int attrcount;

    lfs_tag_t tmask;
    lfs_tag_t ttag;
    uint16_t begin;
    uint16_t end;
    int16_t diff;

    int (*cb)(void *data, lfs_tag_t tag, const void *buffer);
    void *data;

    lfs_tag_t tag;
    const void *buffer;
    struct lfs_diskoff disk;
};
```

用于遍历目录：

* `lfs_mdir_t`:指向当前目录的元数据结构。
* ` off`:当前遍历的偏移量。
* `ptag`:父标签，用于标识父目录
* `lfs_mattr`:指向属性数组的指针
* `attracount`:属性数组的数量
* `tmask`:标签掩码，用于过滤标签
* `ttag`:目标标签，和`tmask`配合使用
* `begin`：遍历起始位置
* `end`:遍历结束位置
* `diff`:差异值，用于计算偏移
* `cb`和`data`:回调函数及其数据指针
* `tag`:当前读取到的标签
* `buffer`:当前处理的数据缓冲区
* `lfs_diskoff`:遍历到的磁盘位置

---

### `lfs_dir_find_match`

```c
struct lfs_dir_find_match {
    lfs_t *lfs;
    const void *name;
    lfs_size_t size;
};
```

用于查找目录

* `lfs`:文件系统实例指针
* `name`和`size`:文件名指针和大小

---

### `lfs_commit`

```c
struct lfs_commit {
    lfs_block_t block;
    lfs_off_t off;
    lfs_tag_t ptag;
    uint32_t crc;

    lfs_off_t begin;
    lfs_off_t end;
};
```

用于管理提交操作

* `block`:当前正在写入的块号
* `off`:当前在块内的偏移量
* `ptag`:父标签，用于标识父目录
* `crc`:用于校验的CRC值
* `begin`:提交区域的开始位置
* `end`:提交区域结束位置

---

### `lfs_dir_commit_commit`

```c
struct lfs_dir_commit_commit {
    lfs_t *lfs;
    struct lfs_commit *commit;
};
```

这命名真的是人类能想出来的吗？第一个 `commit` 表示这是一个目录的提交操作，第二个 `commit` 表示这是提交过程中的具体提交动作。

* `lfs`:文件系统实例指针
* `commit`：提交操作指针

---

### `lfs_tortoise_t`

```c
struct lfs_tortoise_t {
    lfs_block_t pair[2];
    lfs_size_t i;
    lfs_size_t period;
};
```

用于实现 Floyd 龟兔判圈算法的结构体

* `pair`:存储两个块号，其中兔子块是乌龟块的2倍，若迭代结束前遇到，则存在圈
* `i`：当前迭代次数
* `period`:存储循环长度

---

### `lfs_fs_parent_match`

```c
struct lfs_fs_parent_match {
    lfs_t *lfs;
    const lfs_block_t pair[2];
};
```

用于查找引用了特定块对的元数据块

* `lfs`:文件系统实例指针
* `paire`:要查找其父块的目标块

## 块设备与缓存

激动人心的时刻正式开始！到这里为止，我们将正式开始对`littlefs`的核心实现进行分析，将近200个函数，就从这里开始吧！

### `lfs_cache_drop`

```c
static inline void lfs_cache_drop(lfs_t *lfs, lfs_cache_t *rcache) {
    // do not zero, cheaper if cache is readonly or only going to be
    // written with identical data (during relocates)
    (void)lfs;
    rcache->block = LFS_BLOCK_NULL;
}
```

将一个`cache`中的`block`设定为无效块，相当于只读模式下的`lfs_cache_zero`。这里虽然没有使用到`lfs`，但是可能为了将来的开发和保持接口一致性还是把它留下来了。另外这里虽然参数名叫`rcache`，但是也被用于释放`pcache`。

---

### `lfs_cache_zero`

```c
static inline void lfs_cache_zero(lfs_t *lfs, lfs_cache_t *pcache) {
    // zero to avoid information leak
    memset(pcache->buffer, 0xff, lfs->cfg->cache_size);
    pcache->block = LFS_BLOCK_NULL;
}
```

将一个`cache`中的`block`设定为无效块，并清空缓存，由于闪存的读写特性在擦除时会将所有位设置为1，所以虽然叫`cache_zero` ，但是用`0xFF`而不是`0x00`来填充。

---

### `lfs_bd_read`

```c
static int lfs_bd_read(lfs_t *lfs,
        const lfs_cache_t *pcache, lfs_cache_t *rcache, lfs_size_t hint,
        lfs_block_t block, lfs_off_t off,
        void *buffer, lfs_size_t size) {
    uint8_t *data = buffer;
    if (off+size > lfs->cfg->block_size
            || (lfs->block_count && block >= lfs->block_count)) {
        return LFS_ERR_CORRUPT;
    }

    while (size > 0) {
        lfs_size_t diff = size;

        if (pcache && block == pcache->block &&
                off < pcache->off + pcache->size) {
            if (off >= pcache->off) {
                // is already in pcache?
                diff = lfs_min(diff, pcache->size - (off-pcache->off));
                memcpy(data, &pcache->buffer[off-pcache->off], diff);

                data += diff;
                off += diff;
                size -= diff;
                continue;
            }

            // pcache takes priority
            diff = lfs_min(diff, pcache->off-off);
        }

        if (block == rcache->block &&
                off < rcache->off + rcache->size) {
            if (off >= rcache->off) {
                // is already in rcache?
                diff = lfs_min(diff, rcache->size - (off-rcache->off));
                memcpy(data, &rcache->buffer[off-rcache->off], diff);

                data += diff;
                off += diff;
                size -= diff;
                continue;
            }

            // rcache takes priority
            diff = lfs_min(diff, rcache->off-off);
        }

        if (size >= hint && off % lfs->cfg->read_size == 0 &&
                size >= lfs->cfg->read_size) {
            // bypass cache?
            diff = lfs_aligndown(diff, lfs->cfg->read_size);
            int err = lfs->cfg->read(lfs->cfg, block, off, data, diff);
            if (err) {
                return err;
            }

            data += diff;
            off += diff;
            size -= diff;
            continue;
        }

        // load to cache, first condition can no longer fail
        LFS_ASSERT(!lfs->block_count || block < lfs->block_count);
        rcache->block = block;
        rcache->off = lfs_aligndown(off, lfs->cfg->read_size);
        rcache->size = lfs_min(
                lfs_min(
                    lfs_alignup(off+hint, lfs->cfg->read_size),
                    lfs->cfg->block_size)
                - rcache->off,
                lfs->cfg->cache_size);
        int err = lfs->cfg->read(lfs->cfg, rcache->block,
                rcache->off, rcache->buffer, rcache->size);
        LFS_ASSERT(err <= 0);
        if (err) {
            return err;
        }
    }

    return 0;
}
```

块设备读函数。函数参数：

- `lfs_t *lfs` ：文件系统实例
- `const lfs_cache_t *pcache`：写入缓存
- `lfs_cache_t *rcache`：读缓存
- `lfs_size_t hint`：读取提示大小
- `lfs_block_t block` ：要读取的块号
- `lfs_off_t off`：块内偏移量
- `void *buffer`：输出缓冲区
- `lfs_size_t size`：要读取的字节数

首先进行边界检查，如块内偏移或块ID本身越界，返回错误。（不过这里返回的是坏块错误，这意味着坏块吗？？）

在循环里先看看`pcache`里有没有能用的，就算不全都能用，也要把能用的部分写到`buffer`里。如果命中缓存了，但是缓存的起始块内偏移量更大，就把两个偏移量之间的部分先给读了，然后直接进下次循环。这之后对`rcache`做完全一样的处理。

然后如果`size`比传入的`hint`大，并且是对齐的，那就直接绕过缓存调用常规读取，向下对齐到`read_size`的倍数以后调用`lfs->cfg`里给的块设备读取函数。（这里可能会留一点比`read_size`更小的东西。)

接下来，我们把读缓存对齐到目标读入的位置，把大小设定为`cache_size`和`hint`预期读入大小中较小的一个，然后调用块设备读取加载内容到缓存并进入下一次读取。这样下一次读入时就一定会命中缓存。

---

### `lfs_bd_cmp`

```c
static int lfs_bd_cmp(lfs_t *lfs,
        const lfs_cache_t *pcache, lfs_cache_t *rcache, lfs_size_t hint,
        lfs_block_t block, lfs_off_t off,
        const void *buffer, lfs_size_t size) {
    const uint8_t *data = buffer;
    lfs_size_t diff = 0;

    for (lfs_off_t i = 0; i < size; i += diff) {
        uint8_t dat[8];

        diff = lfs_min(size-i, sizeof(dat));
        int err = lfs_bd_read(lfs,
                pcache, rcache, hint-i,
                block, off+i, &dat, diff);
        if (err) {
            return err;
        }

        int res = memcmp(dat, data + i, diff);
        if (res) {
            return res < 0 ? LFS_CMP_LT : LFS_CMP_GT;
        }
    }

    return LFS_CMP_EQ;
}
```

比较块设备中的数据和给定的`buffer`是否一致。函数参数：

- `lfs_t *lfs`:文件系统实例
- `const lfs_cache_t *pcache`:写入缓存
- `lfs_cache_t *rcache`: 读缓存
- `lfs_size_t hint`:读取提示大小
- `lfs_block_t block`:要比较的块号
- `lfs_off_t off`:块内偏移量
- `const void *buffer`:要比较的数据缓冲区
- `lfs_size_t size`:要比较的字节数

调用`lfs_bd_read`,8字节8字节的读，然后和给定的`buffer`比，根据比较结果返回比较结果的枚举。这里虽然会调用很多次`lfs_bd_read`，但只要`hint`给定不太小，就会命中很多次缓存，所以不会太影响性能。

---

### `lfs_db_crc`

```c
static int lfs_bd_crc(lfs_t *lfs,
        const lfs_cache_t *pcache, lfs_cache_t *rcache, lfs_size_t hint,
        lfs_block_t block, lfs_off_t off, lfs_size_t size, uint32_t *crc) {
    lfs_size_t diff = 0;

    for (lfs_off_t i = 0; i < size; i += diff) {
        uint8_t dat[8];
        diff = lfs_min(size-i, sizeof(dat));
        int err = lfs_bd_read(lfs,
                pcache, rcache, hint-i,
                block, off+i, &dat, diff);
        if (err) {
            return err;
        }

        *crc = lfs_crc(*crc, &dat, diff);
    }

    return 0;
}
```

计算块设备数据 CRC 校验值的函数。函数参数：

- `lfs_t *lfs`:文件系统实例
- `const lfs_cache_t *pcache`:写入缓存
- `lfs_cache_t *rcache`:读缓存
- `lfs_size_t hint`:读取提示大小
- `lfs_block_t block`:块号
- `lfs_off_t off`;偏移量
- `lfs_size_t size`:要计算CRC的数据大小
- `uint32_t *crc`: CRC结果的指针

调用`lfs_bd_read`,8字节8字节的读并更新CRC值。虽说同样会命中缓存吧，但是为啥一定要8字节8字节地读呢？

---

### `lfs_bd_flush`

```c
static int lfs_bd_flush(lfs_t *lfs,
        lfs_cache_t *pcache, lfs_cache_t *rcache, bool validate) {
    if (pcache->block != LFS_BLOCK_NULL && pcache->block != LFS_BLOCK_INLINE) {
        LFS_ASSERT(pcache->block < lfs->block_count);
        lfs_size_t diff = lfs_alignup(pcache->size, lfs->cfg->prog_size);
        int err = lfs->cfg->prog(lfs->cfg, pcache->block,
                pcache->off, pcache->buffer, diff);
        LFS_ASSERT(err <= 0);
        if (err) {
            return err;
        }

        if (validate) {
            // check data on disk
            lfs_cache_drop(lfs, rcache);
            int res = lfs_bd_cmp(lfs,
                    NULL, rcache, diff,
                    pcache->block, pcache->off, pcache->buffer, diff);
            if (res < 0) {
                return res;
            }

            if (res != LFS_CMP_EQ) {
                return LFS_ERR_CORRUPT;
            }
        }

        lfs_cache_zero(lfs, pcache);
    }

    return 0;
}
```

将写缓存中的数据刷新到块设备，并验证有没有坏块（可选）。

简单地检验合法性以后把写缓存的中的数据刷新到块设备，然后清空写缓存。在`littlefs`中检验坏块的办法是写入后立马读出来看看一不一样，读入时还要更新读缓存。

---

### `lfs_bd_sync`

```c
static int lfs_bd_sync(lfs_t *lfs,
        lfs_cache_t *pcache, lfs_cache_t *rcache, bool validate) {
    lfs_cache_drop(lfs, rcache);

    int err = lfs_bd_flush(lfs, pcache, rcache, validate);
    if (err) {
        return err;
    }

    err = lfs->cfg->sync(lfs->cfg);
    LFS_ASSERT(err <= 0);
    return err;
}
```

块设备同步函数。先使读缓存失效，然后将写缓存中的数据刷新到块设备，最后调用块设备本身的同步函数，ez。

---

### `lfs_bd_prog`

```c
static int lfs_bd_prog(lfs_t *lfs,
        lfs_cache_t *pcache, lfs_cache_t *rcache, bool validate,
        lfs_block_t block, lfs_off_t off,
        const void *buffer, lfs_size_t size) {
    const uint8_t *data = buffer;
    LFS_ASSERT(block == LFS_BLOCK_INLINE || block < lfs->block_count);
    LFS_ASSERT(off + size <= lfs->cfg->block_size);

    while (size > 0) {
        if (block == pcache->block &&
                off >= pcache->off &&
                off < pcache->off + lfs->cfg->cache_size) {
            // already fits in pcache?
            lfs_size_t diff = lfs_min(size,
                    lfs->cfg->cache_size - (off-pcache->off));
            memcpy(&pcache->buffer[off-pcache->off], data, diff);

            data += diff;
            off += diff;
            size -= diff;

            pcache->size = lfs_max(pcache->size, off - pcache->off);
            if (pcache->size == lfs->cfg->cache_size) {
                // eagerly flush out pcache if we fill up
                int err = lfs_bd_flush(lfs, pcache, rcache, validate);
                if (err) {
                    return err;
                }
            }

            continue;
        }

        // pcache must have been flushed, either by programming and
        // entire block or manually flushing the pcache
        LFS_ASSERT(pcache->block == LFS_BLOCK_NULL);

        // prepare pcache, first condition can no longer fail
        pcache->block = block;
        pcache->off = lfs_aligndown(off, lfs->cfg->prog_size);
        pcache->size = 0;
    }

    return 0;
}
```

没啥可说的，和`lfs_bd_read`的执行逻辑基本一致。区别在于写函数只检查和操作写入缓存，并且没有传入参数`hint`。

---

### `lfs_bd_erase`

```c
static int lfs_bd_erase(lfs_t *lfs, lfs_block_t block) {
    LFS_ASSERT(block < lfs->block_count);
    int err = lfs->cfg->erase(lfs->cfg, block);
    LFS_ASSERT(err <= 0);
    return err;
}
```

这位更是ez，那么到此为止，所有的快函数操作函数就都看完了。总结一下，这些函数非平凡的部分其实就是在`lfs->cfg`接受的外部块设备函数的基础上，增加了读、写两个缓存块和有关的操作。



## 工具函数

### `lfs_size_t`

```c
static inline lfs_size_t lfs_path_namelen(const char *path) {
    return strcspn(path, "/");
}
```

返回路径首个构成的长度（即到第一个/之前）

---

### `lfs_path_islast`

```c
static inline bool lfs_path_islast(const char *path) {
    lfs_size_t namelen = lfs_path_namelen(path);
    return path[namelen + strspn(path + namelen, "/")] == '\0';
}
```

判断路径是不是只剩下最后一段构成

---

### `lfs_path_isdir`

```c
static inline bool lfs_path_isdir(const char *path) {
    return path[lfs_path_namelen(path)] != '\0';
}
```

通过判断是不是只有一段构成来判断是不是目录。但是这个也挺奇怪的，例如"dir/file.txt"这种文件路径，也会被判断为是目录。`littlefs`的实现不太可能犯如此低级的的错误，大概率是设计如此，并不是一般意义上的判断函数。

---

### `lfs_pair_swap`

```c
static inline void lfs_pair_swap(lfs_block_t pair[2]) {
    lfs_block_t t = pair[0];
    pair[0] = pair[1];
    pair[1] = t;
}
```

交换一个块对的指向

---

### `lfs_pair_isnull`

```c
static inline bool lfs_pair_isnull(const lfs_block_t pair[2]) {
    return pair[0] == LFS_BLOCK_NULL || pair[1] == LFS_BLOCK_NULL;
}
```

利用宏判断块对是否为无效块

---

### `lfs_pair_cmp`

```c
static inline int lfs_pair_cmp(
        const lfs_block_t paira[2],
        const lfs_block_t pairb[2]) {
    return !(paira[0] == pairb[0] || paira[1] == pairb[1] ||
             paira[0] == pairb[1] || paira[1] == pairb[0]);
}
```

检查两个块对是否有一样的部分，注意，是没有一样的部分才返回`true`

---

### `lfs_pair_issync`

```c
static inline bool lfs_pair_issync(
        const lfs_block_t paira[2],
        const lfs_block_t pairb[2]) {
    return (paira[0] == pairb[0] && paira[1] == pairb[1]) ||
           (paira[0] == pairb[1] && paira[1] == pairb[0]);
}
```

检查两个块对是否完全一致（不要求顺序）

---

### `lfs_pair_fromle32`

```c
static inline void lfs_pair_fromle32(lfs_block_t pair[2]) {
    pair[0] = lfs_fromle32(pair[0]);
    pair[1] = lfs_fromle32(pair[1]);
}
```

将小端序块对转化为大端序（仅在大端序的机器上）。

---

### `lfs_pair_tole32`

```c
static inline void lfs_pair_tole32(lfs_block_t pair[2]) {
    pair[0] = lfs_tole32(pair[0]);
    pair[1] = lfs_tole32(pair[1]);
}
```

将大端序块对转化为小端序

---

### `lfs_tag_isvalid`

```c
static inline bool lfs_tag_isvalid(lfs_tag_t tag) {
    return !(tag & 0x80000000);
}
```

通过看第1位是不是0检验tag是否有效

---

### `lfs_tag_isdelete`

```c
static inline bool lfs_tag_isdelete(lfs_tag_t tag) {
    return ((int32_t)(tag << 22) >> 22) == -1;
}
```

通过检验后10位的`lenth`字段是否为`0x3ff`的特殊值判断是否被删除

---

### `lfs_tag_type1`

```c
static inline uint16_t lfs_tag_type1(lfs_tag_t tag) {
    return (tag & 0x70000000) >> 20;
}
```

读取第2~4位的`type`字段掩码，确定其所在分类

---

### `lfs_tag_type2`

```c
static inline uint16_t lfs_tag_type2(lfs_tag_t tag) {
    return (tag & 0x78000000) >> 20;
}
```

读取2~5位的`type`掩码，不过这种读法并没有在[SPEC.md](./docs/技术文档/littlefs_SPEC_zh)中说明，文档中认为只有3位的掩码

---

### `lfs_tag_type3`

```c
static inline uint16_t lfs_tag_type2(lfs_tag_t tag) {
    return (tag & 0x7ff00000) >> 20;
}
```

读取2~12位的完整`type`编码

---

### `lfs_tag_chunk`

```c
static inline uint8_t lfs_tag_chunk(lfs_tag_t tag) {
    return (tag & 0x0ff00000) >> 20;
}
```

读取第5-12位块字段，`type1`+`chunk`=`type`。

---

### `lfs_tag_splice`

```c
static inline int8_t lfs_tag_splice(lfs_tag_t tag) {
    return (int8_t)lfs_tag_chunk(tag);
}
```

把`chunk`转为8字节整数。

---

### `lfs_tag_id`

```c
static inline uint16_t lfs_tag_id(lfs_tag_t tag) {
    return (tag & 0x000ffc00) >> 10;
}
```

读取第13~22位的`id`，`type`+`id`是标签的唯一标识。

---

### `lfs_tag_size`

```c
static inline lfs_size_t lfs_tag_size(lfs_tag_t tag) {
    return tag & 0x000003ff;
}
```

读取最后10位的`length`字段，注意`0x3fff`表示删除

---

### `lfs_tag_dsize`

```c
static inline lfs_size_t lfs_tag_dsize(lfs_tag_t tag) {
    return sizeof(tag) + lfs_tag_size(tag + lfs_tag_isdelete(tag));
}
```

查看标签及指向内容的长度，若被删除则为4字节，否则为4字节+`length`。

---

### `lfs_gstate_xor`

```c
static inline void lfs_gstate_xor(lfs_gstate_t *a, const lfs_gstate_t *b) {
    for (int i = 0; i < 3; i++) {
        ((uint32_t*)a)[i] ^= ((const uint32_t*)b)[i];
    }
}
```

对两个全局状态取异或，每个全局状态由3个32位整数构成。

---

### `lfs_gstate_iszero`

```c
static inline bool lfs_gstate_iszero(const lfs_gstate_t *a) {
    for (int i = 0; i < 3; i++) {
        if (((uint32_t*)a)[i] != 0) {
            return false;
        }
    }
    return true;
}
```

判断全局状态是否全为0

---

### `lfs_gstate_hasorphans`

```c
static inline bool lfs_gstate_hasorphans(const lfs_gstate_t *a) {
    return lfs_tag_size(a->tag);
}
```

在全局状态标签中，`tag`的`size`即表示当前孤儿节点的数量，

---

### `lfs_gstate_getorphans`

```c
static inline uint8_t lfs_gstate_getorphans(const lfs_gstate_t *a) {
    return lfs_tag_size(a->tag) & 0x1ff;
}
```

获取孤儿节点的数量，当然问题很明显，为什么不是和10位1取与而是和后9位，似乎是因为倒数第10位是用于标记是否涉及超级快

---

### `lfs_gstate_hasmove`

```c
static inline bool lfs_gstate_hasmove(const lfs_gstate_t *a) {
    return lfs_tag_type1(a->tag);
}
```

如果a的tag的`type1`字段非空，即认为发生了移动。虽说全局状态确实只有正在移动和没有正在移动这两种状态吧，但是感觉还是应该校验一下是不是`0x7ff`更合适。

---

### `lfs_gstate_needssuperblock`

```c
static inline bool lfs_gstate_needssuperblock(const lfs_gstate_t *a) {
    return lfs_tag_size(a->tag) >> 9;
}
```

检验是否需要超级快，即倒数第10位。

---

### `lfs_gstate_hasmovehere`

```c
static inline bool lfs_gstate_hasmovehere(const lfs_gstate_t *a,
        const lfs_block_t *pair) {
    return lfs_tag_type1(a->tag) && lfs_pair_cmp(a->pair, pair) == 0;
}
```

首先查看是否在移动中，其次看记录的块对是不是已经不一样了，如果都是，就认为（断电前）已经发生了移动。

---

### `lfs_gstate_fromle32`

```c
static inline void lfs_gstate_fromle32(lfs_gstate_t *a) {
    a->tag     = lfs_fromle32(a->tag);
    a->pair[0] = lfs_fromle32(a->pair[0]);
    a->pair[1] = lfs_fromle32(a->pair[1]);
}
```

将小端序全局状态转化为大端序（仅在大端序的机器上）。

---

### `lfs_gstate_tole32`

```c
static inline void lfs_gstate_tole32(lfs_gstate_t *a) {
    a->tag     = lfs_tole32(a->tag);
    a->pair[0] = lfs_tole32(a->pair[0]);
    a->pair[1] = lfs_tole32(a->pair[1]);
}
```

将大端序全局状态转化为小端序。

---

### `lfs_fcrc_fromle32`

```c
static void lfs_fcrc_fromle32(struct lfs_fcrc *fcrc) {
    fcrc->size = lfs_fromle32(fcrc->size);
    fcrc->crc = lfs_fromle32(fcrc->crc);
}
```

将小端序FCRC转化为大端序（仅在大端序的机器上）。

---

### `lfs_fcrc_tole32`

```c
static void lfs_fcrc_tole32(struct lfs_fcrc *fcrc) {
    fcrc->size = lfs_tole32(fcrc->size);
    fcrc->crc = lfs_tole32(fcrc->crc);
}
```

将大端序FCRC转化为小端序

---

### `lfs_ctz_fromle32`

```c
static void lfs_ctz_fromle32(struct lfs_ctz *ctz) {
    ctz->head = lfs_fromle32(ctz->head);
    ctz->size = lfs_fromle32(ctz->size);
}
```

将小端序CTZ转化为大端序（仅在大端序的机器上）。

---

### `lfs_ctz_tole32`

```c
static void lfs_ctz_tole32(struct lfs_ctz *ctz) {
    ctz->head = lfs_tole32(ctz->head);
    ctz->size = lfs_tole32(ctz->size);
}
```

将大端序CTZ转化为小端序



---

### `lfs_superblock_fromle32`

```c
static inline void lfs_superblock_fromle32(lfs_superblock_t *superblock) {
    superblock->version     = lfs_fromle32(superblock->version);
    superblock->block_size  = lfs_fromle32(superblock->block_size);
    superblock->block_count = lfs_fromle32(superblock->block_count);
    superblock->name_max    = lfs_fromle32(superblock->name_max);
    superblock->file_max    = lfs_fromle32(superblock->file_max);
    superblock->attr_max    = lfs_fromle32(superblock->attr_max);
}

```

将小端序超级块转化为大端序（仅在大端序的机器上）。

---

### `lfs_superblock_tole32`

```c
static inline void lfs_superblock_tole32(lfs_superblock_t *superblock) {
    superblock->version     = lfs_tole32(superblock->version);
    superblock->block_size  = lfs_tole32(superblock->block_size);
    superblock->block_count = lfs_tole32(superblock->block_count);
    superblock->name_max    = lfs_tole32(superblock->name_max);
    superblock->file_max    = lfs_tole32(superblock->file_max);
    superblock->attr_max    = lfs_tole32(superblock->attr_max);
}
```

将大端序超级块转化为小端序

---

### `lfs_mlist_isopen`

```c
static bool lfs_mlist_isopen(struct lfs_mlist *head,
        struct lfs_mlist *node) {
    for (struct lfs_mlist **p = &head; *p; p = &(*p)->next) {
        if (*p == (struct lfs_mlist*)node) {
            return true;
        }
    }

    return false;
}
```

给定一个`lfs_mlist`(的头)和一个节点，遍历改列表，如目标节点在列表中，返回`true`，否则返回`false`

---

### `lfs_mlist_remove`

```c
static void lfs_mlist_remove(lfs_t *lfs, struct lfs_mlist *mlist) {
    for (struct lfs_mlist **p = &lfs->mlist; *p; p = &(*p)->next) {
        if (*p == mlist) {
            *p = (*p)->next;
            break;
        }
    }
}
```

从给定的`lfs`的`mlist`中查找并删除一个给定的节点

---

### `lfs_mlist_append`

```c
static void lfs_mlist_append(lfs_t *lfs, struct lfs_mlist *mlist) {
    mlist->next = lfs->mlist;
    lfs->mlist = mlist;
}
```

将一个节点插入`lfs`的`mlist`并作为头

---

### `lfs_fs_disk_version`

```c
static uint32_t lfs_fs_disk_version(lfs_t *lfs) {
    (void)lfs;
#ifdef LFS_MULTIVERSION
    if (lfs->cfg->disk_version) {
        return lfs->cfg->disk_version;
    } else
#endif
    {
        return LFS_DISK_VERSION;
    }
}
```

返回磁盘版本，若指定则返回指定值，否则返回最新版本。

---

### ` lfs_fs_disk_version_major`

```c
static uint16_t lfs_fs_disk_version_major(lfs_t *lfs) {
    return 0xffff & (lfs_fs_disk_version(lfs) >> 16);

}
```

返回磁盘大版本

---

### `lfs_fs_disk_version_minor`

```c
static uint16_t lfs_fs_disk_version_minor(lfs_t *lfs) {
    return 0xffff & (lfs_fs_disk_version(lfs) >> 0);
}
```

返回磁盘小版本



## 块分配器

### `lfs_alloc_ckpoint`

```c
static void lfs_alloc_ckpoint(lfs_t *lfs) {
    lfs->lookahead.ckpoint = lfs->block_count;
}c
```

重置`checkpoint`到总块数，表示可以重新开始扫描所有块。

---

### `lfs_alloc_drop`

```c
static void lfs_alloc_drop(lfs_t *lfs) {
    lfs->lookahead.size = 0;
    lfs->lookahead.next = 0;
    lfs_alloc_ckpoint(lfs);
}
```

重置前瞻缓冲区，用于挂载和遍历失败时

---

### `lfs_alloc_lookahead`

```c
static int lfs_alloc_lookahead(void *p, lfs_block_t block) {
    lfs_t *lfs = (lfs_t*)p;
    lfs_block_t off = ((block - lfs->lookahead.start)
            + lfs->block_count) % lfs->block_count;

    if (off < lfs->lookahead.size) {
        lfs->lookahead.buffer[off / 8] |= 1U << (off % 8);
    }

    return 0;
}
```

把给定的块在前瞻缓冲区的位图中进行标记，当前瞻缓冲区的大小超过剩余块时，块起始的部分也将被包括。函数声明中没有直接使用`lfs`的指针是为了使函数可以被作为回调函数使用。

---

### `lfs_alloc_scan`

```c
static int lfs_alloc_scan(lfs_t *lfs) {
    // move lookahead buffer to the first unused block
    //
    // note we limit the lookahead buffer to at most the amount of blocks
    // checkpointed, this prevents the math in lfs_alloc from underflowing
    lfs->lookahead.start = (lfs->lookahead.start + lfs->lookahead.next) 
            % lfs->block_count;
    lfs->lookahead.next = 0;
    lfs->lookahead.size = lfs_min(
            8*lfs->cfg->lookahead_size,
            lfs->lookahead.ckpoint);

    // find mask of free blocks from tree
    memset(lfs->lookahead.buffer, 0, lfs->cfg->lookahead_size);
    int err = lfs_fs_traverse_(lfs, lfs_alloc_lookahead, lfs, true);
    if (err) {
        lfs_alloc_drop(lfs);
        return err;
    }

    return 0;
}
```

将前瞻缓冲区移动到首个未使用的块，从这里可以看出之前对`lookahead.next`字段的认识有误，**next 并不是“下一个空闲块”的直观位置**，而是当前扫描窗口中的索引，随着扫描逐步增加。除此之外，该块还把缓冲区全部设置为0，并将`lfs_alloc_lookahead`作为回调函数传入遍历函数，使得缓冲区中已经被使用过的块继续被设置为1。

---

### `lfs_alloc`

```c
static int lfs_alloc(lfs_t *lfs, lfs_block_t *block) {
    while (true) {
        // scan our lookahead buffer for free blocks
        while (lfs->lookahead.next < lfs->lookahead.size) {
            if (!(lfs->lookahead.buffer[lfs->lookahead.next / 8]
                    & (1U << (lfs->lookahead.next % 8)))) {
                // found a free block
                *block = (lfs->lookahead.start + lfs->lookahead.next)
                        % lfs->block_count;

                // eagerly find next free block to maximize how many blocks
                // lfs_alloc_ckpoint makes available for scanning
                while (true) {
                    lfs->lookahead.next += 1;
                    lfs->lookahead.ckpoint -= 1;

                    if (lfs->lookahead.next >= lfs->lookahead.size
                            || !(lfs->lookahead.buffer[lfs->lookahead.next / 8]
                                & (1U << (lfs->lookahead.next % 8)))) {
                        return 0;
                    }
                }
            }

            lfs->lookahead.next += 1;
            lfs->lookahead.ckpoint -= 1;
        }

        // In order to keep our block allocator from spinning forever when our
        // filesystem is full, we mark points where there are no in-flight
        // allocations with a checkpoint before starting a set of allocations.
        //
        // If we've looked at all blocks since the last checkpoint, we report
        // the filesystem as out of storage.
        //
        if (lfs->lookahead.ckpoint <= 0) {
            LFS_ERROR("No more free space 0x%"PRIx32,
                    (lfs->lookahead.start + lfs->lookahead.next)
                        % lfs->block_count);
            return LFS_ERR_NOSPC;
        }

        // No blocks in our lookahead buffer, we need to scan the filesystem for
        // unused blocks in the next lookahead window.
        int err = lfs_alloc_scan(lfs);
        if(err) {
            return err;
        }
    }
}
```

从文件系统中寻找并分配一个空闲块。首先在位图中查看当前索引位置（`next`标记的地方）是否是空闲块。如找不到，就逐个增加`next`，看下一个块，并逐个减少`checkpoint`(所以它的用处是标记一共还有多少块？似乎是的，毕竟开始的时候就标记为块总数)。如果找到，则提供给传入的`block`以此时的块号，然后将next一直增加到下一个空闲快的位置。至此，我们基于前瞻缓冲区实现了一个块分配器。

## 目录与元数据对操作

### `lfs_dir_getslice`

```c
static lfs_stag_t lfs_dir_getslice(lfs_t *lfs, const lfs_mdir_t *dir,
        lfs_tag_t gmask, lfs_tag_t gtag,
        lfs_off_t goff, void *gbuffer, lfs_size_t gsize) {
    lfs_off_t off = dir->off;
    lfs_tag_t ntag = dir->etag;
    lfs_stag_t gdiff = 0;

    // synthetic moves
    if (lfs_gstate_hasmovehere(&lfs->gdisk, dir->pair) &&
            lfs_tag_id(gmask) != 0) {
        if (lfs_tag_id(lfs->gdisk.tag) == lfs_tag_id(gtag)) {
            return LFS_ERR_NOENT;
        } else if (lfs_tag_id(lfs->gdisk.tag) < lfs_tag_id(gtag)) {
            gdiff -= LFS_MKTAG(0, 1, 0);
        }
    }

    // iterate over dir block backwards (for faster lookups)
    while (off >= sizeof(lfs_tag_t) + lfs_tag_dsize(ntag)) {
        off -= lfs_tag_dsize(ntag);
        lfs_tag_t tag = ntag;
        int err = lfs_bd_read(lfs,
                NULL, &lfs->rcache, sizeof(ntag),
                dir->pair[0], off, &ntag, sizeof(ntag));
        if (err) {
            return err;
        }

        ntag = (lfs_frombe32(ntag) ^ tag) & 0x7fffffff;

        if (lfs_tag_id(gmask) != 0 &&
                lfs_tag_type1(tag) == LFS_TYPE_SPLICE &&
                lfs_tag_id(tag) <= lfs_tag_id(gtag - gdiff)) {
            if (tag == (LFS_MKTAG(LFS_TYPE_CREATE, 0, 0) |
                    (LFS_MKTAG(0, 0x3ff, 0) & (gtag - gdiff)))) {
                // found where we were created
                return LFS_ERR_NOENT;
            }

            // move around splices
            gdiff += LFS_MKTAG(0, lfs_tag_splice(tag), 0);
        }

        if ((gmask & tag) == (gmask & (gtag - gdiff))) {
            if (lfs_tag_isdelete(tag)) {
                return LFS_ERR_NOENT;
            }

            lfs_size_t diff = lfs_min(lfs_tag_size(tag), gsize);
            err = lfs_bd_read(lfs,
                    NULL, &lfs->rcache, diff,
                    dir->pair[0], off+sizeof(tag)+goff, gbuffer, diff);
            if (err) {
                return err;
            }

            memset((uint8_t*)gbuffer + diff, 0, gsize - diff);

            return tag + gdiff;
        }
    }

    return LFS_ERR_NOENT;
}
```

从目录中读取匹配给定条件的数据切片，函数参数：
`lfs_t *lfs`：文件系统实例
`const lfs_mdir_t *dir` ：目录元数据
`lfs_tag_t gmask`：标签掩码，用于匹配
`lfs_tag_t gtag`：目标标签
`lfs_off_t goff` ：数据偏移量
`void *gbuffer` ：输出缓冲区
`lfs_size_t gsize`：要读取的大小

这里呢实际上是允许从匹配的tag中读取任意大小的数据的底层函数，可以被封装到`lfs_dir_get`和`lfs_dir_getread`等上层函数。开始的时候检验有没有发生移动，如果有并且移动后`tag`和给定的相同，就认为已经被移动了并返回条目缺失的错误，如果发生了移动，但是`gtag`所给的`id`更大，就把`gdiff`的`id`-1,。接下来进行向前检查，每次都将`off`减去一个当前条目的大小，然后从块中读取一个新标签（实际上是和当前标签的异或，再异或一次得到原标签），然后开始处理旧的标签，如果标签掩码不为0，且现有标签id小于等于 `(gtag - gdiff)` 的 id（即目标标签减去已累积的 splice 偏移），就把diff加上此标签id，同时若tag是创建文件标签，`id=gtag - gdiff`，也返回条目缺失。最后检查是否满足`(*gmask* & tag) == (*gmask* & (*gtag* - gdiff))`，若是且标签不是删除的，取当前标签记录中数据部分的大小`（lfs_tag_size(tag)）`与请求大小 `gsize` 的最小值，在当前目录块和当前偏移量加上指定偏移量处读此大小的数据到`gbuffer`上，然后若不能填满则把剩下的东西填充为0。最后返回值为匹配的标签加上`gdiff`。

这段函数是头一个看的云里雾里的，真没看懂在干嘛。当然写了这么一串东西，大概干什么还是知道的。阻止完全理解的部分主要在于`gdiff`对`id`的操作，我之前以为`id`只是一个标识符，只要唯一即可，但是现在看来还有偏序关系甚至数量关系，阅读之后对于`id`操作的代码之后需要回来补充这里（//TODO）。

---

### `lfs_dir_get`

```c
static lfs_stag_t lfs_dir_get(lfs_t *lfs, const lfs_mdir_t *dir,
        lfs_tag_t gmask, lfs_tag_t gtag, void *buffer) {
    return lfs_dir_getslice(lfs, dir,
            gmask, gtag,
            0, buffer, lfs_tag_size(gtag));
}
```

那么这段代码就很简单了吗，直接调用`lfs_dir_getslice`去读一个匹配tag的内容。

---

### `lfs_dir_getread`

```c
static int lfs_dir_getread(lfs_t *lfs, const lfs_mdir_t *dir,
        const lfs_cache_t *pcache, lfs_cache_t *rcache, lfs_size_t hint,
        lfs_tag_t gmask, lfs_tag_t gtag,
        lfs_off_t off, void *buffer, lfs_size_t size) {
    uint8_t *data = buffer;
    if (off+size > lfs->cfg->block_size) {
        return LFS_ERR_CORRUPT;
    }

    while (size > 0) {
        lfs_size_t diff = size;

        if (pcache && pcache->block == LFS_BLOCK_INLINE &&
                off < pcache->off + pcache->size) {
            if (off >= pcache->off) {
                // is already in pcache?
                diff = lfs_min(diff, pcache->size - (off-pcache->off));
                memcpy(data, &pcache->buffer[off-pcache->off], diff);

                data += diff;
                off += diff;
                size -= diff;
                continue;
            }

            // pcache takes priority
            diff = lfs_min(diff, pcache->off-off);
        }

        if (rcache->block == LFS_BLOCK_INLINE &&
                off < rcache->off + rcache->size) {
            if (off >= rcache->off) {
                // is already in rcache?
                diff = lfs_min(diff, rcache->size - (off-rcache->off));
                memcpy(data, &rcache->buffer[off-rcache->off], diff);

                data += diff;
                off += diff;
                size -= diff;
                continue;
            }

            // rcache takes priority
            diff = lfs_min(diff, rcache->off-off);
        }

        // load to cache, first condition can no longer fail
        rcache->block = LFS_BLOCK_INLINE;
        rcache->off = lfs_aligndown(off, lfs->cfg->read_size);
        rcache->size = lfs_min(lfs_alignup(off+hint, lfs->cfg->read_size),
                lfs->cfg->cache_size);
        int err = lfs_dir_getslice(lfs, dir, gmask, gtag,
                rcache->off, rcache->buffer, rcache->size);
        if (err < 0) {
            return err;
        }
    }

    return 0;
}
```

这里和`lfs_bd_read`的处理逻辑几乎完全一致，除了块号检验变为了检验是否为`LFS_BLOCK_INLINE`。就是套了一层缓存的`lfs_dir_getslice`。

---

### ` lfs_dir_traverse`

```c
static int lfs_dir_traverse(lfs_t *lfs,
        const lfs_mdir_t *dir, lfs_off_t off, lfs_tag_t ptag,
        const struct lfs_mattr *attrs, int attrcount,
        lfs_tag_t tmask, lfs_tag_t ttag,
        uint16_t begin, uint16_t end, int16_t diff,
        int (*cb)(void *data, lfs_tag_t tag, const void *buffer), void *data) {
    // This function in inherently recursive, but bounded. To allow tool-based
    // analysis without unnecessary code-cost we use an explicit stack
    struct lfs_dir_traverse stack[LFS_DIR_TRAVERSE_DEPTH-1];
    unsigned sp = 0;
    int res;

    // iterate over directory and attrs
    lfs_tag_t tag;
    const void *buffer;
    struct lfs_diskoff disk = {0};
    while (true) {
        {
            if (off+lfs_tag_dsize(ptag) < dir->off) {
                off += lfs_tag_dsize(ptag);
                int err = lfs_bd_read(lfs,
                        NULL, &lfs->rcache, sizeof(tag),
                        dir->pair[0], off, &tag, sizeof(tag));
                if (err) {
                    return err;
                }

                tag = (lfs_frombe32(tag) ^ ptag) | 0x80000000;
                disk.block = dir->pair[0];
                disk.off = off+sizeof(lfs_tag_t);
                buffer = &disk;
                ptag = tag;
            } else if (attrcount > 0) {
                tag = attrs[0].tag;
                buffer = attrs[0].buffer;
                attrs += 1;
                attrcount -= 1;
            } else {
                // finished traversal, pop from stack?
                res = 0;
                break;
            }

            // do we need to filter?
            lfs_tag_t mask = LFS_MKTAG(0x7ff, 0, 0);
            if ((mask & tmask & tag) != (mask & tmask & ttag)) {
                continue;
            }

            if (lfs_tag_id(tmask) != 0) {
                LFS_ASSERT(sp < LFS_DIR_TRAVERSE_DEPTH);
                // recurse, scan for duplicates, and update tag based on
                // creates/deletes
                stack[sp] = (struct lfs_dir_traverse){
                    .dir        = dir,
                    .off        = off,
                    .ptag       = ptag,
                    .attrs      = attrs,
                    .attrcount  = attrcount,
                    .tmask      = tmask,
                    .ttag       = ttag,
                    .begin      = begin,
                    .end        = end,
                    .diff       = diff,
                    .cb         = cb,
                    .data       = data,
                    .tag        = tag,
                    .buffer     = buffer,
                    .disk       = disk,
                };
                sp += 1;

                tmask = 0;
                ttag = 0;
                begin = 0;
                end = 0;
                diff = 0;
                cb = lfs_dir_traverse_filter;
                data = &stack[sp-1].tag;
                continue;
            }
        }

popped:
        // in filter range?
        if (lfs_tag_id(tmask) != 0 &&
                !(lfs_tag_id(tag) >= begin && lfs_tag_id(tag) < end)) {
            continue;
        }

        // handle special cases for mcu-side operations
        if (lfs_tag_type3(tag) == LFS_FROM_NOOP) {
            // do nothing
        } else if (lfs_tag_type3(tag) == LFS_FROM_MOVE) {
            // Without this condition, lfs_dir_traverse can exhibit an
            // extremely expensive O(n^3) of nested loops when renaming.
            // This happens because lfs_dir_traverse tries to filter tags by
            // the tags in the source directory, triggering a second
            // lfs_dir_traverse with its own filter operation.
            //
            // traverse with commit
            // '-> traverse with filter
            //     '-> traverse with move
            //         '-> traverse with filter
            //
            // However we don't actually care about filtering the second set of
            // tags, since duplicate tags have no effect when filtering.
            //
            // This check skips this unnecessary recursive filtering explicitly,
            // reducing this runtime from O(n^3) to O(n^2).
            if (cb == lfs_dir_traverse_filter) {
                continue;
            }

            // recurse into move
            stack[sp] = (struct lfs_dir_traverse){
                .dir        = dir,
                .off        = off,
                .ptag       = ptag,
                .attrs      = attrs,
                .attrcount  = attrcount,
                .tmask      = tmask,
                .ttag       = ttag,
                .begin      = begin,
                .end        = end,
                .diff       = diff,
                .cb         = cb,
                .data       = data,
                .tag        = LFS_MKTAG(LFS_FROM_NOOP, 0, 0),
            };
            sp += 1;

            uint16_t fromid = lfs_tag_size(tag);
            uint16_t toid = lfs_tag_id(tag);
            dir = buffer;
            off = 0;
            ptag = 0xffffffff;
            attrs = NULL;
            attrcount = 0;
            tmask = LFS_MKTAG(0x600, 0x3ff, 0);
            ttag = LFS_MKTAG(LFS_TYPE_STRUCT, 0, 0);
            begin = fromid;
            end = fromid+1;
            diff = toid-fromid+diff;
        } else if (lfs_tag_type3(tag) == LFS_FROM_USERATTRS) {
            for (unsigned i = 0; i < lfs_tag_size(tag); i++) {
                const struct lfs_attr *a = buffer;
                res = cb(data, LFS_MKTAG(LFS_TYPE_USERATTR + a[i].type,
                        lfs_tag_id(tag) + diff, a[i].size), a[i].buffer);
                if (res < 0) {
                    return res;
                }

                if (res) {
                    break;
                }
            }
        } else {
            res = cb(data, tag + LFS_MKTAG(0, diff, 0), buffer);
            if (res < 0) {
                return res;
            }

            if (res) {
                break;
            }
        }
    }

    if (sp > 0) {
        // pop from the stack and return, fortunately all pops share
        // a destination
        dir         = stack[sp-1].dir;
        off         = stack[sp-1].off;
        ptag        = stack[sp-1].ptag;
        attrs       = stack[sp-1].attrs;
        attrcount   = stack[sp-1].attrcount;
        tmask       = stack[sp-1].tmask;
        ttag        = stack[sp-1].ttag;
        begin       = stack[sp-1].begin;
        end         = stack[sp-1].end;
        diff        = stack[sp-1].diff;
        cb          = stack[sp-1].cb;
        data        = stack[sp-1].data;
        tag         = stack[sp-1].tag;
        buffer      = stack[sp-1].buffer;
        disk        = stack[sp-1].disk;
        sp -= 1;
        goto popped;
    } else {
        return res;
    }
}
```

