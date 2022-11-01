## 1.make和new的区别

**不同点**

1. 作用变量的类型不同，new用于给string，int，array分配内存；make用于给slice，map，channel分配内存。
2. 返回的类型不同，new返回变量的指针，make则返回变量本身。
3. new只是分配内存并清零，并没有初始化内存；make即分配内存，也初始化内存。

**相同点**

1. 都会在栈上分配内存。
2. 给变量分配内存。

## 2.array和slice的区别

**相同点**

1. 不论是array还是slice都只能存储一组相同类型的数据结构。
2. 都是通过下标来访问，并且有长度和容量，长度通过len获取，容量通过cap获取。
3. 访问不能越界。

**不同点**

1. array是定长，一旦申请则不能变化；slice是边长，可以动态变化。
2. array是值类型，slice是引用类型。

## 3.slice

### 3.1slice底层数据结构

1. ~~~go
   type slice struct{
   	array unsafe.Pointer
   	len int
   	cap int
   }
   ~~~

2. 使用方式

   1. 使用make创建
   2. 使用array创建
### 3.2slice扩容

**slice扩容遵循以下原则**

1. 如果旧的容量*2<cap，则newcap = cap
2. 否则如果
   1. oldlen<1024,newcap=oldcap*2
   2. oldlen>=1024,newcap=oldcap*1.25

### 3.3slice深拷贝和浅拷贝

**深拷贝**

拷贝的是数据本身，创建新对象与旧对象不共享内存，对新对象的操作不会影响到旧对象。实现方式有:

1. copy(slice2,slice1)
2. append赋值

**浅拷贝**

拷贝的是数据地址，只复制指针，新对象和旧对象的内存地址是一样的，会同时修改新旧对象。实现方式有:

1. slice2:=slice1

## 4.map

### 4.1map底层数据结构

~~~go
// A header for a Go map.
type hmap struct {
    count     int 
    // 代表哈希表中的元素个数，调用len(map)时，返回的就是该字段值。
    flags     uint8 
    // 状态标志（是否处于正在写入的状态等）
    B         uint8  
    // buckets（桶）的对数
    // 如果B=5，则buckets数组的长度 = 2^B=32，意味着有32个桶
    noverflow uint16 
    // 溢出桶的数量
    hash0     uint32 
    // 生成hash的随机数种子
    buckets    unsafe.Pointer 
    // 指向buckets数组的指针，数组大小为2^B，如果元素个数为0，它为nil。
    oldbuckets unsafe.Pointer 
    // 如果发生扩容，oldbuckets是指向老的buckets数组的指针，老的buckets数组大小是新的buckets的1/2;非扩容状态下，它为nil。
    nevacuate  uintptr        
    // 表示扩容进度，小于此地址的buckets代表已搬迁完成。
    extra *mapextra 
    // 存储溢出桶，这个字段是为了优化GC扫描而设计的，下面详细介绍
 }

~~~

~~~go
// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8        
    // len为8的数组
    // 用来快速定位key是否在这个bmap中
    // 一个桶最多8个槽位，如果key所在的tophash值在tophash中，则代表该key在这个桶中
}

~~~

~~~go
type bmap struct{
    tophash [8]uint8
    keys [8]keytype 
    // keytype 由编译器编译时候确定
    values [8]elemtype 
    // elemtype 由编译器编译时候确定
    overflow uintptr 
    // overflow指向下一个bmap，overflow是uintptr而不是*bmap类型，保证bmap完全不含指针，是为了减少gc，溢出桶存储到extra字段中
}
~~~

![image-20221026092616764](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221026092616764.png)

![image-20221026092709764](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221026092709764.png)

### 4.2map的创建方式

1. 通过make创建
2. 字面量创建

### 4.3map什么时候会发生扩容

1. 当达到负载因子(load factor)的最大极限时。
2. 溢出桶(overflow buckets)过多时。

### 4.4map扩容的类型

1. **等量扩容**:数据不多但是溢出桶太多了(主要目的是对buckets中的数据进行整理),所谓等量扩容，实际上并不是扩大容量，buckets数量不变，重新做一遍类似增量扩容的搬迁动作，把松散的键值对 重新排列一次，以使bucket的使用率更高，进而保证更快的存取。
2. **翻倍扩容**:数据过多,当负载因子过大时，就新建一个bucket，新的bucket长度是原来的2倍，然后旧bucket数据搬迁到新的bucket。 考虑到如果map存储了数以亿计的key-value，一次性搬迁将会造成比较大的延时，Go采用逐步搬迁策略，即每次访 问map时都会触发一次搬迁，每次搬迁2个键值对。

### 4.5map扩容步骤

**步骤1**

1. 创建一组新桶
2. oldbuckets指向原有的数组。
3. buckets 指向新数组。
4. 将map标记为扩容状态。

**步骤2**

1. 将所有的数据从旧桶驱逐到新桶。
2. 采用渐进式驱逐。
3. 每次操作一个旧桶时，将旧桶数据驱逐到新桶。
4. 读取时不进行驱逐，只判断是读取新桶还是旧桶。

**步骤3**

1. 等所有的旧桶驱逐完成后
2. 将oldbuckets进行回收。

### 4.6map的并发问题

