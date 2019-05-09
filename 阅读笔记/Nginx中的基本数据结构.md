## Nginx基本数据结构

Nginx中基本数据结构主要包括整形类型、字符串类型、内存池类型、缓冲区类型。

### 整形类型

  ```cpp
  /* Nginx 整数数据类型 */
  /* 在文件 src/core/ngx_config.h 定义了基本的数据映射 */
  typedef intptr_t        ngx_int_t;
  typedef uintptr_t       ngx_uint_t;
  typedef intptr_t        ngx_flag_t;
  /* 其中 intptr_t uintptr_t 定义在文件 /usr/include/stdint.h 文件中*/
  ```
### 字符串类型

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
### 内存池类型

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
### 缓冲区类型

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
### 缓冲区链表类型
  ngx_chain_t 数据类型是与缓冲区类型 ngx_buf_t 相关的链表结构。

  ```cpp
  struct ngx_chain_s {
    ngx_buf_t    *buf;  /* 指向当前缓冲区 */
    ngx_chain_t  *next; /* 指向下一个chain，形成chain链表 */
  };
  ```
需要注意的是，Nginx为了节约内存开销，其只有ngx_pool_t会真实的保存数据而ngx_chain_s和ngx_buf_t都只是利用指针指向某个位置来实现保存数据，所以对于数据处理需谨慎。
