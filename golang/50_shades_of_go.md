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

func main() {
	fmt.Println("hello there!")
}
```

*注意: 你应该使用 gofmt 来自动的格式化你的代码.*

### 第二条: 未使用的变量 ###

如果你的代码中含有未使用的变量将导致编译失败. 在函数中定义的局部变量必须被使用, 而未使用的全局变量和函数及函数参数则没有影响, 仅仅只是对变量重新赋值也会被视为未使用的变量.

一个会导致该错误的示例如下:

```
package main

import (
	"fmt"
)

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

用 := 方式进行声明的变量只能用于函数内部.

```
package main

// myvar := 1        // error
var myvar = 1

func main() {
	myvar2 := 1
	_ = myvar2
}
```

### 第五条: 使用短变量声明重新声明变量 ###

不能在独立语句中重新声明变量, 但是在多变量声明中允许声明至少一个新变量. 注意重新声明的变量必须位于同一个块中, 否则作用域会覆盖外部变量.

```
package main

import (
	"fmt"
)

var two int

func print2(three int) {
	fmt.Println(three)
	fmt.Println(two)
}

func main() {
	one := 0
	// one := 1      // error
	one, two := 1, 2

	one, two = two, one
	fmt.Println(two)
	print2(two)
}

// output: 1 1 0
```

### 第六条: 不能使用短变量声明来设置字段值 ###

使用短变量声明来设置字段值会导致编译错误.

```
package main

import (
	"fmt"
)

type info struct {
	result int
}

func work() (int, error) {
	return 13, nil
}

func main() {
	var data info

	// data.result, err := work() //error
	var err error
	data.result, err = work()
	_ = err
	fmt.Printf("info: %+v\n", data)
}
```

### 第七条: 意外的变量作用域覆盖 ###

使用短变量声明语法时很容易意外覆盖其他的变量(可以见第五条).

```
package main

import "fmt"

func main() {
	x := 1
	fmt.Println(x) //prints 1
	{
		fmt.Println(x) //prints 1
		x := 2
		fmt.Println(x) //prints 2
	}
	fmt.Println(x) //prints 1 (bad if you need 2)
}
```

*注意: 你可以使用 vet 命令来查找这种问题. 默认情况下 vet 不会执行作用域覆盖检查, 可以使用 -shadow 标识启用该检查.*

*注意: vet 命令不会报告所有的作用域覆盖问题, 可以使用 go-nyet 提供更多的检测.*

### 第八条: 不能使用 nil 来初始化一个没有显式类型的变量 ###

nil 标识符可以用作接口, 函数, 指针, map, slice 和 channel 的零值. 如果声明变量时不指定变量类型则会导致编译失败, 因为编译器无法猜测变量的类型.

```
package main

func main() {
	// var x = nil          //error
	var x interface{} = nil

	_ = x
}
```

### 第九条: 使用值为 nil 的 slice 和 map ###

向值为 nil 的 slice 增加元素是可以的, 但是对值为 nil 的 map 进行存操作会导致 panic.

```
package main

func main() {
	var s []int
	s = append(s, 1)

	// var m map[string]int
	// m["one"] = 1         // panic
}
```

### 第十条: map 的容量 ###

可以在创建 map 的时候指定容量, 但是不能用 cap 函数获取 map 的容量.

```
package main

func main() {
	m := make(map[string]int, 99)
	_ = m
	// cap(m)     //  error
}
```

### 第十一条: string 类型的值不能为 nil ###

string 类型的零值是空字符串, 且不能被赋值为 nil.

```
package main

func main() {
	// var x string = nil //error
	// if x == nil { //error
	//     x = "default"
	// }

	var x string //defaults to "" (zero value)
	if x == "" {
		x = "default"
	}
}
```

### 第十二条: 数组类型的函数参数 ###

在 Go 中数组是值语义, 如果将数组传递给函数时, 函数会获得原始数组数据的副本, 此时对参数数组的修改并不会反应到原始数组中. 如果需要修改原始数组, 可以使用数组指针或者 slice.

```
package main

import "fmt"

func main() {
	x := [3]int{1, 2, 3}

	func(arr [3]int) {
		arr[0] = 7
		fmt.Println(arr) //prints [7 2 3]
	}(x)

	fmt.Println(x) //prints [1 2 3] (not ok if you need [7 2 3])

	func(arr *[3]int) {
		(*arr)[0] = 7
		fmt.Println(arr) //prints &[7 2 3]
	}(&x)

	fmt.Println(x) //prints [7 2 3]

	func(arr []int) {
		arr[0] = 7
		fmt.Println(arr) //prints [7 2 3]
	}(x[:])

	fmt.Println(x) //prints [7 2 3]
}
```

### 第十三条: 注意将 range 子句用于 slice 获得的值 ###

Go 中的 range 子句会产生两个值, 第一个值是索引, 第二个是元素值.

```
package main

import "fmt"

func main() {
	x := []string{"a", "b", "c"}

	for i, v := range x {
		fmt.Println(i) //prints 0, 1, 2
		fmt.Println(v) //prints a, b, c
	}
}
```

### 第十四条: slice 是一维的 ###

