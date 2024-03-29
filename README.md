# MemoryPool

# **为什么要用内存池**

为什么要用内存池？首先，在7 * 24h的服务器中如果不使用内存池，而使用malloc和free，那么就非常容易产生内存碎片，早晚都会申请内存失败；并且在比较复杂的代码或者继承的屎山中，非常容易出现内存泄漏导致mmo的问题。

为了解决这两个问题，内存池就应运而生了。内存池预先分配一大块内存来做一个内存池，业务中的内存分配和释放都由这个内存池来管理，内存池内的内存不足时其内部会自己申请。所以内存碎片的问题就交由内存池的算法来优化，而内存泄漏的问题只需要遵守内存池提供的api，就非常容易避免内存泄漏了。

即使出现了内存泄漏，排查的思路也很清晰。1.检查是不是内存池的问题；2.如果不是内存池的问题，就检查是不是第三方库的内存泄漏。

# **内存池的使用场景**

全局内存池

一个连接一个内存池(本文实现这个场景的内存池)

# **总体介绍**

由于本文是一个连接一个内存池，所以后续介绍和代码都是以4k为分界线，大于4k的我们认为是大块内存；小于4k的我们认为是小块内存。并且注意这里的4k，并不是严格遵照4096，而是在描述上，用4k比较好描述。

在真正使用内存之前，内存池提前分配一定数量且大小相等的内存块以作备用，当真正被用户调用api分配内存的时候，直接从内存块中获取内存（指小块内存），当内存块不够用了，再有内存池取申请新的内存块。而如果是需要大块内存，则内存池直接申请大块内存再返回给用户。

内存池：就是将这些提前申请的内存块组织管理起来的数据结构，内存池实现原理主要分为分配，回收，扩容三部分。

内存池原理之小块内存：分配=> 内存池预申请一块4k的内存块，这里称为block，即block=4k内存块。当用户向内存池申请内存size小于4k时，内存池从block的空间中划分出去size空间，当再有新申请时，再划分出去。扩容=> 直到block中的剩余空间不足以分配size大小，那么此时内存池会再次申请一块block，再从新的block中划分size空间给用户。回收=> 每一次申请小内存，都会在对应的block中引用计数加1，每一次释放小内存时，都会在block中引用计数减1，只有当引用计数为零的时候，才会回收block使他重新成为空闲空间，以便重复利用空间。这样，内存池避免频繁向内核申请/释放内存，从而提高系统性能。

内存池原理之大块内存：分配=> 因为大块内存是大于4k的，所以内存池不预先申请内存，也就是用户申请的时候，内存池再申请内存，然后返回给用户。扩容=> 大块内存不存在扩容。回收=> 对于大块内存来说，回收就直接free掉即可。

上面理论讲完了，下面来介绍如何管理小块内存和大块内存。

## 向外提供的api

```
mp_create_pool：创建一个线程池，其核心是创建struct mp_pool_s这个结构体，并申请4k内存，将各个指针指向上文初始状态的图一样。

mp_destroy_pool：销毁内存池，遍历小块结构体和大块结构体，进行free释放内存

mp_malloc：提供给用户申请内存的api

mp_calloc：通过mp_malloc申请内存后置零，相当于calloc

mp_free：释放由mp_malloc返回的内存

mp_reset_pool：将block的last置为初始状态，销毁所有大块内存

monitor_mp_poll：监控内存池状态
```

