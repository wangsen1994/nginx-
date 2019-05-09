## Nginx高级数据结构
Nginx中主要现实了6个高级容器，熟练使用这些容器可以大大提高开发Nginx模块的效率。其主要包括ngx_queue_t双向链表、ngx_array_t动态数组、ngx_list_t单向链表、ngx_rbtree_t红黑树、ngx_radix_tree_t基数树和支持通配符的散列表。

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
