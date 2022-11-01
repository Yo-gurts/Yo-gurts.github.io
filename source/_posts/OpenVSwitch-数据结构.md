---
title: OpenVSwitch-数据结构
top_img: transparent
date: 2022-11-01 14:34:27
updated: 2022-11-01 14:34:27
tags:
  - OpenVSwitch
categories: OpenVSwitch
keywords:
description: OVS中常用的数据结构分析
---

## 常用宏

> include/openvswitch/util.h #119

**typeof**：typeof不是C语言本身的关键词或运算符，它是GCC的一个扩展，作用正如其字面意思，用某种已有东西（变量、函数等）的类型去定义新的变量类型。

```c
typeof(int *)a, b;     // 等价于 int *a, *b;
typeof(a) c;           // 等价于 int *c

#define OVS_TYPEOF(OBJECT) typeof(OBJECT)
```

**offsetof**：获取结构体`TYPE`中某个成员`MEMBER`的偏移量（相对于结构体地址）。

```c
/* /usr/lib/gcc/x86_64-linux-gnu/9/include/stddef.h */
#define offsetof(TYPE, MEMBER) __builtin_offsetof (TYPE, MEMBER)
/* 类似于 */
#define offsetof(TYPE, MEMBER)    ((size_t)&((TYPE *)0)->MEMBER)

/* 使用方法 */
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    struct s {
        int i;
        char c;
        double d;
        char a[];
    };

    /* Output is compiler dependent */
    printf("offsets: i=%zd; c=%zd; d=%zd a=%zd\n",
            offsetof(struct s, i), offsetof(struct s, c),
            offsetof(struct s, d), offsetof(struct s, a));
    printf("sizeof(struct s)=%zd\n", sizeof(struct s));
    exit(EXIT_SUCCESS);
}
```

### OBJECT_OFFSETOF

给定一个指向结构体的指针`OBJECT`，返回其成员`MEMBER`的偏移量。

```c
#define OBJECT_OFFSETOF(OBJECT, MEMBER) offsetof(typeof(*(OBJECT)), MEMBER)

/* 使用方法 */
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

#define OBJECT_OFFSETOF(OBJECT, MEMBER) offsetof(typeof(*(OBJECT)), MEMBER)

int main(void)
{
    struct s {
        int i;
        char c;
        double d;
        char a[];
    } *obj = NULL;

    /* Output is compiler dependent */
    printf("offsets: i=%zd; c=%zd; d=%zd a=%zd\n",
            OBJECT_OFFSETOF(obj, i), OBJECT_OFFSETOF(obj, c),
            OBJECT_OFFSETOF(obj, d), OBJECT_OFFSETOF(obj, a));
    printf("sizeof(struct s)=%zd\n", sizeof(struct s));
    exit(EXIT_SUCCESS);
}
```

### CONTAINER_OF

已知结构体`STRUCT`中某个成员`MEMBER`的地址为`POINTER`，返回该结构体`STRUCT`的首地址。

```c
/* include/openvswitch/util.h #119 */
#define CONTAINER_OF(POINTER, STRUCT, MEMBER)                           \
        ((STRUCT *) (void *) ((char *) (POINTER) - offsetof (STRUCT, MEMBER)))

/* 使用方法 */
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

#define CONTAINER_OF(POINTER, STRUCT, MEMBER)                           \
        ((STRUCT *) (void *) ((char *) (POINTER) - offsetof (STRUCT, MEMBER)))

int main(void)
{
    struct s {
        int i;
        char c;
        double d;
        char a[];
    } obj;

    printf("obj addr: %p\n", &obj);
    printf("obj addr from i: %p\n", CONTAINER_OF(&obj.i, struct s, i));
    printf("obj addr from c: %p\n", CONTAINER_OF(&obj.c, struct s, c));
    printf("obj addr from d: %p\n", CONTAINER_OF(&obj.d, struct s, d));
    printf("obj addr from a: %p\n", CONTAINER_OF(&obj.a, struct s, a));

    exit(EXIT_SUCCESS);
}
```

