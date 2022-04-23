---
title: STL容器
date: 2022-04-23 13:28:02
updated: 2022-04-23 13:28:02
tags:
  - STL
  - C++
categories: C++
keywords:
description:
top_img: transparent
---

效率说明：

- emplace_back() 是c11新加的，功能与 push_back() 一样，但效率更高，推荐优先使用

## 序列式容器

### Array

**固定大小的数组**，必须在建立时就指明其大小。

- `.front()`：返回第一元素的**引用**，你可以使用这些操作函数更改元素内容
- `.back() `：返回最末元素的**引用**，`c.back() = 10`


### Vector

Vector是一个dynamic array。

内存重新分配很耗时间。

**在array尾部附加元素或移除元素都很快速**，但是在array的中段或起始段安插元素就比较费时

- `.front()`：返回第一元素的**引用** (不检查是否存在第一元素)
- `.back() `：返回最末元素的**引用** (不检查是否存在最末元素)，`c.back() = 10`
- `.push_back(elem)`：尾部附加元素
- `.pop_back()`：移除最后一个元素
- `.insert(pos, elem)`：在iterator 位置 pos 之前方插入一个 elem 拷贝，并返回新元素的位置
- `.insert(pos, n, elem)`：在iterator 位置 pos 之前方插入 n 个 elem 拷贝，并返回第一个新元素的位置
- `.emplace(pos, args...)`：在iterator 位置 pos 之前方插入一个以 args 为初值的元素，并返回新元素的位置
- `.emplace_back(args...)`：附加一个以 args 为初值的元素于未尾，不返回任何东西
- `.resize(num)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都需以 default 构造函数完成初始化)
- `.resize(num, elem)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都是 elem 的拷贝)
- `.clear()`：移除所有元素，将容器清空

### Deque

“double-ended queue”的缩写。它是一个dynamic array，可以向两端发展。

因此**不论在尾部或头部安插元素都十分迅速**。在**中间部分安插元素则比较费时**，因为必须移动其他元素。

访问元素时deque内部结构会多一个间接过程，所以元素的访问和迭代嚣的动作会稍稍慢一些。

特别注意，除了头尾两端，在任何地点安插或删除元素都将导致指向deque元素的任何pointer、reference和iterator失效。

deque的内存重分配优于vector，deque不必在内存重新分配时复制所有元素。

- `.front()`：返回第一元素的**引用** (不检查是否存在第一元素)
- `.back() `：返回最末元素的**引用** (不检查是否存在最末元素)，`c.back() = 10`
- `.push_back(elem)`：尾部附加元素
- `.pop_back()`：移除最后一个元素
- **`.push_front(elem)`：头部附加元素**
- **`.pop_front()`：移除第一个元素**
- `.insert(pos, elem)`：在iterator 位置 pos 之前方插入一个 elem 拷贝，并返回新元素的位置
- `.insert(pos, n, elem)`：在iterator 位置 pos 之前方插入 n 个 elem 拷贝，并返回第一个新元素的位置
- `.emplace(pos, args...)`：在iterator 位置 pos 之前方插入一个以 args 为初值的元素，并返回新元素的位置
- `.emplace_back(args...)`：附加一个以 args 为初值的元素于未尾，不返回任何东西
- **`.emplace_front(args...)`：附加一个以 args 为初值的元素于起点，不返回任何东西**
- `.resize(num)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都需以 default 构造函数完成初始化)
- `.resize(num, elem)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都是 elem 的拷贝)
- `.clear()`：移除所有元素，将容器清空


### List

双向链表，**在任何位置上执行安插或删除动作都非常迅速**。

