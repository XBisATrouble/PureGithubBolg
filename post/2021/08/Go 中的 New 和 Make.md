## Go 中的 New 和 Make
思考来源于一道题：下列代码可以运行吗？
```
package main

type Param map[string]interface{}

type Show struct {
   Param
}

func main() {
   s := new(Show)
   s.Param["RMB"] = 10000
}
```
答案是不可以，因为 new 关键字不分配初始值，仅仅返回一片区域指针，无法操作 Param
应该改为 s := Show{Param{}}， 继而由此题总结 Go 中 New 和 Make 的坑



### 辨析二者区别
Go中new和make都是用来内存分配的原语，区别是，new 只分配内存，make 用于 slice、map 和 channel 的初始化。

![avatar](../../../static/images/2021/golang-make-and-new.png)


* new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针
* make 的作用是初始化内置的数据结构，返回的是指向结构体的指针，其本身是个引用

```
// new
i := new(int) // 返回一个指向该类型的指针

// make
slice := make([]int, 0, 100)    // slice 是一个包含 data、cap 和 len 的结构体
hash := make(map[int]bool, 10)  // hash 是一个指向 runtime.hmap 结构体的指针（语法糖成引用）
ch := make(chan int, 5)         // ch 是一个指向 runtime.hchan 结构体的指针
```

至于分配在堆还是栈上，不重要，Go中决定变量分配位置的是变量逃逸分析。

### 关键字 new
new(T) 函数是一个分配内存的内建函数，用来分配内存，并初始化零值，返回零值指针。
在编译过程中，使用 new 大致会产生 2 种情况：

1. 若该对象申请的空间为 0，则返回表示空指针的 zerobase 变量，这类对象比如：slice, map, channel 以及一些结构体等。

2. 其他情况则会使用 `runtime.newobject` 函数，调用了 `runtime.mallocgc` 函数去开辟一段内存空间，然后返回那块空间的地址

```
func main() {
   a := new(map[int]int)
    fmt.Println(*a) // nil， 参考情况 1

   b := new(int)
   fmt.Println(*b)  // 0, 参考情况 2

    var a int
    return &a         // 等价于a := new(int)
}
```

无论是直接使用 new，还是使用 var 初始化变量，它们在编译器看来都是 `ONEW` 和 `ODCL` 节点。如果变量会逃逸到堆上，这些节点在这一阶段都会通过 `runtime.object` 函数并在堆上分配。
如果通过 var 或者 new 创建的变量不需要在当前作用域外生存，例如不用作为返回值返回给调用方，那么就不需要初始化在堆上。（逃逸分析）

### 关键字make
```
func make(t Type, size ...IntegerType) Type {}
```
先上函数签名，可以看到 make 返回的是 Type，而不是 *Type，返回的还是这三个引用类型本身，但是 make 调用的 `runtime.makeslice` 等返回的都是类型指针，对应代码块2中的注释。

make严格意义上来说是golang提供给开发者的语法糖，在编译期间，make会被替换为具体的特性函数。

```
     case OMAKE:
                args := n.List.Slice()

                n.List.Set(nil)
                l := args[0]
                l = typecheck(l, Etype)
                t := l.Type

                i := 1
                switch t.Etype {
                case TSLICE:
                        ...

                case TMAP:
                        ...

                case TCHAN:
                        ...
                }

                n.Type = t
```

### 总结

Go 语言中 make 和 new 关键字，make 关键字的作用是创建 slice、hash 和 Channel 等内置的数据结构，而 new 的作用是为类型申请一片内存空间，并返回指向这片内存的指针。

### 参考
1. [5.5 make 和 new ](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/)
2. [一文理清 Go 引用的常见疑惑](https://zhuanlan.zhihu.com/p/84580859)