## Nginx高级数据结构
Nginx中主要现实了5个高级容器，熟练使用这些容器可以大大提高开发Nginx模块的效率。其主要包括ngx_queue_t双向链表、ngx_array_t动态数组、ngx_list_t单向链表、ngx_rbtree_t红黑树和支持通配符的散列表。

### ngx_queue_t双向链表

实现文件：src/core/ngx_queue.h/.c 在 Nginx 的队列实现中，实质就是具有头节点的双向循环链表，这里的双向链表中的节点是没有数据区的，只有两个指向节点的指针。需注意的是队列链表的内存分配不是直接从内存池分配的，即没有进行内存池管理，而是需要我们自己管理内存，所有我们可以指定它在内存池管理或者直接在堆里面进行管理，最好使用内存池进行管理。

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

  链表排序：队列链表排序采用的是稳定的简单插入排序方法，即从第一个节点开始遍历，依次将当前节点(q)插入前面已经排好序的队列(链表)中。

  获取队列中节点数据地址：由队列基本结构和以上操作可知，nginx 的队列操作只对链表指针进行简单的修改指向操作，并不负责节点数据空间的分配。因此，用户在使用nginx队列时，要自己定义数据结构并分配空间，且在其中包含一个 ngx_queue_t 的指针或者对象，当需要获取队列节点数据时，使用ngx_queue_data宏。

  在 Nginx 的队列链表中，其维护的是指向链表节点的指针，并没有实际的数据区，所有对实际数据的操作需要我们自行操作，队列链表实质是双向循环链表，其操作是双向链表的基本操作。其优势在于实现了排序，比较轻量级，支持两个链表的合并。

### ngx_array_t数组结构

实现文件：src/core/ngx_array.h/.c ngx_array_t的设计与C++中的STL的设计相似，其内存分配是基于内存池的，并不是一成不变的，若内存不足时，按照当前数组的两倍内存大小进行申请，这样可以减少内存分配的次数。

```cpp
    typedef struct {
      void        *elts;  /* 指向数组数据区域的首地址 */
      ngx_uint_t   nelts; /* 数组实际数据的个数 */
      size_t       size;  /* 单个元素所占据的字节大小 */
      ngx_uint_t   nalloc;/* 数组容量 */
      ngx_pool_t  *pool;  /* 数组对象所在的内存池 */
    } ngx_array_t;

  ```

  基本操作
  ```cpp
    /* 创建新的动态数组 */
    ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
    /* 销毁数组对象，内存被内存池回收 */
    void ngx_array_destroy(ngx_array_t *a);
    /* 在现有数组中增加一个新的元素 */
    void *ngx_array_push(ngx_array_t *a);
    /* 在现有数组中增加 n 个新的元素 */
    void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);

    /* 当一个数组对象被分配在堆上，且调用ngx_array_destroy之后，若想重新使用，则需调用该函数 */
    /* 若数组对象被分配在栈上，则需调用此函数 */
    static ngx_inline ngx_int_t
    ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
    {
      /*
       * set "array->nelts" before "array->elts", otherwise MSVC thinks
       * that "array->nelts" may be used without having been initialized
       */

       /* 初始化数组成员，注意：nelts必须比elts先初始化 */
       array->nelts = 0;
       array->size = size;
       array->nalloc = n;
       array->pool = pool;

       /* 分配数组数据域所需要的内存 */
       array->elts = ngx_palloc(pool, n * size);
       if (array->elts == NULL) {
         return NGX_ERROR;
       }

       return NGX_OK;
     }
  ```
  由于存在ngx_array_push_n操作，其扩容操作会稍微复杂一点，其会根据插入的数据n的个数与原数据中节点个数的大小关系来进行扩容，若n小于原始空间，则扩容增加为原始两倍，若大于，则扩容为2n。

### ngx_list_t链表结构

