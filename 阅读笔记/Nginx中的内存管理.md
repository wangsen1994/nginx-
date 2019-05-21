## Nginx内存池管理

Nginx把内存分配归结为大内存分配与小内存分配，若申请的内存大小比同页的内存池最大值max还大，则采用大内存分配。

* 大内存分配：直接向系统申请一块内存，然后将其挂到内存池头部的large字段
* 小块内存分配，则是从已有的内存池数据区中分配

### 内存管理函数：
* ngx_alloc：封装malloc分配内存
* ngx_calloc：封装malloc分配内存，并初始化空间内容为0
* ngx_memalign：返回基于一个指定 alignment 的大小为 size 的内存空间，且其地址为 alignment 的整数倍，alignment 为2的幂

内存分配流程：

<div align=center><img width = '80%' src="F:\graduate\github\nginx-Source-reading\datum\nginx内存分配.png"/></div>

### 内存池基本数据结构

```cpp
/* 内存池结构 */
/* 文件 core/ngx_palloc.h */
typedef struct {/* 内存池数据结构模块 */
    u_char               *last; /* 当前内存分配的结束位置，即下一段可分配内存的起始位置 */
    u_char               *end;  /* 内存池的结束位置 */
    ngx_pool_t           *next; /* 指向下一个内存池 */
    ngx_uint_t            failed;/* 记录内存池内存分配失败的次数 */
} ngx_pool_data_t;  /* 维护内存池的数据块 */

struct ngx_pool_s {/* 内存池的管理模块，即内存池头部结构 */
    ngx_pool_data_t       d;    /* 内存池的数据块 */
    size_t                max;  /* 内存池数据块的最大值 */
    ngx_pool_t           *current;/* 指向当前内存池 */
    ngx_chain_t          *chain;/* 指向一个 ngx_chain_t 结构 */
    ngx_pool_large_t     *large;/* 大块内存链表，即分配空间超过 max 的内存 */
    ngx_pool_cleanup_t   *cleanup;/* 析构函数，释放内存池 */
    ngx_log_t            *log;/* 内存分配相关的日志信息 */
};
/* 文件 core/ngx_core.h */
typedef struct ngx_pool_s   ngx_pool_t;
typedef struct ngx_chain_s  ngx_chain_t;
```
大内存块数据结构
```cpp
typedef struct ngx_pool_large_s ngx_pool_large_t;

struct ngx_pool_large_s{
          ngx_pool_large_t  *next;    //指向下一块大块内存
          void    *alloc;             //指向分配的大块内存
};
```
其他数据结构
```cpp
typedef void (*ngx_pool_cleanup_pt)(void *data);    //cleanup的callback类型

typedef struct ngx_pool_cleanup_s ngx_pool_cleanup_t;

struct ngx_pool_cleanup_s{
    ngx_pool_cleanup_pt handler;
    void    *data;              //指向要清除的数据
    ngx_pool_cleanup_t *next;   //下一个cleanup callback
};

typedef struct {
    ngx_fd_t   fd;
    u_char    *name;
    ngx_log_t *log;
} ngx_pool_cleanup_file_t;
```

内存池基本机构之间的关系如下图所示:

<div align=center><img width = '80%' src="F:\graduate\github\nginx-Source-reading\datum\内存数据结构.png"/></div>
---
ngx_pool_t 的逻辑结构

<div align=center><img width = '80%' src="F:\graduate\github\nginx-Source-reading\datum\内存池结构.png"/></div>

内存对齐操作
```cpp
#define ngx_align_ptr(p,a) (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
```