在 Go 中不直接支持类似 C 语言的多维数组, 需要使用将一维的数组或 slice 模拟为多维数组的方式进行处理.

```
package main

import (
	"fmt"
)

func main() {
	// 第一种方式: 使用独立的 slice 模拟多维数组
	x := 2
	y := 4

	table := make([][]int, x)
	for i := range table {
		table[i] = make([]int, y)
	}

	// 第二种方式: 使用共享的 slice 模拟多维数组
	h, w := 2, 4

	raw := make([]int, h*w)
	for i := range raw {
		raw[i] = i
	}
	fmt.Println(raw, &raw[4])
	//prints: [0 1 2 3 4 5 6 7] <ptr_addr_x>

	table2 := make([][]int, h)
	for i := range table2 {
		table2[i] = raw[i*w : i*w+w]
	}

	fmt.Println(table2, &table2[1][0])
	//prints: [[0 1 2 3] [4 5 6 7]] <ptr_addr_x>
}
```

使用第一种方式时第二维数组还可以动态的增长; 使用第二种方式时要注意内容可能被覆盖的问题.

### 第十五条: 获取 map 中不存在的 key 值 ###

map 的访问操作会返回两个值, 第二个值用于标识该 key 是否存在与 map 中. 一般应该避免使用返回一个零值的方式来判断 key 是否存在.

```
package main

import "fmt"

func main() {
	x := map[string]string{"one": "a", "two": "", "three": "c"}

	if v := x["two"]; v == "" { //incorrect
		fmt.Println("no entry")
	}

	x1 := map[string]string{"one": "a", "two": "", "three": "c"}

	if _, ok := x1["two"]; !ok {
		fmt.Println("no entry")
	}
}
```

### 第十六条: string 是不可变的 ###

尝试使用索引操作符去更新字符串变量中的单个字符将导致失败, 字符串是只读的字节 slice. 如果确实需要更新字符串, 可以将它转换为 []byte 再更新.

```
package main

import "fmt"

func main() {
	x := "text"
	// x[0] = 'T'      // error

	xbytes := []byte(x)
	xbytes[0] = 'T'
	fmt.Println(string(xbytes))
}
```

*注意: 这不是真正的更新字符的正确方法, 因为给定的字符可能存储在多个字节中. 如果需要更新字符可以将其转换为 []rune.*

### 第十七条: string 和 []byte 之间的转换 ###

### 第十八条: string 和索引操作符 ###

### 第十九条: string 并不总是 utf-8 格式的文本 ###

### 第二十条: 注意 string 的长度 ###

### 第二十一条: 在多行的 slice, array 和 map 字面量中缺少逗号 ###

### 第二十二条: log.Fatal 和 log.Panic 不仅仅是记日志 ###

### 第二十三条: 内置数据结构的操作不是协程安全的 ###

### 第二十四条: 将 range 子句用于 string 获得的值 ###

### 第二十五条: 使用 for-range 子句迭代 map ###

### 第二十六条: switch 语句中的 fallthrough 行为 ###

### 第二十七条: 自增和自减 ###

### 第二十八条: 按位 NOT 运算 ###

### 第二十九条: 运算符优先级差异 ###

### 第三十条: 结构体中的未导出字段不会被编码 ###

### 第三十一条: 程序不会主动等待所有 goroutine 退出 ###

### 第三十二条: 发送到无缓冲的 channel 操作会等待接收者就绪才会返回 ###

### 第三十三条: 向关闭的 channel 发送值会导致 panic ###

### 第三十四条: 使用值为 nil 和 channel ###

### 第三十五条: 值对象的方法不能修改原值 ###

## 中级初学者 ##

### 第三十六条: 注意关闭 http 响应的 body ###

### 第三十七条: 注意关闭 http 连接 ###

### 第三十八条: 将 JSON 中的数值解码为 interface ###

### 第三十九条: 比较结构体, 数组, slice 和 map 类型的值 ###

### 第四十条: 从 panic 中恢复 ###

### 第四十一条: 在 range 子句中更新和引用值 ###

### 第四十二条: 在 slice 中隐藏数据 ###

### 第四十三条: 意外的 slice 数据覆写 ###

### 第四十四条: slice 的持久化 ###

### 第四十五条: 类型定义和方法 ###

### 第四十六条: 跳出 for-switch 和 for-select 语句 ###

### 第四十七条: for 语句中的迭代变量和闭包 ###

### 第四十八条: defer 语句中的函数参数求值时机 ###

### 第四十九条: defer 语句的执行规则 ###

### 第五十条: 失败的类型断言 ###

### 第五十一条: 阻塞的 goroutines 和资源泄露 ###

## 高级初学者 ##

### 第五十二条: 在值实例上使用指针方法 ###

### 第五十三条: 更新 map 的值 ###

### 第五十四条: nil interface 和 nil interface 值 ###

### 第五十五条: 堆变量和栈变量 ###

### 第五十六条: GOMAXPROCS, 并发和并行 ###

### 第五十七条: 读操作和写操作的重排序 ###

### 第五十八条: 抢占式调度 ###

# 参考资料 #

- [50 Shades of Go: Traps, Gotchas, and Common Mistakes for New Golang Devs](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html)