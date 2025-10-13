---
layout: post
tags: [kernel_pwn]
title: "linux内核基础 三 slub分配器详解"
author: wsxk
date: 2022-10-16
comments: true
---

`更新于2022-10-19`<br>
PS:请在学习完 [linux内核基础一](https://wsxk.github.io/linux_kernel_basic_one/)以及[Linux内核基础二](https://wsxk.github.io/linux_kernel_basic_two/)和相应的练习后进行三的学习<br>


- [buddy system](#buddy-system)
- [slub分配器工作原理](#slub分配器工作原理)
  - [kmem\_cache](#kmem_cache)
  - [kmem\_cache\_cpu](#kmem_cache_cpu)
  - [kmem\_cache\_node](#kmem_cache_node)
  - [slub API](#slub-api)
  - [slub各个数据结构间的关系](#slub各个数据结构间的关系)
  - [slub内存分配原理/释放原理](#slub内存分配原理释放原理)
    - [分配](#分配)
    - [释放](#释放)
  - [kmalloc](#kmalloc)
- [references](#references)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## buddy system<br>
作为linux的最底层的内存管理器，详细请移步[linux内核基础一](https://wsxk.github.io/linux_kernel_basic_one/),不过多赘述<br>
## slub分配器工作原理<br>
`buddy system + slub` 构成了linux内核的内存分配系统<br>，关于slub的叙述在[linux内核基础一](https://wsxk.github.io/linux_kernel_basic_one/)不是很详细，在重新学习了一下slub的原理后进行更详细的描述<br>
其中要注意的一点是 slab、slob、slub 都是slab的一种（slab是最初设计的，机制复杂，效率较低；slob是针对嵌入式在slab的基础上编写的内存分配器，比较轻量；slub是在slab上简化代码逻辑，提高效率和安全性，已经成为现行linux内核的主流分配器）<br>
slab会诞生的原因也很简单:buddy system按页提供内存，如果要申请15字节的内存，你一次给一个page过于浪费了，因此slab实现用于提供更细粒度的内存规划<br>
因为先行主流是slub，所以直接学习slub,首先需要大家了解的是`kmem_cache`、`kmem_cache_cpu`、`kmem_cache_node`这三个重要结构体<br>
### kmem_cache<br>
slub在向buddy system申请来若干页内存后，（该内存由slub管理器自由划分，被称为一个`slab`）,将内存划分为若干个`object`，为此我们需要知道该`object`的大小啊，剩余数量啊，等等信息,kmem_cache就是用于记录这些信息的管理结构体。<br>
其定义如下:
```c
struct kmem_cache{
    struct kmem_cache_cpu __percpu * cpu_slab;//percpu变量表示该变量始终位于cpu本地内存缓冲池中（分配内存时优先分配在cou内存缓冲池中以保证命中率）
    slab_flags_t flags;//分配掩码，例如经常使用的SLAB_HWCACHE_ALIGN标志
    unsigned long min_partial;//限制 kmem_cache_node中的partial链表的 slab最大数量
    int size;//分配的object大小，用户填写的
    int object_size;//实际的object大小，size+对齐+各种元数据（用于管理object）
    int offset;//下一个object相对于当前object的偏移
#ifdef CONFIG_SLUB_CPU_PARTIAL
    int cpu_partial;//限制 kmem_cache_cpu中的partial链表的所有slab 的free object的最大值
#endif
    struct kmem_cache_order_objects oo;//低16位表示一个slab中的所有object数量，高16位表示page数量
    struct kmem_cache_order_ojbects max;//等于oo
    struct kmem_cache_order_ojbects min;//按照oo分配内存时出现内存不足后使用该定义分配
    gfp_t allocflags;//从buddy system 分配内存的掩码
    int refcount;
    void (*ctor) (void *);
    int inuse;//object size对齐后的大小
    int align;//字节对齐大小
    int reserved;
    const char * name;//sysfs文件系统显示时使用
    struct list_head list;//系统有一个slab_caches链表，所有的slab都会挂入此链表。
    struct kmem_cache_node * node[MAX_NUMNODES];//slab节点 NUMA系统中每个内存控制器都有一个节点
};
```

### kmem_cache_cpu<br>
kmem_cache_cpu是对于cpu本地内存缓冲池的描述，每个cpu对应一个结构体<br>
其定义如下:
```c
struct kmem_cache_cpu{
    void ** freelist;//空闲object的链表
    unsigned long tid;//神奇的标志，用于cpu的同步
    struct page *page;//当前cpu使用的slab
#ifdef CONFIG_SLUB_CPU_PARTIAL
    struct page *partial;//本地 slab partial链表，部分使用object的slab组成
#endif
};
```

### kmem_cache_node<br>
kmem_cache_node用以描述一个 slab节点,由同一内存控制器下的所有cpu共享<br>
其定于如下:
```c
struct kmem_struct_node{
    spinlock_t list_lock;//锁
    unsigned long nr_partial;//当前 node中的partial链表中的slab数量
    struct list_head partial; //slab节点的 slab partial链表
};
```

### slub API<br>
我们需要用到的API就4个，非常少，顾名思义也能看出怎么用。<br>
```c
struct kmem_cache *kmem_cache_create(const char *name,// name
        size_t size,//object的size
        size_t align,//对齐要求
        unsigned long flags,//分配内存掩码
        void (*ctor)(void *));//分配对象的构造回调函数
void kmem_cache_destroy(struct kmem_cache *);
void *kmem_cache_alloc(struct kmem_cache *cachep, int flags);
void kmem_cache_free(struct kmem_cache *cachep, void *objp);
```

### slub各个数据结构间的关系<br>
借图，侵删<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/4a471520078976.png)

### slub内存分配原理/释放原理<br>
[http://www.wowotech.net/memory_management/426.html](http://www.wowotech.net/memory_management/426.html)写的真的很好，还配有图示实例,非常有助于理解<br>
强推！但是为了巩固理解，这里还是自己重新写一遍吧<br>
#### 分配<br>
在调用kmem_cache_alloc函数后发生的事情<br>
同样借用大佬的图，侵删
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/fb5c1519305301.png)<br>
> 1. 申请时，首先看`kmem_cache_cpu->page`是否存在，若不存在，进入第2步
> > 若存在，查看`kmem_cacha_cpu->freelist`是否有空闲object
> > > 若有空闲object，从 freelist里取出object，返回object结束。
> > > 若没有空闲object，进入第2步
> 2. 查看 `kmem_cache_cpu->partial`链表中是否有 未完全使用的`slab`
> > 若有未完全使用的`slab`，取出一个并替换 `kmem_cache_cpu->page`的当前值，进入第1步
> > 若没有，进入第3步
> 3. 查看 `kmem_cache_node->partial`链表中是否有未完全使用的`slab`
> > 如果有未完全使用的`slab`，取出一个并替换 `kmem_cache_cpu->page`的当前值，进入第1步
> > 如果没有，申请一个新的slab(挂在node里）并返回 其中一个object

#### 释放<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/9eb91519305304.png)<br>
> 1. 首先查看释放的object所属的slab是否`kmem_cache_cpu->page`指向的slab
> > 若是，直接free即可
> > 若不是，进入第2步
> 2. 查看该slab是否原先是满的slab（即不存在于kmem_cache_cpu何kmem_cache_node的partial链表中）
> 
> > 若是，将该slab加入`kmem_cache_cpu->partial`链表中，随后判断`kmem_cache_cpu->partial`中的free_objects数量是否大于`kmem_cache->cpu_partial`的值
> > > 若大于，除开该slab外，原来`kmem_cache_cpu->partial`链表的所有slab链入`kmem_cache_node->partial`链表中
> > > 小于等于，则返回
> > 
> > 若不是，查看该slab在释放object后是否为空slab
> > > 若是空slab，判断`kmem_cache_node->partial`的slab数量是否大于等于`kmem_cache->min_partial`的值
> > > > 若大于等于，则将该slab归还给buddy system后返回
> > > > 若小于，直接返回（该slab本身就存在于`kmem_cache_cpu->partial`或`kmem_cahce_node->partial`中)
> > > >
> > > 若不是空slab，直接返回


### kmalloc<br>
kmalloc底层就是使用了slab分配器的。<br>
在linux系统启动时，会调用`create_kmalloc_caches ()`创建多个管理不同大小对象的kmem_cache。kmem_cache的名称以及大小使用struct kmalloc_info_struct管理，如下图所示<br>
`create_kmalloc_caches`内部会调用`create_kmalloc_cache`创建以上不同size的`kmem_cache`
```c
const struct kmalloc_info_struct kmalloc_info[] __initconst = { 
    {NULL,                        0},     {"kmalloc-96",             96},
    {"kmalloc-192",           192},     {"kmalloc-8",               8},
    {"kmalloc-16",             16},     {"kmalloc-32",             32},
    {"kmalloc-64",             64},     {"kmalloc-128",           128},
    {"kmalloc-256",           256},     {"kmalloc-512",           512},
    {"kmalloc-1024",         1024},     {"kmalloc-2048",         2048},
    {"kmalloc-4096",         4096},     {"kmalloc-8192",         8192},
    {"kmalloc-16384",       16384},     {"kmalloc-32768",       32768},
    {"kmalloc-65536",       65536},     {"kmalloc-131072",     131072},
    {"kmalloc-262144",     262144},     {"kmalloc-524288",     524288},
    {"kmalloc-1048576",   1048576},     {"kmalloc-2097152",   2097152},
    {"kmalloc-4194304",   4194304},     {"kmalloc-8388608",   8388608},
    {"kmalloc-16777216", 16777216},     {"kmalloc-33554432", 33554432},
    {"kmalloc-67108864", 67108864}
};
```
在调用完成后，系统会将这些`kmem_cache`放在全局变量`kmalloc_caches`中<br>
kmalloc源码如下:
```c
static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
    if (__builtin_constant_p(size)) {//gcc工具用来判断参数是否是一个常数,优化
        if (size > KMALLOC_MAX_CACHE_SIZE)
            return kmalloc_large(size, flags);
        if (!(flags & GFP_DMA)) {
            int index = kmalloc_index(size);//判断size属于那个kmem_cache
            if (!index)
                return ZERO_SIZE_PTR;
            return kmem_cache_alloc_trace(kmalloc_caches[index], flags, size);//分配
        }
    }
    return __kmalloc(size, flags);
}
```

## references<br>
[per_cpu变量](http://www.wowotech.net/kernel_synchronization/per-cpu.html)<br>
[slub详解](http://www.wowotech.net/memory_management/426.html)<br>
[知乎](https://zhuanlan.zhihu.com/p/382056680)
[kmalloc_caches](https://blog.csdn.net/caofengtao1314/article/details/117354093?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-117354093-blog-123162764.pc_relevant_aa&spm=1001.2101.3001.4242.1&utm_relevant_index=3)