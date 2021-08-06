## New和Make
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
答案是不可以，因为new关键字不分配初始值，仅仅返回一片区域指针，无法操作Param

应该改为 s := Show{Param{}}， 继而由此题总结Golang中New和Make的坑
