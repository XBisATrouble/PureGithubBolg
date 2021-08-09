## Go 中的数组与切片
在 Go 中有数组(Array)和切片(Slice)两种数据结构。

数组是一种具有固定长度的基本数据结构，在 Go 中与 C 语言一样数组一旦创建了它的长度就不允许改变，数组的空余位置用0填补，不允许数组越界。
切片是基于数组的实现，是长度动态不固定的数据结构，本质上是一个对数组字序列的引用，提供了对数组的轻量级访问，类似于 STL 中的动态数组。

### Array 底层
Go 的数组 array 底层和 C 的数组一样，是一段连续的内存空间，通过下标访问数组中的元素。
array 只有长度 len 属性而且是固定长度的，一旦创建了它的长度就不允许改变，数组的空余位置用 0 填补，不允许数组越界。

array 的赋值是值拷贝，下代码，因为值拷贝的原因，c 的修改并没有影响到 d 。
```
func main() {
    c := [3]int{1, 2, 3}
    d := c
    c[0] = 999
    fmt.Println(d) // 输出[1, 2, 3]
}
```

### Slice 底层
```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
slice 是一个特殊的引用类型,但是它自身也是个结构体，属性 len 表示可用元素数量,读写操作不能超过这个限制,不然就会 panic，属性 cap 表示最大扩张容量,当然这个扩张容量也不是无限的扩张,它是受到了底层数组 array 的长度限制,超出了底层 array 的长度就会 panic，**slice 的数据存在数组当中**。

![avatar](../../../static/images/2021/2019-02-20-golang-slice-struct.png)

- slice 的重要知识点:
  - slice 的底层是数组指针。
  - 当 append 后，slice 长度不超过容量 cap，新增的元素将直接加在数组中。
  - 当 append 后，slice 长度超过容量 cap，将会返回一个新的 slice。

### Slice 中的坑
#### Slice作为形参
slice作为形参传入，可以直接进行修改

```
func Assign0(s []int) {
	s[0] = 0
}

func Assign1(s []int) {
	s = []int{6, 6, 6}
}
func main() {
    s := []int{1, 2, 3, 4}
    Assign0(s)
    fmt.Println(s) // (1) 输出：0 2 3 4
    Assign1(s)
    fmt.Println(s) // (2) 输出：0 2 3 4
}
```
(1)：函数签名中的形参s作为值传递进入函数，但是 Slice 本身是一种特殊的引用，故函数内部切片中的底层数组和 main 函数中指向相同，所以修改对外部起作用。
(2)：同样，切片作为引用进行值传递进函数，但当 s 进行 []int{} 声明之后，底层的数组已经指向其他位置，外部输出不会改变。

- Tips：
    - Go 中只有值传递，Channel、Hash 和 Map 的 make 会返回对应的引用类型本身。
    - Slice有所不同，他返回的是底层 array 的指针，也可以达到引用效果。
    - **引用类型不是传引用**

#### Slice扩容
slice如果涉及到append，警惕扩容陷阱
```
func main() {
    s := make([]int, 0, 1)
    s = append(s, 1, 2)
    fmt.Println(s, len(s), cap(s)) // 输出：[1, 2] 2 2
    s = append(s, 3)
    fmt.Println(s, len(s), cap(s)) // (3) 输出：[1, 2, 3] 3 4
}
```
(3)：初始化容量为 1，长度为 0 的 slice，append 两次后发生了扩容，容量低于 1024 都是双倍扩容

- Tips：
    - 如果函数中使用make初始slice，最好**指定容量，避免多次扩容**

```
func main() {
    s := []int{1, 2, 3}
    fmt.Println(s, len(s), cap(s)) // 输出：[1, 2, 3] 3 3
    a := s // 将 s 值传递给 a，传递后公用一个底层 array

    s = append(s, 4) // 超过了原来数组的容量
    s[0] = 999
    fmt.Println(s, len(s), cap(s)) // (4) 输出：[999, 2, 3, 4] 4 6
    fmt.Println(a, len(a), cap(a)) // (5) 输出：[1, 2, 3] 3 3
}
```
(5)：扩容之后，可以看到底层array已经改变，所以 s 的修改不会改变 a

### 结论
1. Go 的 slice 类型中包含了一个 array 指针以及 len 和 cap 两个 int 类型的成员。
2. Go 中的参数传递实际都是值传递，将 slice 作为参数传递时，函数中会创建一个 slice 参数的副本，这个副本同样也包含 array, len, cap这三个成员。
3. 副本中的array指针与原slice指向同一个地址，所以当修改副本slice的元素时，原slice的元素值也会被修改。但是如果修改的是副本slice的len和cap时，原slice的len和cap仍保持不变。
4. 如果在操作副本时由于扩容操作导致重新分配了副本slice的array内存地址，那么之后对副本slice的操作则完全无法影响到原slice，包括slice中的元素。

### 参考资料
1. [3.2 切片](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/)
2. [Go Slice探秘](https://juejin.cn/post/6844904177022271501)
3. [Golang中切片和数组的简单介绍](https://km.woa.com/group/17746/articles/show/472634?kmref=search&from_page=1&no=1)
4. [Go 切片绕坑指南](https://segmentfault.com/a/1190000020994159)