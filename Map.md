# Go Map

Map最主要的数据结构有两种：哈希查找表（Hash table）、搜索树（Search tree）。

哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）
- 开销主要在哈希函数的计算以及数组的常数访问时间。在很多场景下，哈希查找表的性能很高。
- 哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：链表法和开放地址法。
  - 链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。
  - 开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

- 搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树。



Go 语言map的实现，底层采用的是哈希查找表，并且使用链表解决哈希冲突

## 基本数据结构

![image](https://user-images.githubusercontent.com/27160394/237041087-e929ff67-7daf-4bfe-a876-a78a83061263.png)

map中比较核心的几个结构分别是hmap、mapextra、bmap


hmap是最顶层的一个对应map的结构，
```
// A header for a Go map.
type hmap struct {
       count     int //map容量，实际上就是len()返回的数值
       flags     uint8 //map全局控制标记，比如写标记位等
       B         uint8 //buckets的数量=2^B
       noverflow uint16 //overflow 的 bmap 近似数
       hash0     uint32 //哈希种子，计算key的哈希的时候会传入哈希函数

       buckets    unsafe.Pointer //指向buckets数组，大小为2^B，若元素个数为0，就为 nil
       oldbuckets unsafe.Pointer //扩容时保存之前buckets的字段，大小是当前buckets一半
       nevacuate  uintptr //指示扩容进度，小于此地址的buckets迁移完成       

       extra *mapextra 
}
```

buckets是指向[]bmap数组的指针，[]bmap中每个位置我们统一称为bucket

```
// A bucket for a Go map.
type bmap struct {
        // tophash generally contains the top byte of the hash value
        // for each key in this bucket. If tophash[0] < minTopHash,
        // tophash[0] is a bucket evacuation state instead.
        tophash [bucketCnt]uint8
}
```

这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构
```
type bmap struct {
    tophash  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

bucket中实际存放者bmap链表，bmap我们统一称为cell，cell里面会最多装 8 个 key-value
* 根据 key 计算出来的 hash 值的低 8 位 算出落入哪个bucket中
* 在桶内遍历相应的cell，根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入cell内的哪个位置（一个cell内最多有8个位置）
* 低8位确定哪个bucket，高8位确定bucket中遍历的cell中的哪个位置
* 每个 cell设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 cell，那就需要再构建一个 cell ，通过 overflow 指针连接起来

![image](https://user-images.githubusercontent.com/27160394/237046577-46401974-e739-4b63-a724-a9a62616e691.png)

 > 注意到 key 和 value 是各自放在一起的，并不是 key/value/key/value/... 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间
 

Extra指向mapextra结构
```
type mapextra struct {
        overflow    *[]*bmap
        oldoverflow *[]*bmap

        // nextOverflow holds a pointer to a free overflow bucket.
        nextOverflow *bmap
}
```
nextOverflow指向的是预分配的overflow bucket，预分配的用完了那么值就变成nil


### 创建

![image](https://user-images.githubusercontent.com/27160394/237049080-2e108f09-1401-4fa9-923d-265db0b2d316.png)

创建主要是`makemap`函数，这对应了make一个map的操作
1. 进行必要的内存分配相关的检查
2. 然后传入hash种子
3. 根据传入的容量和装载因子确定B的值，然后进行实际的空间分配
  * 默认创建2^B 个bucket，如果 B大于等于4，那么就预先额外创建一些overflow bucket
  * 除了最后一个overflow bucket，其余overflow bucket的overflow指针都是nil，最后一个overflow bucket的overflow指针指向bucket数组第一个元素，作为哨兵，说明到了到结尾了



### 查询
![image](https://user-images.githubusercontent.com/27160394/237049335-c22557ea-2d79-46e8-9915-4026095ed405.png)


查询是`mapaccess1()`函数，大体流程
1. 首先进行异常判断
2. 然后计算key的hash值，根据低B位确定落在那个bucket中
3. 然后遍历该bucket中的每一个cell
4. 再根据高8位确定落在cell的具体哪个位置，如果key确实对的上，返回value，如果对不上就继续下一个cell的查找。


### 赋值
> mapassign(): 这个函数返回要进行赋值的value的地址，而不直接进行赋值
1. 首先判断异常，然后计算hash值，并置写标记防止并发写
2. 根据hash值低B位确定bucket，如果正在扩容，要先主动将目标bucket迁移到新bucket中，然后重新计算落在哪个bucket中
3. 然后遍历每一个cell，这里会记录第一次为空的位置，当没有查到目标key时，就在这个位置添加
4. 跟查询类似的遍历cell的过程中如果你找到了key，返回对应value地址，如果没找到，需要添加，就要先判断是否需要扩容，如果不需要扩容，在前面记录的空位置添加，如果需要扩容，先扩容，然后再遍前面的查找过程，进行添加
5. 添加完成后，count加1，清除写标记

### 删除
> mapdelete()

1. 先进行异常判断
2. 然后跟之前查询类似，先查找到key的位置
3. 如果有就清除掉，tophash置空，count减1。


### 扩容
扩容是一个自动触发的过程，并不由用户主动调用，前面也看到过，在赋值操作过程中会判断是否符合扩容条件，符合就会触发扩容

需要有一个指标来衡量前面描述的情况，这就是装载因子。Go 源码里这样定义 装载因子
```
loadFactor := count / (2^B)
```
count 就是 map 的元素个数，2^B 表示 bucket 数量。


#### 扩容条件

1. 不是正在扩容
2. 装载因子超过阈值，源码里定义的阈值是 6.5。
3. overflow数量 > 2^15时，也即overflow数量超过32768时。

```
//扩容条件是：
//不是正在扩容 && (元素个数/bucket数超过装载因子 || 太多overflow bucket)
if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
   hashGrow(t, h)
   goto again // Growing the table invalidates everything, so try again
}
```

1.不是正在扩容
```
//oldbuckets字段不为空，即当前正在进行扩容
func (h *hmap) growing() bool {
   return h.oldbuckets != nil
}
```

2.  `overLoadFactor`
 * 每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的
 * 对于条件 2，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）
```
//装载因子=loadFactorNum/loadFactorDen，也就是13/2=6.5
//当count大于8（bucketCnt），并且count大于装载因子*桶数量，返回true
func overLoadFactor(count int, B uint8) bool {
   return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```

3.  `tooManyOverflowBuckets`
- 是对第 2 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定，你能拿我怎么办？overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难
- 对于条件 3，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。


```
//如果B小于15，则overflow数量大于2^B
//如果B大于15,则overflow数量大于2^15
//返回true
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
   if B > 15 {
      B = 15
   }
   // The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
   return noverflow >= uint16(1)<<(B&15)
}