- `.front()`：返回第一元素的**引用**
- `.back() `：返回最末元素的**引用**，`c.back() = 10`
- `.push_back(elem)`：尾部附加元素
- `.pop_back()`：移除最后一个元素
- `.push_front(elem)`：头部附加元素
- `.pop_front()`：移除第一个元素
- `.insert(pos, elem)`：在iterator 位置 pos 之前方插入一个 elem 拷贝，并返回新元素的位置
- `.insert(pos, n, elem)`：在iterator 位置 pos 之前方插入 n 个 elem 拷贝，并返回第一个新元素的位置
- `.emplace(pos, args...)`：在iterator 位置 pos 之前方插入一个以 args 为初值的元素，并返回新元素的位置
- `.emplace_back(args...)`：附加一个以 args 为初值的元素于未尾，不返回任何东西
- `.emplace_front(args...)`：附加一个以 args 为初值的元素于起点，不返回任何东西
- `.resize(num)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都需以 default 构造函数完成初始化)
- `.resize(num, elem)`：将元素数量改为 num (如果 size() 变大，多出来的新元素都是 elem 的拷贝)
- `.remove(val)`：移除所有值为 val 的元素
- `.remove_if(op)`：移除所有造成 op(elem) 返回 true 的元素
- `.clear()`：移除所有元素，将容器清空
- `.merge(other, comp)`：将两个有序 List 合并成一个
- `.unique()`：移除所有重复的元素，重复元素保留第一个
- `.sort()`：对 List 按照升序排序


### Forward_List

forward list原则上就是一个受限的list，不支持任何“后退移动"或“效率低下"的操作。

基于这个原因，它不提供成员函数如push_back() 甚至size()。

- `.front()`：返回第一元素的**引用**
- `.push_front(elem)`：头部附加元素
- `.pop_front()`：移除第一个元素
- `.insert_after(pos, args)`：在指定的 pos 后插入 args
- `.emplace_after(pos, args)`：在指定的 pos 后插入 args
- `.emplace_front(args...)`：附加一个以 args 为初值的元素于起点
- `.remove(val)`：移除所有值为 val 的元素
- `.remove_if(op)`：移除所有造成 op(elem) 返回 true 的元素
- `.clear()`：移除所有元素，将容器清空
- `.merge(other, comp)`：将两个有序 List 合并成一个
- `.unique()`：移除所有重复的元素，重复元素保留第一个
- `.sort()`：按照升序排序


## 关联式容器

关联式容器依据特定的排序准则，自动为其元素排序。

自动排序造成的一个重要限制: 你不能直接改变元素值，因为会打乱原本正确的顺序。

**要改变元素值，必须先删除旧元素，再插入新元素。**

通常关联式容器由**二又搜索树**实现。


### set、multiset

**由于 set 不允许重复值，set 的 insert/emplace 返回值为 pair<iterator, bool>，其中 bool 是指是否插入成功。**

- `.count(val)`：返回值为val的个数
- `.find(val)`：返回值为val的第一个元素的迭代器，找不到返回end()
- `.contains(val)`：判断是否包含某个元素（c++20）
- `.lower_bound(val)`：返回val的第一个可安插位置，第一个 >= val 的位置
- `.upper_bound(val)`：返回val的第一个可安插位置，第一个 > val 的位置
- `.equal_range(val)`：返回val可安插的第一个和最后一个位置
- `.insert(val)`：插入元素，返回插入的元素的迭代器
- `.insert(pos, val)`：插入元素，返回插入的元素的迭代器（pos是提示插入起点）
- `.emplace(args...)`：插入元素，返回插入的元素的迭代器
- `.emplace_hint(pos, args...)`：插入元素，返回插入的元素的迭代器（pos 是提示插入起点）
- `.merge(other)`：将两个 set 合并成一个
- `.clear()`：移除所有元素，将容器清空

### map、multimap

- `.count(val)`：返回值为val的个数
- `.find(val)`：返回值为val的第一个元素的迭代器，找不到返回end()
- `.contains(val)`：判断是否包含某个元素（c++20）
- `.lower_bound(val)`：返回val的第一个可安插位置，第一个 >= val 的位置
- `.upper_bound(val)`：返回val的第一个可安插位置，第一个 > val 的位置
- `.equal_range(val)`：返回val可安插的第一个和最后一个位置
- `.insert(val)`：插入元素，返回插入的元素的迭代器
- `.insert(pos, val)`：插入元素，返回插入的元素的迭代器（pos是提示插入起点）
- `.emplace(args...)`：插入元素，返回插入的元素的迭代器
- `.emplace_hint(pos, args...)`：插入元素，返回插入的元素的迭代器（pos 是提示插入起点）
- `.merge(other)`：将两个 map 合并成一个
- `.clear()`：移除所有元素，将容器清空

## 无序容器

通过 hash 实现查找

### unorder_set、unorder_multiset

- `.count(val)`：返回值为val的个数
- `.find(val)`：返回值为val的第一个元素的迭代器，找不到返回end()
- `.contains(val)`：判断是否包含某个元素（c++20）
- `.insert(val)`：插入元素，返回插入的元素的迭代器
- `.insert(pos, val)`：插入元素，返回插入的元素的迭代器（pos是提示插入起点）
- `.emplace(args...)`：插入元素，返回插入的元素的迭代器
- `.emplace_hint(pos, args...)`：插入元素，返回插入的元素的迭代器（pos 是提示插入起点）
- `.merge(other)`：将两个 set 合并成一个
- `.clear()`：移除所有元素，将容器清空

### unorder_map、unorder_multimap

- `.count(val)`：返回值为val的个数
- `.find(val)`：返回值为val的第一个元素的迭代器，找不到返回end()
- `.contains(val)`：判断是否包含某个元素（c++20）
- `.insert(val)`：插入元素，返回插入的元素的迭代器
- `.insert(pos, val)`：插入元素，返回插入的元素的迭代器（pos是提示插入起点）
- `.emplace(args...)`：插入元素，返回插入的元素的迭代器
- `.emplace_hint(pos, args...)`：插入元素，返回插入的元素的迭代器（pos 是提示插入起点）
- `.merge(other)`：将两个 set 合并成一个
- `.clear()`：移除所有元素，将容器清空

## 特殊容器

### Stack

- `.push()`： 将一个元素放入 stack 中
- `.emplace()`： 将一个元素放入 stack 中，效率比 push 更高
- `.top()`： 返回 stack 内的下一个元素
- `.pop()`： 从 stack 内移除一个元素

### Queue

- `.front()`：返回第一元素的**引用**
- `.back() `：返回最末元素的**引用**，`c.back() = 10`
- `.push()`： 将一个元素放入 queue 中
- `.emplace()`： 将一个元素放入 queue 中，效率比 push 更高
- `.top()`： 返回 queue 内的下一个元素
- `.pop()`： 从 queue 内移除一个元素

### Priority_queue

和queue类似，但其元素有优先级，dequeue时并非取最先放入的元素，而是优先级最高的元素！

如果同时存在若干个优先级最高的元素，我们无法确知究竟哪一个会入选。

- `.push()`： 将一个元素放入priority queue中
- `.emplace()`： 将一个元素放入priority queue中，效率比 push 更高
- `.top()`： 返回priority queue内的下一个元素
- `.pop()`： 从priority queue内移除一个元素

### Bitset

Bitset造出了一个内含bit或Boolean值上且**大小固定的array**。

- `bitset<size>(value)`：以 value 初始化大小为size 的 bitset
- `bitset<numeric_limits<unsigned long>::digits>(value)`
- `.any()`：检查是否有 bit 被设置为1
- `.all()`：检查是否所有 bit 都为1
- `.none()`：检查是否所有 bit 都为0
- `.set(pos)`：设置 pos 位置的 bit 为1
- `.reset()`：设置所有 bit 为0
- `.reset(pos)`：设置 pos 位置的 bit 为0
- `.flip()`：翻转所有 bit
- `.flip(pos)`：翻转 pos 位置的 bit
- `.to_string()`：输出为01字符串
- `.to_ullong()`：输出为 unsigned long long 类型
