## 如何分配定长记录？
首先是基本问题，如何分配定长记录？例如，我们有一个 Page 的内存，大小为 4KB，现在要以 N 字节为单位进行分配。为了简化问题，就以 16 字节为单位进行分配。

解法有很多，比如，bitmap。4KB / 16 / 8 = 32, 用 32 字节做 bitmap即可，实现也相当简单。

出于最大化内存利用率的目的，我们使用另一种经典的方式，freelist。将 4KB 的内存划分为 16 字节的单元，每个单元的前8个字节作为节点指针，指向下一个单元。初始化的时候把所有指针指向下一个单元；分配时，从链表头分配一个对象出去；释放时，插入到链表。

由于链表指针直接分配在待分配内存中，因此不需要额外的内存开销，而且分配速度也是相当快
![如何分配定长记录？](https://pic2.zhimg.com/80/v2-8627f1c08819b6c8bd03d0b74935ba19_1440w.webp)


## 如何分配定长记录？
![如何分配定长记录](https://pic2.zhimg.com/80/v2-2c7c35cc567510ab8187b18297217b81_1440w.webp)


## 大的对象如何分配？
上面讲的是基于 Page，分配小于Page的对象，但是如果分配的对象大于一个 Page，我们就需要用多个 Page 来分配了：

![大的对象如何分配？](https://pic4.zhimg.com/80/v2-10c1aaa4eb52977fb9330d015d851cdf_1440w.webp)

## 如何管理page
![PageHeap](https://pic1.zhimg.com/80/v2-8092c1103b6f8f3bc64952e922451ea4_1440w.webp)


## 全局对象分配
![CentralCache](https://pic3.zhimg.com/80/v2-31841a2868a542b4e5384f938b56124e_1440w.webp)

## 本地线程局部对象分配
![ThreadCache](https://pic4.zhimg.com/80/v2-05a8740554bedf4dc0a6912c6e8551db_1440w.webp)

## refer
* [图解 TCMalloc](https://zhuanlan.zhihu.com/p/29216091)