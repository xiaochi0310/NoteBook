Go 内存管理(一)前言

Go 内存管理(一)TCMalloc内存管理原理

##### 一、TCMalloc

Go内存管理是基于TCMalloc基础上进行设计的，所以在学习Go内存管理之前先学习TCMalloc原理

TCMalloc(Thread Cache Malloc)是线程级别的内存管理模式。

TCMalloc优势：

1、速度快    

2、减少锁竞争。对于小对象，只有在对应线程分配的空闲块不足的时候，才会使用到锁；对于大对象，TCMalloc尝试使用有效的自旋锁

总结来说就是：最大化内存使用率，最小化分配时间。

![1610590804111](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610590804111.png)

上图来自：https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-16

本文也参考该文学习的，写的非常的详细，有兴趣的可以点击学习~（多像大佬学习）

基本把这张图搞懂了，TCMalloc理解的也没啥问题了。

~提醒，图从右开始看

**图中一些名词的解释：**

1、Pages 

Pages是TCMalloc管理的内存基本单位，默认大小是8KB

2、span

一个或者多个Pages组成一个span，TCMolloc以span为单位向系统申请内存。span是由PageHeap进行管理的，可以被拆分成多个相同的page size用于小对象使用；也可以作为一个整体被中大对象进行使用。

![1610941978975](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610941978975.png)

申请内存，分裂span；回收内存，合并span。

3、size class

每一个size class都对应着不同空闲块的大小 ，例如8字节、16字节等。共有(1B~256KB)分为85个类别。

4、ThreadCache 

为每个线程分配的单独申请的内存，申请和释放该区域内存的时候无需加锁。可以看到ThreadCache是由多个class组成的，不同class size 有单独的空闲链表。

5、CentralCache

当ThreadCache没有空闲对象的时候，会像CentralCache进行申请。CentralCache指的是所有线程公用的缓存，

既然是线程公用的缓存，所以在申请的使用自旋锁。可以看到CentralCache也是以size class不同大小进行分类。

6、PageHeap

完成Page和span的映射。CentralCache内存不够时，会向PageHeap进行申请。PageHeap基本单位是span，CentralCache会将span拆分成size class大小进行使用。

从图中可以看到PageHeap分成2种，小于等于128 list都按照链表来进行缓存管理；超过128的存储在一个有序的set。

7、VirtualMemory

虚拟内存，用户申请内存实际对虚拟内存进行申请。(而不是物理内存)



##### 二、内存分配与回收

对照本文第一张图看更容易理解。

1）小对象内存分配以及内存回收原理：（小对象大小：(0, 256KB]）

分配：当一个线程申请内存的时候，将要分配的内存大小映射到对应的size class，(无需加锁)查看ThreadCache中size class对应的FreeList。若ThreadCache的FreeList有空闲对象，则返回一个空闲对象，分配结束；若ThreadCache没有空闲对象的时候，向CentralCache中对应的class size获取对象，CentralCache是所以线程共享的，所以需要自旋锁，若有可用对象，将分配的class size放到ThreadCache的FreeList中，返回对象，分配结束；如果CentralCache也没有可用的对象，向PageHeap申请一个span，将span拆分成class size放到CentralCache的freeList中。

回收：根据申请内存地址计算页号，通过页号找到对应的span，通过span知道对应的size class，若没超过ThreadCache的阈值（2MB)，则使用垃圾回收机制移动到CentralCache

2）中对象内存分配以及内存回收原理：（中对象大小：(256KB, 1MB])

分配：在PageHeap中的span list顺序选择一个非空链表M(n个page)，然后按照内存大小将M分成2类，一种是满足大小的k个page,返回对象，分配结束。另外一种的n-k的page会继续放在n-kpage的span list中。
若PageHeap没有合适的空闲块时，就按照大对象内存分配进行分配

回收：根据申请内存地址计算页号，通过页号找到对应的span，寻找到对应的span大小，进行回收

3）大对象内存分配以及内存回收原理：（大对象大小：(1MB, +∞)）

分配：在PageHeap中的span set，选取最新的span进行分配(n个page)，也是分成2类，一种是满足大小的k个page,返回对象，分配结束。另外一种的n-k的，若n-k>128,将剩下的page放在span set中，其他会继续放在n-k个page的span list中。

回收：根据申请内存地址计算页号，通过页号找到对应的span，寻找到对应的span大小，进行回收，若没有对应的大小，则继续放在span set中



##### 三、内存碎片处理

内存碎片就是不能再分配给应用使用。分配内部碎片和外部碎片，内部碎片就是内部碎片是分配器分配的内存大于程序申请的内存，内部产生碎片；外部碎片就是内存块太小，不足以分配给应用使用。

对于TCMalloc是怎么处理内部碎片和外部碎片的？

**内部碎片**：

TCMalloc提前分配了多种size-class：8， 16， 32， 48， 64， 80， 96， 112， 128， 144， 160， 176…

TCMalloc的目标就是产生最多12.5%的内存碎片。可以看到上面不是按照2的幂级数分配的大小，这是因为如果按照2的幂产生的碎片会更大。比如申请65字节，2幂申请的话会分配128，而按照TCMalloc只分配80，相应的减少了很多碎片。

16字节以内，每8字节划分一个size class：8,16

16~128字节，每16字节划分一个size class：32,48,64…

128B~256字节，按照每次增加x/8进行增加：128+128/8=144 以此类推

大于大于1024的 size-class 其实都以128对齐：

**外部碎片**：

TCMalloc的CentralCache向PageHeap申请内存的时候，是以Page为单位进行申请的。当申请1024的时候，

1page(8192)%1024=0没有内存碎片，当时当申请class-size为1152的时候（8192%1152=128）产生128的外部碎片，为了使得内存碎片率最多12.5%，可以多申请几个Page来解决。也就是合并相邻的Page，可以减少外部碎片。

TCMalloc也考虑相同的class-size进行合并，这里的相同就是指分配的对象大小相同，取一个碎片更少的size进行使用。



学习了TCMalloc的内存管理的原理，下一文我们学习Go语言的内存管理原理。





> 参考文章
>
> http://blog.itpub.net/15480802/viewspace-1452219/
>
> https://blog.csdn.net/junlon2006/article/details/77854898
>
> https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-16
>
> http://www.360doc.com/content/13/0915/09/8363527_314549128.shtml
>
> https://blog.csdn.net/kelvin_yin/article/details/78997953

