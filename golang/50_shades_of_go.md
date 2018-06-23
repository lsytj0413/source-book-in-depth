# 常见的 50 个错误和陷阱 #

Go 是一门简单而有趣的语言, 但是像其他大多数的语言一样, Go 也有很多的陷阱, 其中一些错误是由于错误的假设和缺乏对细节的了解.

本文描述了一些常见的错误和陷阱, 以帮助您节省一些代码的调试时间.

## 初级初学者 ##

### 第一条: 左括号不能放在单独的一行 ###

在 Go 语言中, 编译器会自动的在行尾插上分号(具体的内容可见[Semicolons](https://tip.golang.org/ref/spec#Semicolons)), 这在某些情况下可能会造成问题:

```
package main

import "fmt"

func main()  
{ //error, can't have the opening brace on a separate line
    fmt.Println("hello there!")
}
```

在上例中, 左大括号另起一行之后会造成编译错误. 正确的代码如下:

```
package main

import "fmt"

func main()  {
    fmt.Println("hello there!")
}
```

*注意: 你应该使用 gofmt 来自动的格式化你的代码.*

## 中级初学者 ##

## 高级初学者 ##

# 参考资料 #

- [50 Shades of Go: Traps, Gotchas, and Common Mistakes for New Golang Devs](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html)