```


1. 


----
什么是哈希函数？哈希函数有什么特性？ 

Golang 标准库中 map 的底层数据结构是什么样子的？ 

Map 的查询时间复杂度如何分析？ 

```
Go语言中map的查找时间复杂度为O(1)，也就是常数级别。这意味着，无论map的大小如何，查找一个键所需的时间都是相同的。

不过，需要注意的是，当map的容量不够时，map会自动进行扩容，扩容的时间复杂度为O(n)，其中n为map的大小。因此，如果您知道map的大小，并且预期map不会再次扩容，那么您可以通过使用make函数来预先分配足够的容量，以保证map不需要再次扩容
```

极端情况下有很多哈希冲突，Golang 标准库如何去避免最坏的查询时间复杂度？
```

```

 Golang map Rehash 的策略是怎样的？
 什么时机会发生 Rehash？ 
 Rehash 具体会影响什么？哈希结果会受到什么影响？ 
 Rehash 过程中存放在旧桶的元素如何迁移？ 

 并发环境共享同一个 map 是安全的吗？ 
 panic 如果并发环境想要用这种哈希容器有什么方案？ sync.Mutex / sync.RWMutex 

 sync.Map 加锁存在什么问题呢？
 
  sync.Map 比加锁的方案好在哪里，它的底层数据结构是怎样的？ 
  ```
  缓存 + map 组成的结构 底层 map 的操作依然是加锁的，但是读的时候使用上缓存可以增加并发性能 

  1. 利用map只读不加锁，做通过read和dirty字段区分读写操作
  2. 读时只读read，不存在还会再读dirty的可能
  ```
    
  
  sync.Map 的 Load() 方法流程？ 
  ··
  
  sync.Map Store() 如何保持缓存层和底层 Map 数据是相同的? 
  是不是每次执行修改都需要去加锁？ 
  一致性哈希了解吗？ 