#### OBJECT_CONTAINING

已知某个结构体对象中某个成员`MEMBER`的地址为`POINTER`，返回该对象的地址。

与`CONTAINER_OF`类似，只是由`STRUCT`替换为指向该结构体的指针`OBJECT`，且`OBJECT`是否为空无影响，只是用于获取结构体类型。

```c
#define OBJECT_CONTAINING(POINTER, OBJECT, MEMBER)                      \
    ((OVS_TYPEOF(OBJECT)) (void *)                                      \
     ((char *) (POINTER) - OBJECT_OFFSETOF(OBJECT, MEMBER)))

/* 使用方法 */
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

#define OBJECT_OFFSETOF(OBJECT, MEMBER) offsetof(typeof(*(OBJECT)), MEMBER)
#define OVS_TYPEOF(OBJECT) typeof(OBJECT)

#define OBJECT_CONTAINING(POINTER, OBJECT, MEMBER)                      \
    ((OVS_TYPEOF(OBJECT)) (void *)                                      \
     ((char *) (POINTER) - OBJECT_OFFSETOF(OBJECT, MEMBER)))

int main(void)
{
    struct s {
        int i;
        char c;
        double d;
        char a[];
    } obj;

    struct s *p = NULL; /* p 为空，但无影响 */
    printf("obj addr: %p\n", &obj);
    printf("obj addr from i: %p\n", OBJECT_CONTAINING(&obj.i, p, i));
    printf("obj addr from c: %p\n", OBJECT_CONTAINING(&obj.c, p, c));
    printf("obj addr from d: %p\n", OBJECT_CONTAINING(&obj.d, p, d));
    printf("obj addr from a: %p\n", OBJECT_CONTAINING(&obj.a, p, a));
    exit(EXIT_SUCCESS);
}
```

#### ASSIGN_CONTAINER

同上，已知某个结构体对象中某个成员`MEMBER`的地址为`POINTER`，但是将该结构体的地址赋值给`OBJECT`，返回`(void)0`。

`,`逗号运算符的优先级最低！所以这里是先对`OBJECT`赋值。

```c
#define ASSIGN_CONTAINER(OBJECT, POINTER, MEMBER) \
    ((OBJECT) = OBJECT_CONTAINING(POINTER, OBJECT, MEMBER), (void) 0)

struct s *p = NULL; /* p 为空，但无影响 */
printf("obj addr from i: %p\n", OBJECT_CONTAINING(p, &obj.i, i)); /* p == &obj */
```

#### INIT_CONTAINER

同上，就多个一个对`OBJECT`初始化为`NULL`的操作。

```c
#define INIT_CONTAINER(OBJECT, POINTER, MEMBER) \
    ((OBJECT) = NULL, ASSIGN_CONTAINER(OBJECT, POINTER, MEMBER))
```

### CONST_CAST

将`const`修饰的指针转为`non-const`类型。当指定的类型`TYPE`与指针`POINTER`类型不匹配时，编译会有警告。

```c
#define BUILD_ASSERT_TYPE(POINTER, TYPE) \
    ((void) sizeof ((int) ((POINTER) == (TYPE) (POINTER))))

#define CONST_CAST(TYPE, POINTER)                               \
    (BUILD_ASSERT_TYPE(POINTER, TYPE),                          \
     (TYPE) (POINTER))

/* 使用方法 */
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

#define BUILD_ASSERT_TYPE(POINTER, TYPE) \
    ((void) sizeof ((int) ((POINTER) == (TYPE) (POINTER))))

#define CONST_CAST(TYPE, POINTER)                               \
    (BUILD_ASSERT_TYPE(POINTER, TYPE),                          \
     (TYPE) (POINTER))

int main(void)
{
    const int constant = 26;
    const int* const_p = &constant;
    int* modifier = CONST_CAST(int *, const_p);
    /* int* modifier = CONST_CAST(double *, const_p);
     * 当指定的类型 double* 与指针类型不匹配时，编译会有警告 */
    *modifier = 3;

    printf("constant: %d\n", constant);
    printf("*modifier: %d\n", *modifier);

    return 0;
}
```