实现文件：src/core/ngx_list.h/.c ngx_list_t 是 Nginx 封装的链表容器，链表容器内存分配是基于内存池进行的，操作方便，效率高。Nginx链表容器和普通链表类似，均有链表表头和链表节点，通过节点指针组成链表
  ```cpp
    /* 链表结构 */
    typedef struct ngx_list_part_s  ngx_list_part_t;

    /* 链表中的节点结构 */
    struct ngx_list_part_s {
      void             *elts; /* 指向该节点数据区的首地址 */
      ngx_uint_t        nelts;/* 该节点数据区实际存放的元素个数 */
      ngx_list_part_t  *next; /* 指向链表的下一个节点 */
    };

    /* 链表表头结构 */
    typedef struct {
      ngx_list_part_t  *last; /* 指向链表中最后一个节点 */
      ngx_list_part_t   part; /* 链表中表头包含的第一个节点 */
      size_t            size; /* 元素的字节大小 */
      ngx_uint_t        nalloc;/* 链表中每个节点所能容纳元素的个数 */
      ngx_pool_t       *pool; /* 该链表节点空间的内存池对象 */
    } ngx_list_t;
  ```
  链表结构示意图：

  <div align=center><img src="https://github.com/wangsen1994/nginx-Source-reading/blob/master/datum/链表结构示意图.png"/></div>

  基本操作
  ```cpp
    /* 创建新的链表 */
    ngx_list_t * ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size)
    /* 初始化链表 */
    static ngx_inline ngx_int_t
    ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size)
    /* 添加元素 */
    void * ngx_list_push(ngx_list_t *l)
  ```
  ngx_list_t链表的节点是可以存储多个元素的数组，若内存不足则可以每次扩容1个数组

  ### ngx_hash_t哈希表结构

  实现文件：src/core/ngx_hash.h/.c 。哈希表结合了数组和链表的特点，使其寻址、插入以及删除操作更加方便。哈希表的过程是将关键字通过某种哈希函数映射到相应的哈希表位置，即对应的哈希值所在哈希表的位置。但是会出现多个关键字映射相同位置的情况导致冲突问题，Nginx使用开放寻址来解决冲突问题，为了处理字符串，Nginx 还实现了支持通配符操作的相关函数。

  哈希表中关键字元素的结构 ngx_hash_elt_t，哈希表元素结构采用 键-值 形式，即<key，value>,其定义如下：
  ```cpp
      /* hash散列表中元素的结构，采用键值及其所以应的值<key，value>*/
      typedef struct {
        void             *value;    /* 指向用户自定义的数据 */
        u_short           len;      /* 键值key的长度 */
        u_char            name[1];  /* 键值key的第一个字符，数组名name表示指向键值key首地址 */
      } ngx_hash_elt_t;
  ```
  哈希表基本结构 ngx_hash_t，其结构定义如下：
  ```cpp
    /* 基本hash散列表结构 */
    typedef struct {
      ngx_hash_elt_t  **buckets;  /* 指向hash散列表第一个存储元素的桶 */
      ngx_uint_t        size;     /* hash散列表的桶个数 */
    } ngx_hash_t;
  ```
  哈希初始化结构 ngx_hash_init_t
  ```cpp
   typedef ngx_uint_t (*ngx_hash_key_pt) (u_char *data, size_t len);

   /* 初始化hash结构 */
   typedef struct {
     ngx_hash_t       *hash;         /* 指向待初始化的基本hash结构 */
     ngx_hash_key_pt   key;          /* hash 函数指针 */

     ngx_uint_t        max_size;     /* hash表中桶bucket的最大个数 */
     ngx_uint_t        bucket_size;  /* 每个桶bucket的存储空间 */

     char             *name;         /* hash结构的名称(仅在错误日志中使用) */
     ngx_pool_t       *pool;         /* 分配hash结构的内存池 */
     /* 分配临时数据空间的内存池，仅在初始化hash表前，用于分配一些临时数组 */
     ngx_pool_t       *temp_pool;
   } ngx_hash_init_t;
  ```
  哈希元素数据 ngx_hash_key_t
  ```cpp
    /* 计算待添加元素的hash元素结构 */
    typedef struct {
      ngx_str_t         key;      /* 元素关键字 */
      ngx_uint_t        key_hash; /* 元素关键字key计算出的hash值 */
      void             *value;    /* 指向关键字key对应的值，组成hash表元素：键-值<key，value> */
    } ngx_hash_key_t;
  ```
  哈希操作

  哈希操作包括初始化函数、查找函数；其中初始化函数是 Nginx 中哈希表比较重要的函数，由于 Nginx 的 hash 表是静态只读的，即不能在运行时动态添加新元素的，一切的结构和数据都在配置初始化的时候就已经规划完毕。

  * 哈希函数
    ```cpp
    #define ngx_hash(key, c)   ((ngx_uint_t) key * 31 + c)
    ngx_uint_t ngx_hash_key(u_char *data, size_t len); /* hash函数 */
    ngx_uint_t ngx_hash_key_lc(u_char *data, size_t len); /* 转换成小写，进行hash */
    ngx_uint_t ngx_hash_strlow(u_char *dst, u_char *src, size_t n); /* 转换前n个为小写，进行hash */
    ```
  * 初始化

  ```cpp
  /* 计算ngx_hash_elt_t的内存大小，并进行对齐操作 */
  #define NGX_HASH_ELT_SIZE(name)                                               \
    (sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))


  ngx_int_t
  ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
  {
      u_char          *elts;
      size_t           len;
      u_short         *test;
      ngx_uint_t       i, n, key, size, start, bucket_size;
      ngx_hash_elt_t  *elt, **buckets;

      if (hinit->max_size == 0) {
          ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                      "could not build %s, you should "
                      "increase %s_max_size: %i",
                      hinit->name, hinit->name, hinit->max_size);
        return NGX_ERROR;
      }
      for (n = 0; n < nelts; n++) {
		      /* 若每个桶bucket的内存空间不足以存储一个关键字元素，则出错返回
      		* 这里考虑到了每个bucket桶最后的null指针所需的空间，即该语句中的sizeof(void *)，
      		* 该指针可作为查找过程中的结束标记
		      */
          if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *))
          {
              ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_bucket_size: %i",
                          hinit->name, hinit->name, hinit->bucket_size);
                          return NGX_ERROR;
         }
      }
      //下面这个test的用处多多. 它的大小是max_size, 是允许的bucket的最大个数.
	    //它的大小就表明, 以后使用这个test数组, 它与bucket是一一对应的. 即bucket[i]与test[i]是相关的
      test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
      if (test == NULL) {
          return NGX_ERROR;
        }
      bucket_size = hinit->bucket_size - sizeof(void *);  // 减去bucket最后的null空间，哨兵元素用来判断是否还有元素

      /* 估算需要的最少的bucket数量 根据地址对齐, 一个ngx_hash_elt_t元素最少也要2*8个字节*/
      start = nelts / (bucket_size / (2 * sizeof(void *)));
      start = start ? start : 1;

      /* 无法理解 */
      if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        start = hinit->max_size - 1000;
      }

      for (size = start; size <= hinit->max_size; size++) {

        ngx_memzero(test, size * sizeof(u_short));

        for (n = 0; n < nelts; n++) {
            if (names[n].key.data == NULL) {
                continue;
            }
			  /* 根据关键字元素的hash值计算存在到测试数组test对应的位置中，即计算bucket在hash表中的编号key,key取值为0～size-1 */
            key = names[n].key_hash % size;
			  /* 累加要被存放在bucket[key]的内存占用大小 */
            test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));

            if (test[key] > (u_short) bucket_size) {
                goto next;
            }
        }
		   /* 若size个bucket桶可以容纳name数组的所有关键字元素，则表示找到合适的bucket数量大小即为size */
        goto found;

      next:
          continue;
        }
      size = hinit->max_size;

      ngx_log_error(NGX_LOG_WARN, hinit->pool->log, 0,
                  "could not build optimal %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i; "
                  "ignoring %s_bucket_size",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size, hinit->name);

    found:

      //这里表明我们已经成功找到了满足条件的size大小.
	    //依旧是test[i]对应bucket[i], 现在我们要求出所有元素总共需要多少内存. 最后申请出这块内存, 并一一分配给每个bucket
	    //首先, 我们算出每个bucket要存放的内存容量, 记录在test数组中
	    //此时NULL指针就需要算上了
      for (i = 0; i < size; i++) {
          test[i] = sizeof(void *);   // 加上NULL指针大小
        }

     for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));  // 算出每个bucket的大小
        }

     len = 0;

     for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }

        test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        len += test[i];  // 算出总长度
     }

     if (hinit->hash == NULL) {

		//值得注意的是, 这里申请的并不是单纯的基本哈希表结构的内存, 而是包含基本哈希表的通配符哈希表.
		//之所以这样设计, 我认为是为了满足将来可能要init通配符哈希表的需求. 既然ngx_hash_wildcard_t中包含基本哈希表, 且使用起来并没有任何麻烦,
		//那么这样是不是要显得更好呢？
        hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        if (hinit->hash == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }
		    //一开始就定义的二级指针bucket, 指向哈希表中 ngx_hash_elt_t * 构成的数组. 即bucket构成的数组
         buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

       } else {
           buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
           if (buckets == NULL) {
              ngx_free(test);
              return NGX_ERROR;
         }
     }

     elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    if (elts == NULL) {
        ngx_free(test);
        return NGX_ERROR;
    }

    elts = ngx_align_ptr(elts, ngx_cacheline_size);
	//下面就为每个bucket分配内存. 之前已经用test记录了每个bucket应该得到的内存大小.
    for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }

        buckets[i] = (ngx_hash_elt_t *) elts;
        elts += test[i];

    }
	//既然每个bucket拥有了它应该有的内存, 那么现在就将key-value数据搬进去
	//现在依旧是test[i]对应bucket[i]. 此时的test数组用于记录当前的某bucket已经有多少内存被初始化了.
	//如果这个元素已经搬到这个bucket中, 下一个元素首地址就是从当前元素首地址加上test[i]开始.
    for (i = 0; i < size; i++) {
        test[i] = 0;
    }

    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        elt->value = names[n].value;
        elt->len = (u_short) names[n].key.len;

		//复制的同时, 将大写字母改为小写
        ngx_strlow(elt->name, names[n].key.data, names[n].key.len);

        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    }

	/* 每个bucket后面加上NULL */
    for (i = 0; i < size; i++) {
        if (buckets[i] == NULL) {
            continue;
        }

        elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);

        elt->value = NULL;
    }

    ngx_free(test);

    hinit->hash->buckets = buckets;
    hinit->hash->size = size;

    return NGX_OK;
  }
  ```