1. map的读写具有并发问题
2. A协程在桶中读数据过程中，B协程驱逐了这个桶，则此时A协程会读到错误的数据或找不到数据。

### 4.7map并发问题的解决方案

1. 给map加锁(开销大，性能低，不提倡)
2. 使用sync.Map
   1. ![image-20221026125204660](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221026125204660.png)


**Map总结**

1. map在扩容时会有并发问题。
2. sync.Map使用了两个map，分离了读写问题。
3. 不会引发扩容的操作(查、改)使用 **read map**
4. 可能引发扩容的操作(新增) 使用 **dirty map**

## 5.G0，M0

**M0** M0表示进程启动的第一个线程，也叫主线程。它和其他的m没有什么区别，要说区别的化，它是由进程启动直接通过汇编直接复制给M0的，这个M0是全局变量，而其他的M都是runtime内自己创建的，一个g0进程只有一个m0。

**g0**  首先需要明确是每一个m都一个g0，因为每个线程都有一个系统堆栈，g0虽然也是g的结构，但和普通的g还是有区别的，最主要的差别就是在栈上的差别，g0上的栈是系统分配的栈，Linux默认是8mb,不能扩展，也不能缩小，而普通g一开始只有2kb大小，可扩展。g0上也没有任何任务函数，也没有任务状态，并且它不能被调度程序抢占。因为调度就是在g0上跑的。

**总结**

m0代表主线程，g0代表了线程的堆栈。调度都是在系统堆栈上的，也就是一定要跑在g0上。

## 6.GC什么会触发？

**内存分配量达到阈值触发GC**

每次内存分配时都会检查当前内存分配量是否达到阈值，如果达到阈值则立即启动GC

**定期触发GC**

默认情况，golang2min触发一次GC。

**手动触发**

通过runtime.gc来手动触发GC。

## 7.逃逸分析

所谓逃逸分析是指由编译器决定内存分配的位置，不需要程序员指定，函数中申请一个新的对象

* 如果分配在堆上，则函数执行结束交给GC来进行处理。
* 如果分配在栈上，则函数执行结束可自动回收。

**逃逸策略**

每当函数中申请新的对象，编译器会根据该对象是否被函数外部引用来决定是否逃逸：

* 如果函数外部没有引用，则优先放在栈中。
* 如果函数外部存在引用，则必定放在堆中

**逃逸场景**

1. 指针逃逸
2. 栈空间不足逃逸
3. 动态类型逃逸
4. 闭包引用对象逃逸

**总结**

* 栈上分配要比堆上更有效率，因为栈上分配欸大不需要gc来处理，而堆上的需要。
* 逃逸分析的目的是决定内存分配在栈上还是堆上。
* 逃逸分析是在编译阶段完成。

## Go中常用的并发控制手段

**Channel**

	1. 优点：实现简单，清晰易懂。
	1. 缺点：当需要大量创建协程，就需要同样数量的channel，而且对子协程派生出的协程不方便控制。

**WaitGroup**

​	子协程个数动态可调整，执行过程如下:

1. 启动goroutine前将计数器通过Add()将计数器设置为待启动的goroutine个数。
2. 启动goroutine后，使用Wait()方法阻塞自己，等待计数器变为0。
3. 每个goroutine执行结束通过Done()方法将计数器减1。
4. 计数器变为0后，阻塞的goroutine被唤醒。

**Context**

​	对子协程派生出来的孙子协程的控制方便。

## 为什么使用通信来共享内存？

1. 避免协程之间的竞争和数据的冲突
2. 更高级的抽象、降低开发难度、增加程序可读性
3. 模块之间更容易解耦合

## Channel发送的情形

1. 直接发送
2. 放入缓存
3. 休眠等待

![image-20221025133358600](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025133358600.png)

## 

**Channel接受的情形**

1. 有等待的G，从G接收
2. 有等待的G，从缓存接收
3. 接收缓存
4. 阻塞接收

**锁的基础**

1. atomic操作
   1. 原子操作是一种硬件层面加锁的机制，可以用于操作一个变量的时候，其他协程/线程没法访问，但只能用于简单变量的操作
2. sema锁(信号量锁)
   1. 核心是一个uint32值，含义是同时可以并发的数量
   2. 每一个sema锁对应一个SemaRoot结构体
   3. 每个SemaRoot中有一个平衡二叉树用于协程排队
3. ![image-20221025135617317](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025135617317.png)



**Muetex(互斥锁)**

![image-20221025135807502](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025135807502.png)



![image-20221025135907103](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025135907103.png)



![image-20221025135923549](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025135923549.png)







![image-20221025140150927](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025140150927.png)





![image-20221025140207356](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025140207356.png)



![image-20221025140345961](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025140345961.png)

**RWMutex(读写锁)**

![image-20221025140749588](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025140749588.png)

**WaitGroup**

![image-20221025141030879](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025141030879.png)

![image-20221025141047773](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025141047773.png)

**协程的底层结构**

![image-20221025141816407](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025141816407.png)

**线程的抽象**

![image-20221025142011765](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025142011765.png)

**单线程循环**

![image-20221025142057914](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025142057914.png)

![image-20221025142618586](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025142618586.png)

![image-20221025142819716](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221025142819716.png)
