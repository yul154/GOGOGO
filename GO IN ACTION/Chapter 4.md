* 数组的内部实现和基础功能
* 使用切片管理数据集合
* 使用映射管理键值对

----

# 4.1 数组

## 内部实现
* 长度固定的数据类型
* 具有相同类型的元素的连续款
* 占用的内存是连续分配


## 声明和初始化

声明数组时需要指定内部存储的数据的类型，以及需要存储的元素的数量

```
var array [5] int
array := [5] int{10,20,30,40,50}
array := [...] int{10,20,30,40,50} //根据初始化的元素数量来确定数组的长度
array := [5] int{1:10; 2:20} // 初始化固定索引的元素值
```
* 数组里存储的数据类型和数组长度就都不能改变了。
* 如果需要存储更多的元素，就需要先创建一个更长的数组，再把原来数组里的值复制到新数组里


## 数组使用

```
array := [5]int{1,2,3,4,5}
array[2] = 10

array := [5]*int{0:new(int), 1:new(int)} // 声明五个指向整型的指针
*array[0] = 10
*array[1] = 20
```

在 Go 语言里，数组是一个值。这意味着数组可以用在赋值操作中。变量名代表整个数组， 因此，同样类型的数组可以赋值给另一个数组
```
var arr1 [5]string
arr2 := [5]string{{"Red", "Blue", "Green", "Yellow", "Pink"}
arr1 = arr2 // 值复制
```

复制数组指针，只会复制指针的值，而不会复制指针所指向的值，如代码清单 4-9 所示。
```
var arr1 [3]*string
arr2 := [3]*string(new(string), new(string), new(string))
*arr2[0] = "red"
*arr2[1] = "Blue"
*arr2[2] = "Green"
arr1 = arr2
```

## 多维数组
```
var arr [4][2]int
arr := [4][2]int{{10, 11}, {20, 21}, {30, 31}, {40, 41}}
arr := [4][2]int{1: {20, 21}, 3: {40, 41}}

var arr [2][2]int
arr[0][0] = 10
var arr3 [2]int = arr[0]
var value int = arr[0][1]
```

## 在函数间传递数组

对于大的数组，可以只传入指向数组的指针，这样只需要复制字节更少的数据而不是内存数据到栈上，
```
var arr [1e6]int
foo(&arr)
func foo(arr *[1e6]int)
```
----
# 切片
> 动态数组

* 可以按需自动增长和缩小
* 切片的动态增长是通过内置函数`append`来实现的

## 内部实现

切片有 3 个字段 的数据结构
1. 指向底层数组的指针
2. 切片访问的元素的个数（即长度）
3. 切片允许增长到的元素个数
![截屏2022-02-08 下午2 48 02](https://user-images.githubusercontent.com/27160394/152933212-b41ed57e-d12e-4e99-86ea-9389547cc9a2.png)

## 创建和初始化
### 1. make和切片的字面量
```
slice := make([]string 5)
slice := make([]int,3,5)
slice :=[]int{1.2.3}
slice := []string{99:""}

array := [3]int{1,2,3}
slice := []int{1,2,3}

var slice []int // nil 切片
slice := make([]int,0)
slice := []int()
```

### 2. 使用

赋值和切片
```
slice := []int{10, 20, 30, 40, 50}
slice[1] = 25

slice := []int{1,2,3,4,5}
newSlice := slice[1:3]
```
新的slice的长度和容量

```
slice[i:j]
length: j-i
capacity: k-i // k is original slice capacity
```

修改切片内容
```
slice := []int{1,2,3,4,5}
newSlice ：= slice[1:3]
newSlice[1] = 35 //slice的第3个元素也被修改了
```
切片增长
```
sl := []int{1, 2, 3, 4, 5}
newSl := sl[1:3]
newSl = append(newSl, 6)
```
如果切片的底层数组没有足够的可用容量，append 函数会创建一个新的底层数组，将被引用的现有的值复制到新数组里，再追加新的值

> `append`会智能地处理底层数组的容量增长。在切片的容量小于1000个元素时，总是会成倍地增加容量.一旦元素个数超过 1000，容量的增长因子会设为 1.25，也就是会每次增加 25%

三个索引
```
source := []string{"Apple", "Orange", "Plum", "Banana", "Grape"}
slice := source[2:3:4] // start from 2, length is 3-2, capacity is 4-2
```
设置长度和容量一样的好处
```
source := []string{"Apple", "Orange", "Plum", "Banana", "Grape"}
slice := source[2:3:3]
slice = append(slice, "Kiwi") // 直接追加元素，并没有修改原数组
```

迭代切片
```
slice := []int{10, 20, 30, 40}
for index, value := range slice {
  fmt.Printf("Index: %d Value: %d\n", index, value)
} 
```

多维切片
> Go 语言里使用 append 函数处理追加的方式很简明：先增长切片，再将新的整型切片赋值 给外层切片的第一个元素
```
slice := [][]int{(10),(100,200)}
slice[0] = append(slice[0], 20)
```

在函数间传递切片就是要在函数间以值的方式传递切片
* 在 64 位架构的机器上，一个切片需要 24 字节的内存：指针字段需要 8 字节，长度和容量 字段分别需要 8 字节
* 由于与切片关联的数据包含在底层数组里，不属于切片本身，所以将切片 复制到任意函数的时候，对底层数组大小都不会有影响
* 复制时只会复制切片本身，不会涉及底层数组
----
# 映射的内部实现和基础功能
> 存储一系列无序的键值对

## 内部实现 
> 映射是无序的集合，意味着没有办法预测键值对被返回的顺序。

1. 映射的散列表包含一组桶
2. 在存储、删除或者查找键值对的时候，所有操作都要先选择一个桶。
3. 把操作映射时指定的键传给映射的散列函数，就能选中对应的桶
4. 散列函数的目的是生成一个索引，这个索引最终将键值对分布到所有可用的桶里

映射使用两个数据结构来存储数据
1. 数组，内部存储的是用于选择桶的散列键的高八位值。这个数组用于区分每个键值对要存在哪个桶里。
2. 数据结构是一个字节数组，用于存储键值对

## 创建和初始化
```
dict := make(map[string]int)
dict := map[string]string{"Red": "#da1337", "Orange": "#e95a22"} //创建映射时，更常用的方法是使用映射字面量
```
* 映射的键可以是任何值,可以是内置的类型，也可以是结构类型，只要这个值可以使用`==`运算符做比较
```
dict := map[int][]string{}
```

## 使用映射 
```
colors := map[string]string{}
colors["Red"] = "#da1337"

value, exists := colors["Blue"]
if exists {
  fmt.Println(value)
} 
delete(colors, "Coral")
```

## 在函数间传递映射

在函数间传递映射并不会制造出该映射的一个副本。实际上，当传递映射给一个函数，并对这个映射做了修改时，所有对这个映射的引用都会察觉到这个修改
