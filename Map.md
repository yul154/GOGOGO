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


在执行扩容动作的时候，可以发现都是以 bucket/oldbucket 为单位的，而不是传统的 buckets/oldbuckets。再结合代码分析，可得知在 Go map 中扩容是采取增量扩容的方式，并非一步到位

根据需扩容的原因不同（overLoadFactor/tooManyOverflowBuckets），分为两类容量规则方向，为等量扩容（不改变原有大小）或双倍扩容


## `sync.map`

适用情况：它在读多写少的情况下，比较适合，可以保证并发安全，同时又不需要锁的开销，在写多读少的情况下反而可能会更差

```
type Map struct {
	mu Mutex
	read atomic.Value // readOnly
	dirty map[any]*entry
	misses int
}
```
sync.Map里有两个map一个是专门用于读的read map，另一个是提供读写的dirty map；优先读read map，若不存在则加锁穿透读dirty map，同时记录一个未从read map读到的计数，当计数到达一定值，就将read map用dirty map进行覆盖

![image](https://github.com/yul154/GOGOGO/assets/27160394/171158f0-bfa2-421d-82f3-910d0c3e208e)


* mutex锁，当涉及到脏数据(dirty)操作时候，需要使用这个锁
* read，读不需要加锁，就是从read中读的，read是atomic.Value类型


基本原理
* 通过 read 和 dirty 两个字段实现数据的读写分离，读的数据存在只读字段 read 上，将最新写入的数据则存在 dirty 字段上
* 读取时会先查询 read，不存在再查询 dirty，写入时则只写入 dirty
* 读取 read 并不需要加锁，而读或写 dirty 则需要加锁
* 另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据更新到 read 中（触发条件：misses=len(dirty)）

read的数据存在readOnly.m中，也是个map，value是个entry的指针，entry是个结构体具体类型如下
* 当我们设置一个key的value时，可以理解为p就是指向这个value的指针
* 当`readOnly.amended = true`的时候，表示read的数据不是最新的，dirty里面包含一些read没有的新key
* Map的dirty也是map类型，从命名来看它是脏的，可以理解某些场景新加kv的时候，会先加到dirty中，它比read要新
* Map的misses，当从read中没读到数据，且amended=true的时候，会尝试从dirty中读取，并且misses会加1，当misssed数量大于等于dirty的长度的时候，就会把dirty赋给read，同时重置missed和dirty

```
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[any]*entry
	amended bool // true if the dirty map contains some key not in m.
}


type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted, and either m.dirty == nil or
	// m.dirty[key] is e.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p unsafe.Pointer // *interface{}
}
```

### Store (新增或者更新一个kv)

![image](https://github.com/yul154/GOGOGO/assets/27160394/57e5faef-e8e9-4c05-9af1-901096910228)

```
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		e.storeLocked(&value)
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}
```

1.  如果m.read存在这个key，并且没有被标记删除，则尝试更新
    - 这里面有个tryStore,tryStore里面有判断p == expunged就返回false 
2. 如果read不存在或者已经被标记删除: 加锁，接下来都是线程安全的。
3. 加完锁有了，所以要再check下
    - 如果read中entry已删除且被标记为expunge，则表明dirty没有key，可添加入dirty，并更新entry
    - 如果read中没有key，但是dirty中有，那么直接修改value
    - 如果read和dirty中都没有这个key，`read.amended == false`，将read中未删除的数据加入到dirty中，amended标记为read与dirty不相同，因为后面即将加入新数据。
7. 然后在dirty中设置新的k、v。（这里可以发现新的k、v都是先加在dirty的map中的，read是没有的）。
8. 现在dirty是比较干净的数据了（已经清空了nil或expunged的key），设置amended=true（说明此时dirty不为空，且dirty中有新数据）
9. 解锁

```
expunged 是一个特殊的标记，用于表示 entry 中的值已经被删除。 并且那个 key 在 dirty map 中已经不存在了
```

### Load（获取一个kv）

![image](https://github.com/yul154/GOGOGO/assets/27160394/94f3ff88-61be-4e05-80a4-37d4d3327a9d)

1. 首先，进入fast path，直接在read中进行查找，如果找到了，就调用entry的load方法，取出p的值
2. 如果read中没有这个key，且amended为false，说明dirty为nil，那么直接返回nil和false
3. 如果read中没有这个key，且amended为true 加锁
4. 又在 readonly 中检查一遍，因为在加锁的时候 dirty 的数据可能已经迁移到了read中
5. read 还没有找到，并且dirty中有数据
6. 通过misslock，不管有没有，先对misses +1，如果miss次数>=len(dirty)，重制dirty和misses, 把dirty copy给read，这样read的数据就是最新的了
8. 如果没有对应的key，就返回nil，有的话，就返回对应的value(dirty和read数据不一致，dirty存在但read不存在该key，直接返回dirty中数据~)

```
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    
    // 将dirty置给read，因为穿透概率太大了(原子操作，耗时很小)
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

### Delete（删除一个k）

1. 当read不存在这个key，且dirty有新数据的时候，加锁
2. 因为加锁的过程，可能read发生变化，所以再次check下
3. dirty中有新数据的时候，直接删除dirty中的k
4. 如果read有，那么就软删除，设置p为nil

在e.delete方法中，并没有真的删掉value，而是将其值改为nil，为什么是nil呢
* 因为p == expunged表示entry不在dirty中了，而此时它其实还是在的（虽然说key已经被删了）
* 而在向空的dirty第一次添加key的时候，调用m.dirtyLocked()方法时，会把nil原子地设置为expunged，因为对于新的dirty map而言，这个entry是不存在于其中的。
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
Go语言中map的查找时间复杂度为O(1)，也就是常数级别。这意味着，无论map的大小如何，查找一个键所需的时间都是相同的。

不过，需要注意的是，当map的容量不够时，map会自动进行扩容，扩容的时间复杂度为O(n)，其中n为map的大小。因此，如果您知道map的大小，并且预期map不会再次扩容，那么您可以通过使用make函数来预先分配足够的容量，以保证map不需要再次扩容
```

 Golang map Rehash 的策略是怎样的？
 什么时机会发生 Rehash？ 
 
 ```
 当哈希表中的元素数量超过了哈希表容量的 2/3 时，会触发扩容操作。此时，哈希表的容量会翻倍，并且哈希表中所有的元素都会重新哈希到新的槽位中。
当哈希表中的元素数量过少，而哈希表的容量过大时，会触发收缩操作。此时，哈希表的容量会减半，并且哈希表中所有的元素都会重新哈希到新的槽位中。
当哈希表的探测序列过长时，会触发重排操作。此时，哈希表中的元素不会重新哈希，但是它们的存储位置会发生变化，从而减少聚簇现象，提高哈希表的性能。
 ```
 
 Rehash 具体会影响什么？哈希结果会受到什么影响？ 
 
 ```
 rehash 操作会影响 Map 的性能。由于重新计算键的哈希值，rehash 操作会消耗一定的计算资源。此外，在 rehash 过程中，原始哈希表的所有键值对都需要复制到新的哈希表中，因此 rehash 操作还会消耗一定的内存空间和时间。
rehash 操作不会直接影响哈希结果。哈希结果是由哈希函数计算得出的，与 Map 中元素的数量和布局无关。rehash 操作只会影响哈希表的布局，即每个键在哈希表中的位置会发生变化，但是每个键的哈希值并不会改变。


作者：程序员祝融
链接：https://juejin.cn/post/7226153290051141692
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
 ```
 
 Rehash 过程中存放在旧桶的元素如何迁移？ 



 panic 如果并发环境想要用这种哈希容器有什么方案？ 
 ```
 sync.Mutex / sync.RWMutex 
 ```
 
 sync.Map 和加锁的区别是什么？
```
sync.Map 和使用锁的区别在于，sync.Map 不需要在并发访问时进行加锁和解锁操作。相比之下，使用锁需要在并发访问时显式地加锁和解锁，以避免竞争条件和数据竞争问题。
```
  
 sync.Map 加锁存在什么问题呢？
 
 ```
 写多读少的情况下，并不适合，因为还是涉及到频繁的加锁、read和dirty交换等开销，搞不好还比常规的map加锁性能更差
 ```
 
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
  
  
  
