# littlefs代码分析—lfs.h

## 宏定义

```c
#define LFS_VERSION 0x0002000a
#define LFS_VERSION_MAJOR (0xffff & (LFS_VERSION >> 16))
#define LFS_VERSION_MINOR (0xffff & (LFS_VERSION >> 0))

#define LFS_DISK_VERSION 0x00020001
#define LFS_DISK_VERSION_MAJOR (0xffff & (LFS_DISK_VERSION >> 16))
#define LFS_DISK_VERSION_MINOR (0xffff & (LFS_DISK_VERSION >> 0))

#define LFS_NAME_MAX 255
 
#define LFS_FILE_MAX 2147483647
 
#define LFS_ATTR_MAX 1022
```

* `LFS_VERSION`指定littlefs版本。

* `LFS_DISK_VERSION`指定磁盘版本。磁盘版本在现阶段的littlefs的使用当中，唯一的用处就是检验是否是`0x00020001`，也就是最新版。在[littlefs release 2.6](https://github.com/littlefs-project/littlefs/releases/tag/v2.6.0)中引入了新特性[FCRC](https://github.com/littlefs-project/littlefs/releases/tag/v2.6.0)。当且仅当磁盘版本为最新版时，才会执行`FCRC`有关的操作。另外在[littlefs release 2.0](https://github.com/littlefs-project/littlefs/releases/tag/v2.0.0)中引入了内联文件，同时磁盘版本更新为2.0，但是在内联文件相关的操作上并不检查磁盘版本。

* `LFS_NAME_MAX`指定文件名默认最大长度为255字节。
* `LFS_FILE_MAX`指定文件名默认最大长度为2147483647字节。
* `LFS_ATTR_MAX`指定用户添加的属性默认最大长度为255字节。

## 别名

```c
typedef uint32_t lfs_size_t
 
typedef uint32_t lfs_off_t
 
typedef int32_t lfs_ssize_t
 
typedef int32_t lfs_soff_t
 
typedef uint32_t lfs_block_t
 
typedef struct lfs_cache lfs_cache_t
 
typedef struct lfs_mdir lfs_mdir_t
 
typedef struct lfs_dir lfs_dir_t
 
typedef struct lfs_file lfs_file_t
 
typedef struct lfs_superblock lfs_superblock_t
 
typedef struct lfs_gstate lfs_gstate_t
 
typedef struct lfs lfs_t
```

嗯...就是一些别名

## 枚举

```c
enum lfs_error {
    LFS_ERR_OK          = 0,    // No error
    LFS_ERR_IO          = -5,   // Error during device operation
    LFS_ERR_CORRUPT     = -84,  // Corrupted
    LFS_ERR_NOENT       = -2,   // No directory entry
    LFS_ERR_EXIST       = -17,  // Entry already exists
    LFS_ERR_NOTDIR      = -20,  // Entry is not a dir
    LFS_ERR_ISDIR       = -21,  // Entry is a dir
    LFS_ERR_NOTEMPTY    = -39,  // Dir is not empty
    LFS_ERR_BADF        = -9,   // Bad file number
    LFS_ERR_FBIG        = -27,  // File too large
    LFS_ERR_INVAL       = -22,  // Invalid parameter
    LFS_ERR_NOSPC       = -28,  // No space left on device
    LFS_ERR_NOMEM       = -12,  // No more memory available
    LFS_ERR_NOATTR      = -61,  // No data/attr available
    LFS_ERR_NAMETOOLONG = -36,  // File name too long
};
```

描述不同错误原因的枚举。

---

```c
enum lfs_type {
    // file types
    LFS_TYPE_REG            = 0x001,
    LFS_TYPE_DIR            = 0x002,

    // internally used types
    LFS_TYPE_SPLICE         = 0x400,
    LFS_TYPE_NAME           = 0x000,
    LFS_TYPE_STRUCT         = 0x200,
    LFS_TYPE_USERATTR       = 0x300,
    LFS_TYPE_FROM           = 0x100,
    LFS_TYPE_TAIL           = 0x600,
    LFS_TYPE_GLOBALS        = 0x700,
    LFS_TYPE_CRC            = 0x500,

    // internally used type specializations
    LFS_TYPE_CREATE         = 0x401,
    LFS_TYPE_DELETE         = 0x4ff,
    LFS_TYPE_SUPERBLOCK     = 0x0ff,
    LFS_TYPE_DIRSTRUCT      = 0x200,
    LFS_TYPE_CTZSTRUCT      = 0x202,
    LFS_TYPE_INLINESTRUCT   = 0x201,
    LFS_TYPE_SOFTTAIL       = 0x600,
    LFS_TYPE_HARDTAIL       = 0x601,
    LFS_TYPE_MOVESTATE      = 0x7ff,
    LFS_TYPE_CCRC           = 0x500,
    LFS_TYPE_FCRC           = 0x5ff,

    // internal chip sources
    LFS_FROM_NOOP           = 0x000,
    LFS_FROM_MOVE           = 0x101,
    LFS_FROM_USERATTRS      = 0x102,
};
```

在各结构体中会使用此枚举，可通过此枚举判断类型

---

```c
enum lfs_open_flags {
    // open flags
    LFS_O_RDONLY = 1,         // Open a file as read only
#ifndef LFS_READONLY
    LFS_O_WRONLY = 2,         // Open a file as write only
    LFS_O_RDWR   = 3,         // Open a file as read and write
    LFS_O_CREAT  = 0x0100,    // Create a file if it does not exist
    LFS_O_EXCL   = 0x0200,    // Fail if a file already exists
    LFS_O_TRUNC  = 0x0400,    // Truncate the existing file to zero size
    LFS_O_APPEND = 0x0800,    // Move to end of file on every write
#endif

    // internally used flags
#ifndef LFS_READONLY
    LFS_F_DIRTY   = 0x010000, // File does not match storage
    LFS_F_WRITING = 0x020000, // File has been written since last flush
#endif
    LFS_F_READING = 0x040000, // File has been read since last flush
#ifndef LFS_READONLY
    LFS_F_ERRED   = 0x080000, // An error occurred during write
#endif
    LFS_F_INLINE  = 0x100000, // Currently inlined in directory entry
};
```

描述文件打开状态的枚举

---

```c
enum lfs_whence_flags {
    LFS_SEEK_SET = 0,   // Seek relative to an absolute position
    LFS_SEEK_CUR = 1,   // Seek relative to the current file position
    LFS_SEEK_END = 2,   // Seek relative to the end of the file
};
```

描述移动文件指针基准位置的枚举

## 结构体

### `lfs_config`

```c
struct lfs_config {
    // Opaque user provided context that can be used to pass
    // information to the block device operations
    void *context;

    // Read a region in a block. Negative error codes are propagated
    // to the user.
    int (*read)(const struct lfs_config *c, lfs_block_t block,
            lfs_off_t off, void *buffer, lfs_size_t size);

    // Program a region in a block. The block must have previously
    // been erased. Negative error codes are propagated to the user.
    // May return LFS_ERR_CORRUPT if the block should be considered bad.
    int (*prog)(const struct lfs_config *c, lfs_block_t block,
            lfs_off_t off, const void *buffer, lfs_size_t size);

    // Erase a block. A block must be erased before being programmed.
    // The state of an erased block is undefined. Negative error codes
    // are propagated to the user.
    // May return LFS_ERR_CORRUPT if the block should be considered bad.
    int (*erase)(const struct lfs_config *c, lfs_block_t block);

    // Sync the state of the underlying block device. Negative error codes
    // are propagated to the user.
    int (*sync)(const struct lfs_config *c);

#ifdef LFS_THREADSAFE
    // Lock the underlying block device. Negative error codes
    // are propagated to the user.
    int (*lock)(const struct lfs_config *c);

    // Unlock the underlying block device. Negative error codes
    // are propagated to the user.
    int (*unlock)(const struct lfs_config *c);
#endif

    // Minimum size of a block read in bytes. All read operations will be a
    // multiple of this value.
    lfs_size_t read_size;

    // Minimum size of a block program in bytes. All program operations will be
    // a multiple of this value.
    lfs_size_t prog_size;

    // Size of an erasable block in bytes. This does not impact ram consumption
    // and may be larger than the physical erase size. However, non-inlined
    // files take up at minimum one block. Must be a multiple of the read and
    // program sizes.
    lfs_size_t block_size;

    // Number of erasable blocks on the device. Defaults to block_count stored
    // on disk when zero.
    lfs_size_t block_count;

    // Number of erase cycles before littlefs evicts metadata logs and moves
    // the metadata to another block. Suggested values are in the
    // range 100-1000, with large values having better performance at the cost
    // of less consistent wear distribution.
    //
    // Set to -1 to disable block-level wear-leveling.
    int32_t block_cycles;

    // Size of block caches in bytes. Each cache buffers a portion of a block in
    // RAM. The littlefs needs a read cache, a program cache, and one additional
    // cache per file. Larger caches can improve performance by storing more
    // data and reducing the number of disk accesses. Must be a multiple of the
    // read and program sizes, and a factor of the block size.
    lfs_size_t cache_size;

    // Size of the lookahead buffer in bytes. A larger lookahead buffer
    // increases the number of blocks found during an allocation pass. The
    // lookahead buffer is stored as a compact bitmap, so each byte of RAM
    // can track 8 blocks.
    lfs_size_t lookahead_size;

    // Threshold for metadata compaction during lfs_fs_gc in bytes. Metadata
    // pairs that exceed this threshold will be compacted during lfs_fs_gc.
    // Defaults to ~88% block_size when zero, though the default may change
    // in the future.
    //
    // Note this only affects lfs_fs_gc. Normal compactions still only occur
    // when full.
    //
    // Set to -1 to disable metadata compaction during lfs_fs_gc.
    lfs_size_t compact_thresh;

    // Optional statically allocated read buffer. Must be cache_size.
    // By default lfs_malloc is used to allocate this buffer.
    void *read_buffer;

    // Optional statically allocated program buffer. Must be cache_size.
    // By default lfs_malloc is used to allocate this buffer.
    void *prog_buffer;

    // Optional statically allocated lookahead buffer. Must be lookahead_size.
    // By default lfs_malloc is used to allocate this buffer.
    void *lookahead_buffer;

    // Optional upper limit on length of file names in bytes. No downside for
    // larger names except the size of the info struct which is controlled by
    // the LFS_NAME_MAX define. Defaults to LFS_NAME_MAX or name_max stored on
    // disk when zero.
    lfs_size_t name_max;

    // Optional upper limit on files in bytes. No downside for larger files
    // but must be <= LFS_FILE_MAX. Defaults to LFS_FILE_MAX or file_max stored
    // on disk when zero.
    lfs_size_t file_max;

    // Optional upper limit on custom attributes in bytes. No downside for
    // larger attributes size but must be <= LFS_ATTR_MAX. Defaults to
    // LFS_ATTR_MAX or attr_max stored on disk when zero.
    lfs_size_t attr_max;

    // Optional upper limit on total space given to metadata pairs in bytes. On
    // devices with large blocks (e.g. 128kB) setting this to a low size (2-8kB)
    // can help bound the metadata compaction time. Must be <= block_size.
    // Defaults to block_size when zero.
    lfs_size_t metadata_max;

    // Optional upper limit on inlined files in bytes. Inlined files live in
    // metadata and decrease storage requirements, but may be limited to
    // improve metadata-related performance. Must be <= cache_size, <=
    // attr_max, and <= block_size/8. Defaults to the largest possible
    // inline_max when zero.
    //
    // Set to -1 to disable inlined files.
    lfs_size_t inline_max;

#ifdef LFS_MULTIVERSION
    // On-disk version to use when writing in the form of 16-bit major version
    // + 16-bit minor version. This limiting metadata to what is supported by
    // older minor versions. Note that some features will be lost. Defaults to 
    // to the most recent minor version when zero.
    uint32_t disk_version;
#endif
};
```

用于初始化`littlefs`的配置结构体，少见的注释详尽。每个成员的作用注释都写的非常清楚了。

---

### `lfs_info`

```c
struct lfs_info {
    // Type of the file, either LFS_TYPE_REG or LFS_TYPE_DIR
    uint8_t type;

    // Size of the file, only valid for REG files. Limited to 32-bits.
    lfs_size_t size;

    // Name of the file stored as a null-terminated string. Limited to
    // LFS_NAME_MAX+1, which can be changed by redefining LFS_NAME_MAX to
    // reduce RAM. LFS_NAME_MAX is stored in superblock and must be
    // respected by other littlefs drivers.
    char name[LFS_NAME_MAX+1];
};
```

用于记录文件信息的结构体，包括区别文件是否目录，文件大小和文件（目录）名。

---

### `lfs_fsinfo`

```c
struct lfs_fsinfo {
    // On-disk version.
    uint32_t disk_version;

    // Size of a logical block in bytes.
    lfs_size_t block_size;

    // Number of logical blocks in filesystem.
    lfs_size_t block_count;

    // Upper limit on the length of file names in bytes.
    lfs_size_t name_max;

    // Upper limit on the size of files in bytes.
    lfs_size_t file_max;

    // Upper limit on the size of custom attributes in bytes.
    lfs_size_t attr_max;
};
```

用于记录文件系统信息的结构体，基本上是`lfs_config`的子集。

---

### `lfs_attr`

```c
struct lfs_attr {
    // 8-bit type of attribute, provided by user and used to
    // identify the attribute
    uint8_t type;

    // Pointer to buffer containing the attribute
    void *buffer;

    // Size of attribute in bytes, limited to LFS_ATTR_MAX
    lfs_size_t size;
};
```

用于记录用户提供的文件属性的信息，包括类型，属性内容的指针和相应的字节数。

---

### `lfs_file_config`

```c
struct lfs_file_config {
    // Optional statically allocated file buffer. Must be cache_size.
    // By default lfs_malloc is used to allocate this buffer.
    void *buffer;

    // Optional list of custom attributes related to the file. If the file
    // is opened with read access, these attributes will be read from disk
    // during the open call. If the file is opened with write access, the
    // attributes will be written to disk every file sync or close. This
    // write occurs atomically with update to the file's contents.
    //
    // Custom attributes are uniquely identified by an 8-bit type and limited
    // to LFS_ATTR_MAX bytes. When read, if the stored attribute is smaller
    // than the buffer, it will be padded with zeros. If the stored attribute
    // is larger, then it will be silently truncated. If the attribute is not
    // found, it will be created implicitly.
    struct lfs_attr *attrs;

    // Number of custom attributes in the list
    lfs_size_t attr_count;
};

```

用于记录文件配置的信息，包括缓存文件的指针，文件的属性列表和数目。

---

### `lfs_cache`

```c
struct lfs_cache {
    lfs_block_t block;
    lfs_off_t off;
    lfs_size_t size;
    uint8_t *buffer;
}
```

存储缓冲区指针，以及缓冲区内容对应的磁盘位置。

---

### `lfs_mdir`

```c
struct lfs_mdir {
    lfs_block_t pair[2];
    uint32_t rev;
    lfs_off_t off;
    uint32_t etag;
    uint16_t count;
    bool erased;
    bool split;
    lfs_block_t tail[2];
}
```

低垂的果子已经摘完了，这里开始就缺乏注释并且开始需要花时间理解。

用于记录目录元数据的结构体

* `lfs_block_t pair[2]`:存储元数据对所在的2个块的位置
* `rev`：*revision*，用于区分块的版本，选择最新的版本作为有效块
* `off`: 目录块中当前的写入偏移量
* `etag`:目录项的entry tag
* `count`:有效条目的数量
* `erased`:标识当前目录块是否已被擦除，确保写操作的安全性
* `split`:标识当前目录是否发生了分裂
* `tail`: TODO, 尚需理解尾块是干嘛的
