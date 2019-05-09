## Nginx高级数据结构
Nginx中主要现实了6个高级容器，熟练使用这些容器可以大大提高开发Nginx模块的效率。其主要包括ngx_queue_t双向链表、ngx_array_t动态数组、ngx_list_t单向链表、ngx_rbtree_t红黑树、ngx_radix_tree_t基数树和支持通配符的散列表。

### 基本数据结构
* 整形类型

  ```cpp
  /* Nginx 整数数据类型 */
  /* 在文件 src/core/ngx_config.h 定义了基本的数据映射 */
  typedef intptr_t        ngx_int_t;
  typedef uintptr_t       ngx_uint_t;
  typedef intptr_t        ngx_flag_t;
  /* 其中 intptr_t uintptr_t 定义在文件 /usr/include/stdint.h 文件中*/
  ```
* 字符串类型

  需要注意的是Nginx中ngx_str_t对原始C语言中的字符串进行了改造，去掉了字符串尾部的'\0'符号，加入了长度len字段，所以进行有关ngx_str_t的操作时，应尽量使用Nginx自身定义的字符串函数，若需使用C字符串函数则需加上'\0'，Nginx这样的实现方式可以减少内存的消耗，若nginx需要重复引用一段字符串内存，data可以指向任意内存，长度表示结束，而不用去copy一份自己的字符串(因为如果要以’\0’结束，而不能更改原字符串，所以势必要copy一段字符串)。
  ```cpp
  /* Nginx 字符串数据类型 */
  /* Nginx 字符串类型是对 C 语言字符串类型的简单封装*/
  typedef struct {
      size_t      len;    /* 字符串的长度 */
      u_char     *data;   /* 指向字符串的第一个字符 */
  } ngx_str_t;

  typedef struct {
      ngx_str_t   key;
      ngx_str_t   value;
  } ngx_keyval_t;

  typedef struct {
      unsigned    len:28;

      unsigned    valid:1;
      unsigned    no_cacheable:1;
      unsigned    not_found:1;
      unsigned    escape:1;

      u_char     *data;
  } ngx_variable_value_t;
  ```
* 内存池类型

  Nginx中的具体内存管理方式查看[Nginx中的内存管理]（）
  ```cpp
  /* 内存池结构 */
  /* 文件 core/ngx_palloc.h */
  typedef struct {/* 内存池数据结构模块 */
      u_char               *last; /* 当前内存分配的结束位置，即下一段可分配内存的起始位置 */
      u_char               *end;  /* 内存池的结束位置 */
      ngx_pool_t           *next; /* 指向下一个内存池 */
      ngx_uint_t            failed;/* 记录内存池内存分配失败的次数 */
  }   ngx_pool_data_t;  /* 维护内存池的数据块 */

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
* 缓冲区类型

  ```cpp
  /* 缓冲区结构 */
  typedef void *            ngx_buf_tag_t;

  typedef struct ngx_buf_s  ngx_buf_t;

  struct ngx_buf_s {
      u_char          *pos;   /* 缓冲区数据在内存的起始位置 */
      u_char          *last;  /* 缓冲区数据在内存的结束位置 */
      /* 这两个参数是处理文件时使用，类似于缓冲区的pos, last */
      off_t            file_pos;
      off_t            file_last;

      /* 由于实际数据可能被包含在多个缓冲区中，则缓冲区的start和end指
      向这块内存的开始地址和结束地址，而pos和last是指向本缓冲区实际包含
      的数据的开始和结尾
     */
     u_char          *start;         /* start of buffer */
     u_char          *end;           /* end of buffer */
     ngx_buf_tag_t    tag;
     ngx_file_t      *file;          /* 指向buffer对应的文件对象 */
     /*
     * 当前缓冲区的影子缓冲区，该成员很少用到。当缓冲区转发上游服务器响
     应时才使用了shadow成员，这是因为nginx太节约内存了，分配一块内存并
     使用ngx_buf_t表示接收到的上游服务器响应后，在向下游客户端转发时可
     能会把这块内存存储到文件中，也可能直接向下游发送，此时nginx绝对不
     会重新复制一份内存用于新的目的，而是再次建立一个ngx_buf_t结构体指
     向原内存，这样多个ngx_buf_t结构体指向了同一份内存，它们之间的关系
     就通过shadow成员来引用，一般不建议使用。
     */
     ngx_buf_t       *shadow;

     /* 为1时，表示该buf所包含的内容在用户创建的内存块中
      * 可以被filter处理变更
      */
      /* the buf's content could be changed */
      unsigned         temporary:1;

      /* 为1时，表示该buf所包含的内容在内存中，不能被filter处理变更 */
      /*
       * the buf's content is in a memory cache or in a read only memory and must not be changed
     */
     unsigned         memory:1;

     /* 为1时，表示该buf所包含的内容在内存中，
      * 可通过mmap把文件映射到内存中，不能被filter处理变更 */
     /* the buf's content is mmap()ed and must not be changed */
     unsigned         mmap:1;

     /* 可回收，即这些buf可被释放 */
     unsigned         recycled:1;
     unsigned         in_file:1; /* 表示buf所包含的内容在文件中 */
     unsigned         flush:1;   /* 刷新缓冲区 */
     unsigned         sync:1;    /* 同步方式 */
     unsigned         last_buf:1;/* 当前待处理的是最后一块缓冲区 */
     unsigned         last_in_chain:1;/* 在当前的chain里面，该buf是最后一个，但不一定是last_buf */

     unsigned         last_shadow:1;
     unsigned         temp_file:1;

     /* STUB */ int   num;
   };
  ```
  ngx_chain_t 数据类型是与缓冲区类型 ngx_buf_t 相关的链表结构。

  ```cpp
  struct ngx_chain_s {
    ngx_buf_t    *buf;  /* 指向当前缓冲区 */
    ngx_chain_t  *next; /* 指向下一个chain，形成chain链表 */
  };
  ```
  需要注意的是，Nginx为了节约内存开销，其只有ngx_pool_t会真实的保存数据而ngx_chain_s和ngx_buf_t都只是利用指针指向某个位置来实现保存数据，所以对于数据处理需谨慎。

### ngx_queue_t双向链表

实现文件：src/core/ngx_queue.h/.c 在 Nginx 的队列实现中，实质就是具有头节点的双向循
环链表，这里的双向链表中的节点是没有数据区的，只有两个指向节点的指针。需注意的是队列链
表的内存分配不是直接从内存池分配的，即没有进行内存池管理，而是需要我们自己管理内存，所
有我们可以指定它在内存池管理或者直接在堆里面进行管理，最好使用内存池进行管理。

```cpp
  /* 队列结构，其实质是具有有头节点的双向循环链表 */
  typedef struct ngx_queue_s  ngx_queue_t;

  /* 队列中每个节点结构，只有两个指针，并没有数据区 */
  struct ngx_queue_s {
      ngx_queue_t  *prev;
      ngx_queue_t  *next;
    };
