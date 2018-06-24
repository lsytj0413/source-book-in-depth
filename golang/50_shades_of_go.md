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

### 第二条: 未使用的变量 ###

如果你的代码中含有未使用的变量将导致编译失败. 在函数中定义的局部变量必须被使用, 而未使用的全局变量和函数及函数参数则没有影响, 仅仅只是对变量重新赋值也会被视为未使用的变量.

一个会导致该错误的示例如下:

```
package main

var gvar int //not an error

func main() {  
    var one int   //error, unused variable
    two := 2      //error, unused variable
    var three int //error, even though it's assigned 3 on the next line
    three = 3

    func(unused string) {
        fmt.Println("Unused arg. No compile error")
    }("what?")
}
```

有两种方式可以避免该错误, 其一是删除未使用的变量, 或者可以采用如下方式:

```
package main

import "fmt"

func main() {  
    var one int
    _ = one

    two := 2 
    fmt.Println(two)

    var three int 
    three = 3
    one = three

    var four int
    four = four
}
```

### 第三条: 未使用的 import ###

如果在不使用任何导出的函数, 接口, 结构体或者变量的情况下导入 package, 则代码将无法编译. 如果确实需要导入 package, 可以使用空白标识符作为软件包名称以避免编译失败.

```
package main

import (  
    _ "fmt"
    "log"
    "time"
)

var _ = log.Println

func main() {  
    _ = time.Now
}
```

*注意: 你可以使用 goimports 来删除未使用的 import.*

### 第四条: 短变量声明只能用于函数内部 ###

### 第五条: 使用短变量声明重新声明变量 ###

### 第六条: 不能使用短变量声明来设置字段值 ###

### 第七条: 意外的变量作用域覆盖 ###

### 第八条: 不能使用 nil 来初始化一个没有显式类型的变量 ###

## 中级初学者 ##

## 高级初学者 ##

# 参考资料 #

- [50 Shades of Go: Traps, Gotchas, and Common Mistakes for New Golang Devs](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html)