## ovs_list

> include/openvswitch/list.h
>
> [Linux内核中常用的数据结构](https://zhuanlan.zhihu.com/p/58087261)

双向链表结构！

```c
struct ovs_list {
    struct ovs_list *prev;     /* Previous list element. */
    struct ovs_list *next;     /* Next list element. */
};

struct s {
    int a;
    int b;
    struct ovs_list node;
}
```

当某个结构体需要实现链表时，只需要将该`ovs_list`嵌入到结构体中。

![ovs_list](../images/OpenVSwitch-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/OVS%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-ovs_list.drawio.png)

`prev/next`指向的是`struct s`中的`node`，当需要`s`的地址时，需要通过上面的宏`CONTAINER_OF`或者`OBJECT_CONTAINING`。

**LIST_FOR_EACH**：`ITER`为结构体的空指针`struct s *p = NULL`，遍历过程中会指向当前节点；`MEMBER`为结构体中`ovs_list`成员的名称`node`；`LIST`为链表的头节点（或者任意链表中的任意一个节点），`LIST`自身不会被遍历到；

```c
#define LIST_FOR_EACH(ITER, MEMBER, LIST)                               \
    for (INIT_CONTAINER(ITER, (LIST)->next, MEMBER);                    \
         &(ITER)->MEMBER != (LIST);                                     \
         ASSIGN_CONTAINER(ITER, (ITER)->MEMBER.next, MEMBER))
```

**LIST_FOR_EACH_CONTINUE**：同上，但`ITER`有初始值，从当前位置的下一个节点开始遍历。

```c
#define LIST_FOR_EACH_CONTINUE(ITER, MEMBER, LIST)                      \
    for (ASSIGN_CONTAINER(ITER, (ITER)->MEMBER.next, MEMBER);             \
         &(ITER)->MEMBER != (LIST);                                     \
         ASSIGN_CONTAINER(ITER, (ITER)->MEMBER.next, MEMBER))
```

**LIST_FOR_EACH_REVERSE**：与`LIST_FOR_EACH`类似，但反向遍历。

```c
#define LIST_FOR_EACH_REVERSE(ITER, MEMBER, LIST)                       \
    for (INIT_CONTAINER(ITER, (LIST)->prev, MEMBER);                    \
         &(ITER)->MEMBER != (LIST);                                     \
         ASSIGN_CONTAINER(ITER, (ITER)->MEMBER.prev, MEMBER))
```

一个例子：

- 将该代码放在`ovs`源码根目录下，编译时指定头文件路径：`gcc test.c -o test -I include`。
- 将宏定义展开：`gcc -E -P test.c -o test.i` ，直接跳转到最后看。

```c
#include "openvswitch/list.h"

#include <stdio.h>
#include <stdlib.h>

struct Person {
    char name;
    int age;
    struct ovs_list node;
};

int main() {
    struct Person A = {.name = 'A', .age = 1};
    struct Person B = {.name = 'B', .age = 2};
    struct Person C = {.name = 'C', .age = 3};
    struct Person D = {.name = 'D', .age = 4};

    struct ovs_list head;   /* 可创建一个单独的 ovs_list 作为 head */
    ovs_list_init(&head);

    /* 插入：在 head 前面插入 A */
    ovs_list_insert(&head, &A.node);    /* A - head */
    ovs_list_insert(&head, &B.node);    /* A - B - head */
    ovs_list_insert(&B.node, &C.node);  /* A - C - B - head */

    /* 替换：用 D 替换原来 C 的位置 */
    ovs_list_replace(&D.node, &C.node); /* A - D - B - head */

    struct Person *p = NULL;

    /* 遍历链表，输出：A - D - B */
    LIST_FOR_EACH(p, node, &head) {
        printf("name: %c\n", p->name);
    }

    /* 删除：删除指定节点，返回删除元素的下一个节点 */
    ovs_list *pl = ovs_list_remove(&D.node); /* pl == &B.node */
}
```

## hmap

> include/openvswitch/hmap.h
> lib/hmap.c
>
> [深入分析Hmap](https://www.sdnlab.com/15552.html)

一种哈希桶实现。

- 桶的数量可自动扩容、收缩，且总是`2^n`。
- 哈希函数就是`key & mask`，这里`mask = 2^n-1`。
- 通过开链法解决哈希冲突。
- 节点数量为0或1时比较特殊。

> 将哈希桶的大小设置为`2^n`是为了加速。假设桶的大小为`len`，那么哈希函数就应该为`key % len`，而当`len = 2^n`时，`key % len == key & (len - 1)`，相比之下，`&`操作比`%`更快。

```c
struct hmap {
    /* buckets 就是哈希桶，本质为指针数组（hmap_node *）
     * 如果 mask == 0, buckets = &one */
    struct hmap_node **buckets;
    /* one 只有当 mask == 0 时才存储数据，mask != 0, 则 one = NULL; */
    struct hmap_node *one;
    /* buckets 数组的大小为 mask + 1 */
    size_t mask;
    /* hmap 中有效 hmap_node 节点个数 */
    size_t n;
};

/* 一个哈希映射节点，用于嵌入到被映射的数据结构中 */
struct hmap_node {
    size_t hash;                /* 哈希值 */
    struct hmap_node *next;     /* 单链表 */
};
```

![hmap](../images/OpenVSwitch-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/OVS%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-hmap.drawio.png)

同样，通过`hmap`得到`hmap_node`的地址后，需要通过`CONTAINER_OF`获取对应结构体对象的地址。

常用方法：

```c
void hmap_init(struct hmap *);   /* 初始化 */
#define hmap_insert(HMAP, NODE, HASH)   /* 插入节点 */
void hmap_remove(struct hmap *, struct hmap_node *);  /* 删除指定节点 */
struct hmap_node *hmap_random_node(const struct hmap *); /* 随机返回一个节点 */
bool hmap_contains(const struct hmap *, const struct hmap_node *); /* 判断hmap是否包含该节点 */

void hmap_destroy(struct hmap *);  /* 销毁hmap，释放buckets的内存，但不负责销毁hmap_node对应的资源 */
void hmap_clear(struct hmap *);    /* 将 buckets 数组清0，大小不变，只是所有指针置为 NULL(0) */
```

**HMAP_FOR_EACH_WITH_HASH**：遍历 `HMAP` 中所有 `hash_node->hash` 值等于 `HASH` 的节点，`Node`实际`struct`的指针；

根据`HASH`找到哈希桶所在的下标，然后遍历对应的链表。链表中，有可能不同`hash`值的节点，这些节点会被跳过。

```c
#define HMAP_FOR_EACH_WITH_HASH(NODE, MEMBER, HASH, HMAP)               \
    for (INIT_CONTAINER(NODE, hmap_first_with_hash(HMAP, HASH), MEMBER); \
         (NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER))                \
         || ((NODE = NULL), false);                                     \
         ASSIGN_CONTAINER(NODE, hmap_next_with_hash(&(NODE)->MEMBER),   \
                          MEMBER))

/* 返回下一个具有相同 hmap_node->hash 值的节点，没有返回 NULL，会跳过hash值不同的节点 */
struct hmap_node *
hmap_next_with_hash(const struct hmap_node *node)
```

**HMAP_FOR_EACH_IN_BUCKET**：同上，但遍历链表时，**不会跳过**有不同`hash`值的节点。

```c
#define HMAP_FOR_EACH_IN_BUCKET(NODE, MEMBER, HASH, HMAP)               \
    for (INIT_CONTAINER(NODE, hmap_first_in_bucket(HMAP, HASH), MEMBER); \
         (NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER))                \
         || ((NODE = NULL), false);                                     \
         ASSIGN_CONTAINER(NODE, hmap_next_in_bucket(&(NODE)->MEMBER), MEMBER))
```

**HMAP_FOR_EACH**：遍历`HMAP`中的所有节点。

```c
#define HMAP_FOR_EACH(NODE, MEMBER, HMAP) \
    HMAP_FOR_EACH_INIT(NODE, MEMBER, HMAP, (void) 0)
#define HMAP_FOR_EACH_INIT(NODE, MEMBER, HMAP, ...)                     \
    for (INIT_CONTAINER(NODE, hmap_first(HMAP), MEMBER), __VA_ARGS__;   \
         (NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER))                \
         || ((NODE = NULL), false);                                     \
         ASSIGN_CONTAINER(NODE, hmap_next(HMAP, &(NODE)->MEMBER), MEMBER))
```

**HMAP_FOR_EACH_SAFE**：同上。

```c
#define HMAP_FOR_EACH_SAFE(NODE, NEXT, MEMBER, HMAP) \
    HMAP_FOR_EACH_SAFE_INIT(NODE, NEXT, MEMBER, HMAP, (void) 0)
#define HMAP_FOR_EACH_SAFE_INIT(NODE, NEXT, MEMBER, HMAP, ...)          \
    for (INIT_CONTAINER(NODE, hmap_first(HMAP), MEMBER), __VA_ARGS__;   \
         ((NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER))               \
          || ((NODE = NULL), false)                                     \
          ? INIT_CONTAINER(NEXT, hmap_next(HMAP, &(NODE)->MEMBER), MEMBER), 1 \
          : 0);                                                         \
         (NODE) = (NEXT))
```

**HMAP_FOR_EACH_CONTINUE**：可以从中间某个位置（`NODE`所在位置）开始遍历。

```c
#define HMAP_FOR_EACH_CONTINUE(NODE, MEMBER, HMAP) \
    HMAP_FOR_EACH_CONTINUE_INIT(NODE, MEMBER, HMAP, (void) 0)
#define HMAP_FOR_EACH_CONTINUE_INIT(NODE, MEMBER, HMAP, ...)            \
    for (ASSIGN_CONTAINER(NODE, hmap_next(HMAP, &(NODE)->MEMBER), MEMBER), \
         __VA_ARGS__;                                                   \
         (NODE != OBJECT_CONTAINING(NULL, NODE, MEMBER))                \
         || ((NODE = NULL), false);                                     \
         ASSIGN_CONTAINER(NODE, hmap_next(HMAP, &(NODE)->MEMBER), MEMBER))
```

一个例子：

- 将该代码放在`ovs`源码根目录下，编译时指定头文件路径：~~`gcc test.c lib/hmap.c -o test -I include`~~。
- 将宏定义展开：` gcc -E -P test.c -o test.p -I include -I .` ，直接跳转到最后看。

```c
#include "openvswitch/hmap.h"

#include <stdio.h>
#include <assert.h>
#include <errno.h>
#include <stdlib.h>
#include <unistd.h>

struct element {
    int value;
    struct hmap_node node;
};

/* Tests basic hmap insertion and deletion. */
static void
test_hmap_insert_delete(void)
{
    enum { N_ELEMS = 100 };

    /* 定义n个元素数组 */
    struct element elements[N_ELEMS];
    int values[N_ELEMS];
    struct hmap hmap;
    size_t i;
    struct hmap_node *data = NULL;

    hmap_init(&hmap);

    for (i = 0; i < N_ELEMS; i++) {
        elements[i].value = i+1;    /* 设置每个元素的hash为 下标+1 */
        hmap_insert(&hmap, &elements[i].node, i+1);
        /* 查看 hmap 自动扩容 */
        printf("i+1 = %d, hmap.mask = %d, .n = %d\n", i, hmap.mask, hmap.n);
        values[i] = i;
    }

    for (i = 0; i < hmap.mask; i++) {
        data = hmap.buckets[i];
        while(1) {
            printf("hash=%d,next=%p\n", data->hash, data->next);
            if (data->next == NULL) break;
            data = data->next;
        }
        printf("--------------------\n");
    }
    hmap_destroy(&hmap);
}
int main()
{
    test_hmap_insert_delete();
    return 0;
}
```

## smap



## simap



## cmap



## skiplist



## sset



## hash



## shash