```

  列队链表的基本（这里不说明实现，具体可以查看源码）
```cppp
  /* h 为链表结构体 ngx_queue_t 的指针；初始化双链表 */
  ngx_queue_int(h)

  /* h 为链表容器结构体 ngx_queue_t 的指针； 判断链表是否为空 */
  ngx_queue_empty(h)

  /* h 为链表容器结构体 ngx_queue_t 的指针，x 为插入元素结构体中 ngx_queue_t 成员的
  指针；将 x 插入到链表头部 */
  ngx_queue_insert_head(h, x)

  /* h 为链表容器结构体 ngx_queue_t 的指针，x 为插入元素结构体中 ngx_queue_t 成员的
  指针。将 x 插入到链表尾部 */
  ngx_queue_insert_tail(h, x)

  /* h 为链表容器结构体 ngx_queue_t 的指针。返回链表容器 h 中的第一个元素的
  ngx_queue_t 结构体指针 */
  ngx_queue_head(h)

  /* h 为链表容器结构体 ngx_queue_t 的指针。返回链表容器 h 中的最后一个元素的
  ngx_queue_t 结构体指针 */
  ngx_queue_last(h)

  /* h 为链表容器结构体 ngx_queue_t 的指针。返回链表结构体的指针 */
  ngx_queue_sentinel(h)

  /* x 为链表容器结构体 ngx_queue_t 的指针。从容器中移除 x 元素 */
  ngx_queue_remove(x)

  /* h 为链表容器结构体 ngx_queue_t 的指针。该函数用于拆分链表，
   * h 是链表容器，而 q 是链表 h 中的一个元素。
   * 将链表 h 以元素 q 为界拆分成两个链表 h 和 n
  */
  ngx_queue_split(h, q, n)

 /* h 为链表容器结构体 ngx_queue_t 的指针， n为另一个链表容器结构体 ngx_queue_t的指
 针合并链表，将 n 链表添加到 h 链表的末尾
 */
 ngx_queue_add(h, n)

 /* h 为链表容器结构体 ngx_queue_t 的指针。返回链表中心元素，即第 N/2 + 1 个 */
 ngx_queue_middle(h)

 /* h 为链表容器结构体 ngx_queue_t 的指针，cmpfunc 是比较回调函数。使用插入排序对链
 表进行排序 */
 ngx_queue_sort(h, cmpfunc)

 /* q 为链表中某一个元素结构体的 ngx_queue_t 成员的指针。返回 q 元素的下一个元素。*/
 ngx_queue_next(q)

 /* q 为链表中某一个元素结构体的 ngx_queue_t 成员的指针。返回 q 元素的上一个元素。*/
 ngx_queue_prev(q)

 /* q 为链表中某一个元素结构体的 ngx_queue_t 成员的指针，type 是链表元素的结构体类型名称，
  * link 是上面这个结构体中 ngx_queue_t 类型的成员名字。返回 q 元素所属结构体的地址
  */
  ngx_queue_data(q, type, link)

  /* q 为链表中某一个元素结构体的 ngx_queue_t 成员的指针，x 为插入元素结构体中
  ngx_queue_t 成员的指针 */
  ngx_queue_insert_after(q, x)
```

  链表排序：队列链表排序采用的是稳定的简单插入排序方法，即从第一个节点开始遍历，依次将
  当前节点(q)插入前面已经排好序的队列(链表)中。

  获取队列中节点数据地址：由队列基本结构和以上操作可知，nginx 的队列操作只对链表指针
  进行简单的修改指向操作，并不负责节点数据空间的分配。因此，用户在使用nginx队列时，要
  自己定义数据结构并分配空间，且在其中包含一个 ngx_queue_t 的指针或者对象，当需要获取
  队列节点数据时，使用ngx_queue_data宏。

  在 Nginx 的队列链表中，其维护的是指向链表节点的指针，并没有实际的数据区，所有对实际
  数据的操作需要我们自行操作，队列链表实质是双向循环链表，其操作是双向链表的基本操作。
  其优势在于实现了排序，比较轻量级，支持两个链表的合并。
