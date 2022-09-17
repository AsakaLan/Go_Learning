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

### 4.2map的创建方式

1. 通过make创建
2. 字面量创建

### 4.3map什么时候会发生扩容

1. 当达到负载因子(load factor)的最大极限时。
2. 溢出桶(overflow buckets)过多时。

### 4.4map扩容的类型

1. 等量扩容:数据不多但是溢出桶太多了(主要目的是对buckets中的数据进行整理)
2. 翻倍扩容:数据过多

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

**Map总结**

1. map在扩容时会有并发问题。
2. sync.Map使用了两个map，分离了读写问题。
3. 不会引发扩容的操作(查、改)使用 **read map**
4. 可能引发扩容的操作(新增) 使用 **dirty map**

































