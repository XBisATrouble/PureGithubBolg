## Go 中的数组与切片
在 Go 中有数组(Array)和切片(Slice)两种数据结构。

数组是一种具有固定长度的基本数据结构，在 Go 中与 C 语言一样数组一旦创建了它的长度就不允许改变，数组的空余位置用0填补，不允许数组越界。切片是基于数组的实现，是长度动态不固定的数据结构，本质上是一个对数组字序列的引用，提供了对数组的轻量级访问，类似于 STL 中的动态数组。

### Array 底层
Go 的数组 array 底层和 C 的数组一样，是一段连续的内存空间，通过下标访问数组中的元素。array 只有长度 len 属性而且是固定长度的。一旦创建了它的长度就不允许改变，数组的空余位置用 0 填补，不允许数组越界。

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

- slice 的重要知识点:
  - slice 的底层是数组指针。
  - 当 append 后，slice 长度不超过容量 cap，新增的元素将直接加在数组中。
  - 当 append 后，slice 长度超过容量 cap，将会返回一个新的 slice。

### Slice 中的坑
#### Slice作为形参
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
    - Go 中只有值传递，Channel、Hash 和 Map 的make会返回对应的指针（引用类型），所以实际是操作引用
    - Slice有所不同，他返回的是底层 array 的指针，也可以达到引用效果，但是 cap 和 len 就只是值了。
    - **引用类型不是传引用**

#### Slice扩容
```
func main() {
    s := make([]int, 0, 4)
    s = append(s, 1, 2, 3)
    fmt.Println(s, len(s), cap(s)) // (3) 输出：[1, 2, 3] 3 4
    s = append(s, 4)
    fmt.Println(s, len(s), cap(s)) // (4) 输出：[1, 2, 3, 4] 4 4
}
```