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

1.赋值和切片
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
volumn: k-i // k is original slice volumn
```

