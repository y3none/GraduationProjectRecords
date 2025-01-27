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

## 函数

激动人心的时刻正式开始！到这里为止，前面的技术文档总计约六千多词，这一段可能就要两万词以上，一百多个函数，开始吧！

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
