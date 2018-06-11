# Python内存管理

原链[Memory management in Python](https://rushter.com/blog/python-memory-managment/)。

这篇文章描述了Python3.6的内存管理。如果你对GC的细节有兴趣，可以阅读我的另一篇文章[Garbage collection in Python](https://rushter.com/blog/python-garbage-collector/)。

Python中的一切皆是对象。有些对象可以包含其它对象，比如列表、元组、字典、类等。因为Python的动态语言天性，这种方式需要对内存进行很多的琐碎的分配。为了加速内存操作和减，Python在通用内存分配器上使用一个称作PyMalloc的特殊管理器。

我们可以将整个系统描绘成一个分层次的集合：

![分层次模型的图解](https://rushter.com/static/uploads/img/memory_layers.svg)

## 小对象的分配

为了减少小对象（少于512字节）的花费，Python将内存分成一些块。大对象使用C的标准分配器。小对象分配器使用一个三级抽象：arena，pool和block。

让我们先来学习最小的结构——block。

## Block

Block是一块大小确定的内存。每个block仅仅存储大小固定的Python对象。Block的大小从8字节到512字节不等，但是必须要为8的倍数（即8字节的填充）。为了方便，这些block可以分成64个大小的类。

|Request in bytes|Size of allocated block|size class idx|
|----------------|-----------------------|--------------|
|1-8              |8| 0|
|9-16|16|1|
|17-24|24|2|
|25-32|32|3|
|33-40|40|4|
|41-48|48|5|
|...|...|...|
|505-512|512|63|

## Pool

一个相同大小的块的集合就是一个pool。正常情况下，pool的大小和内存中的页一直（即4Kb）。现在pool的大小有助于段。如果一个对象需要销毁，内存管理器可以用一个相同大小的新对象填充它所占的位置。

每个pool拥有一个特殊的头结构，定义如下：
```c
/* Pool for small blocks. */
struct pool_header {
    union { block *_padding;
            uint count; } ref;          /* number of allocated blocks    */
    block *freeblock;                   /* pool's free list head         */
    struct pool_header *nextpool;       /* next pool of this size class  */
    struct pool_header *prevpool;       /* previous pool       ""        */
    uint arenaindex;                    /* index into arenas of base adr */
    uint szidx;                         /* block size class index        */
    uint nextoffset;                    /* bytes to virgin block         */
    uint maxnextoffset;                 /* largest valid nextoffset      */
};
```
相同大小的块组成的pool使用双向链表链接到一起（`nextpool`he `prevpool`字段）。`szidx`字段保存大小分类的索引，`ref.count`保存被使用的块的数量。`arenaindex`保存pool所在的arena。

`freeblock`字段描述如下：
> Pool中的块按需创建。`poll->freeblock`

