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

当你将 string 转换为 []byte (或者相反)时会获得一个原始值的拷贝. Go 也有一些 string 与 []byte 类型转换中的优化, 以避免额外的分配:

- 当将 []byte 用于 map 查找时: m[string(key)]
- 在 range 子句中进行转换时: for i, v := range []byte(str)

### 第十八条: string 和索引操作符 ###

在 string 上使用索引操作符会返回一个字节值.

```
package main

import "fmt"

func main() {  
    x := "text"
    fmt.Println(x[0]) //print 116
    fmt.Printf("%T",x[0]) //prints uint8
}
```

如果你需要访问字符(unicode代码点), 需要使用 range 子句. 官方的 "unicode/utf8" package 和 "golang.org/x/exp/utf8string" package 也非常有用.

### 第十九条: string 并不总是 utf-8 格式的文本 ###

string 的值并不总是 utf8 的文本, 它可以包含任意字节. 当 string 的值是字符串字面量时它一般是 utf8 字节, 即使是这时也可以使用转义字符包含其他的数据.

可以使用 "unicode/utf8" package 中的 ValidString 函数来判断 string 是否是 utf8:

```
package main

import (  
    "fmt"
    "unicode/utf8"
)

func main() {  
    data1 := "ABC"
    fmt.Println(utf8.ValidString(data1)) //prints: true

    data2 := "A\xfeC"
    fmt.Println(utf8.ValidString(data2)) //prints: false
}
```

### 第二十条: 注意 string 的长度 ###

内置函数 len 返回的是字节数而不是字符数. 可以使用 "unicode/utf8" package 的 RuneCountInString 函数来获取字符数:

```
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	data := "♥"
	fmt.Println(utf8.RuneCountInString(data)) //prints: 1

	data = "é"
	fmt.Println(len(data))                    //prints: 3
	fmt.Println(utf8.RuneCountInString(data)) //prints: 2
}
```

但是如果一个字符由多个 rune 组成则该函数也不能返回正确的字符数.

### 第二十一条: 在多行的 slice, array 和 map 字面量中缺少逗号 ###

在多行的 slice, map 和数组字面量中可能需要保留最后一个逗号.

```
package main

func main() {
	x := []int{
		1,
		// 2 //error
	}
	_ = x

	x = []int{
		1,
		2,
	}
	x = x

	y := []int{3, 4} //no error
	y = y
}
```

### 第二十二条: log.Fatal 和 log.Panic 不仅仅是记日志 ###

当调用 log.Fatal* 和 log.Panic* 系列函数时, 应用程序将会被终止.

```
package main

import "log"

func main() {
	log.Fatalln("Fatal Level: log entry") //app exits here
	log.Println("Normal Level: log entry")
}

```

### 第二十三条: 内置数据结构的操作不是协程安全的 ###

内置的数据结构, 例如 slice, map 不是协程安全的. 当需要在多个协程中访问这些数据时, 可以使用 "sync" package 来进行同步.

### 第二十四条: 将 range 子句用于 string 获得的值 ###

当将 range 子句用于 string 类型的值时, 第二个值返回的时字符, 索引值是该字符的第一个字节的位置. 注意实际的字符可能由多个 rune 组成, 更多的信息可以查看 "golang.org/x/text/unicode/norm" 包.

range 子句会尝试将数据解释为 utf8 文本, 对于它不理解的字节序列会返回 0xfffd(即 unicode 占位字符).

```
package main

import "fmt"

func main() {  
    data := "A\xfe\x02\xff\x04"
    for _,v := range data {
        fmt.Printf("%#x ",v)
    }
    //prints: 0x41 0xfffd 0x2 0xfffd 0x4 (not ok)

    fmt.Println()
    for _,v := range []byte(data) {
        fmt.Printf("%#x ",v)
    }
    //prints: 0x41 0xfe 0x2 0xff 0x4 (good)
}
```

### 第二十五条: 使用 for-range 子句迭代 map ###

Go 会尝试对 map 的序列进行随机化处理, 多次运行可能得到不同的顺序.

```
package main

import "fmt"

func main() {  
    m := map[string]int{"one":1,"two":2,"three":3,"four":4}
    for k,v := range m {
        fmt.Println(k,v)
    }
}
```

多次运行以上代码将看到键值对以不同的顺序输出.

### 第二十六条: switch 语句中的 fallthrough 行为 ###

switch 语句中的 case 块在默认情况下不会自动落入下一个 case 块, 除非你使用 fallthrough 指定.

```
package main

import "fmt"

func main() {
	isSpace := func(ch byte) bool {
		switch ch {
		case ' ': //error
		case '\t':
			return true
		}
		return false
	}

	fmt.Println(isSpace('\t')) //prints true (ok)
	fmt.Println(isSpace(' '))  //prints false (not ok)
}
```

### 第二十七条: 自增和自减 ###

Go 语言不支持自增和自减的前缀版本, 也不能在表达式中使用.

```
package main

import "fmt"

func main() {
	data := []int{1, 2, 3}
	i := 0
	// ++i //error
	// fmt.Println(data[i++]) //error

	i++
	fmt.Println(data[i])
}
```

### 第二十八条: 按位 NOT 运算 ###

Go 使用 ^ 作为一元 NOT 运算符, ^ 也是二元 XOR 运算符, 并且使用 &^ 作为 AND NOT 运算符.

```
package main

import "fmt"

func main() {
	var a uint8 = 0x82
	var b uint8 = 0x02
	fmt.Printf("%08b [A]\n", a)
	fmt.Printf("%08b [B]\n", b)

	fmt.Printf("%08b (NOT B)\n", ^b)
	fmt.Printf("%08b ^ %08b = %08b [B XOR 0xff]\n", b, 0xff, b^0xff)

	fmt.Printf("%08b ^ %08b = %08b [A XOR B]\n", a, b, a^b)
	fmt.Printf("%08b & %08b = %08b [A AND B]\n", a, b, a&b)
	fmt.Printf("%08b &^%08b = %08b [A 'AND NOT' B]\n", a, b, a&^b)
	fmt.Printf("%08b&(^%08b)= %08b [A AND (NOT B)]\n", a, b, a&(^b))
}
```

### 第二十九条: 运算符优先级差异 ###

一些运算符的优先级差异如下:

```
package main

import "fmt"

func main() {  
    fmt.Printf("0x2 & 0x2 + 0x4 -> %#x\n",0x2 & 0x2 + 0x4)
    //prints: 0x2 & 0x2 + 0x4 -> 0x6
    //Go:    (0x2 & 0x2) + 0x4
    //C++:    0x2 & (0x2 + 0x4) -> 0x2

    fmt.Printf("0x2 + 0x2 << 0x1 -> %#x\n",0x2 + 0x2 << 0x1)
    //prints: 0x2 + 0x2 << 0x1 -> 0x6
    //Go:     0x2 + (0x2 << 0x1)
    //C++:   (0x2 + 0x2) << 0x1 -> 0x8

    fmt.Printf("0xf | 0x2 ^ 0x2 -> %#x\n",0xf | 0x2 ^ 0x2)
    //prints: 0xf | 0x2 ^ 0x2 -> 0xd
    //Go:    (0xf | 0x2) ^ 0x2
    //C++:    0xf | (0x2 ^ 0x2) -> 0xf
}
```

### 第三十条: 结构体中的未导出字段不会被编码 ###

在 struct 中的未导出字段(小写开头) 是不会被 (json, xml, gob 等)编码的, 当解码时这些字段将被设置为零值.

```
package main

import (
	"encoding/json"
	"fmt"
)

type MyData struct {
	One int
	two string
}

func main() {
	in := MyData{1, "two"}
	fmt.Printf("%#v\n", in) //prints main.MyData{One:1, two:"two"}

	encoded, _ := json.Marshal(in)
	fmt.Println(string(encoded)) //prints {"One":1}

	var out MyData
	json.Unmarshal(encoded, &out)

	fmt.Printf("%#v\n", out) //prints main.MyData{One:1, two:""}
}
```

### 第三十一条: 程序不会主动等待所有 goroutine 退出 ###

在 Go 中应用程序不会主动等待所有的 goroutines 完成:

```
package main

import (
	"fmt"
	"time"
)

func main() {
	workerCount := 2

	for i := 0; i < workerCount; i++ {
		go doit(i)
	}
	time.Sleep(1 * time.Second)
	fmt.Println("all done!")
}

func doit(workerId int) {
	fmt.Printf("[%v] is running\n", workerId)
	time.Sleep(3 * time.Second)
	fmt.Printf("[%v] is done\n", workerId)
}

// [0] is running
// [1] is running
// all done!
```

一种常见的解决方案是使用 WaitGroup 来等待 goroutine 的退出, 对于有消息处理循环的 goroutine, 可以向每个 goroutine 发送一个退出消息, 或者关闭它接收消息的 channel.

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	done := make(chan struct{})
	workerCount := 2

	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go doit(i, done, wg)
	}

	close(done)
	wg.Wait()
	fmt.Println("all done!")
}

func doit(workerId int, done <-chan struct{}, wg sync.WaitGroup) {
	fmt.Printf("[%v] is running\n", workerId)
	defer wg.Done()
	<-done
	fmt.Printf("[%v] is done\n", workerId)
}

// [1] is running
// [1] is done
// [0] is running
// [0] is done
// fatal error: all goroutines are asleep - deadlock!
```

上面的代码看似正确的, 但是还包含一个隐藏的导致死锁的错误. 错误原因是每个 goroutine 都获得了原始 WaitGroup 变量的副本, 当在 goroutine 中执行 wg.Done() 时对原始变量没有影响. 正确版本的代码如下:

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	done := make(chan struct{})
	wq := make(chan interface{})
	workerCount := 2

	for i := 0; i < workerCount; i++ {
		wg.Add(1)
		go doit(i, wq, done, &wg)
	}

	for i := 0; i < workerCount; i++ {
		wq <- i
	}

	close(done)
	wg.Wait()
	fmt.Println("all done!")
}

func doit(workerId int, wq <-chan interface{}, done <-chan struct{}, wg *sync.WaitGroup) {
	fmt.Printf("[%v] is running\n", workerId)
	defer wg.Done()
	for {
		select {
		case m := <-wq:
			fmt.Printf("[%v] m => %v\n", workerId, m)
		case <-done:
			fmt.Printf("[%v] is done\n", workerId)
			return
		}
	}
}
```

现在的代码会按照我们预期的方式工作.

### 第三十二条: 发送到无缓冲的 channel 操作会等待接收者就绪才会返回 ###

对于没有缓冲的 channel, 发送者在接收者接收消息之后就会继续运行, 这可能导致接收者没有足够的时间来处理消息.

```
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		for m := range ch {
			fmt.Println("processed:", m)
		}
	}()

	ch <- "cmd.1"
	ch <- "cmd.2" //won't be processed
}
```

### 第三十三条: 向关闭的 channel 发送值会导致 panic ###

从关闭的 channel 接收数据是安全的, 获取的第二个值会被设置为 false 代表未读取到数据. 而将数据发送到关闭的 channel 会导致 panic.

```
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)
	for i := 0; i < 3; i++ {
		go func(idx int) {
			ch <- (idx + 1) * 2
		}(i)
	}

	//get the first result
	fmt.Println(<-ch)
	close(ch) //not ok (you still have other senders)
	//do other work
	time.Sleep(2 * time.Second)
}
```

### 第三十四条: 使用值为 nil 的 channel ###

向值为 nil 的 channel 写入或读取数据会导致阻塞.

```
package main

import (
	"fmt"
	"time"
)

func main() {
	var ch chan int
	for i := 0; i < 3; i++ {
		go func(idx int) {
			ch <- (idx + 1) * 2
		}(i)
	}

	//get first result
	fmt.Println("result:", <-ch)
	//do other work
	time.Sleep(2 * time.Second)
}

```

### 第三十五条: 值对象的方法不能修改原值 ###

方法就像普通函数一样, 如果它是声明为值对象的方法那得到的就是对象的副本.

```
package main

import "fmt"

type data struct {
	num   int
	key   *string
	items map[string]bool
}

func (this *data) pmethod() {
	this.num = 7
}

func (this data) vmethod() {
	this.num = 8
	*this.key = "v.key"
	this.items["vmethod"] = true
}

func main() {
	key := "key.1"
	d := data{1, &key, make(map[string]bool)}

	fmt.Printf("num=%v key=%v items=%v\n", d.num, *d.key, d.items)
	//prints num=1 key=key.1 items=map[]

	d.pmethod()
	fmt.Printf("num=%v key=%v items=%v\n", d.num, *d.key, d.items)
	//prints num=7 key=key.1 items=map[]

	d.vmethod()
	fmt.Printf("num=%v key=%v items=%v\n", d.num, *d.key, d.items)
	//prints num=7 key=v.key items=map[vmethod:true]
}
```

## 中级初学者 ##

### 第三十六条: 注意关闭 http 响应的 body ###

当使用标准的 http 库进行请求时, 需要关闭响应的 Body, 即使你没有读取它或者返回的是空响应.

```
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	resp, err := http.Get("https://api.ipify.org?format=json")
	if err != nil {
		fmt.Println(err)
		return
	}

	defer resp.Body.Close() //ok, most of the time :-)
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(string(body))
}
```

很多时候请求获得的返回值是 nil 的 resp 和非 nil 的 err. 但是当得到一个重定向的失败时, 这两个值都会出现非 nil 的情况. 需要注意这种情况下的资源泄露.

在原始的 resp.Body.Close() 实现中会读取并丢弃剩余的数据, 这确保了对于使用了 keepalive 的连接可以用于其他的 http 请求. 但在最新的 http 库中没有了这个行为, 用户需要读取并丢弃剩余的数据, 否则 http 连接可能会被关闭而不是被重用. 可以使用如下的代码来实现这个行为:

```
_, err = io.Copy(ioutil.Discard, resp.Body)
```

### 第三十七条: 注意关闭 http 连接 ###

在一些 http 服务器会使用 keepalive 来保持连接, 默认情况下标准的 http 库会在服务端要求关闭连接时关闭, 这表示你可以遇到意料之外的内存耗尽错误.

可以通过将 http 请求体的 Close 字段设置为 true 来要求关闭连接, 或者将请求头中的 Connection 设置为 close 来达到同一效果.

```
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	req, err := http.NewRequest("GET", "http://golang.org", nil)
	if err != nil {
		fmt.Println(err)
		return
	}

	req.Close = true
	//or do this:
	//req.Header.Add("Connection", "close")

	resp, err := http.DefaultClient.Do(req)
	if resp != nil {
		defer resp.Body.Close()
	}

	if err != nil {
		fmt.Println(err)
		return
	}

	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(len(string(body)))
}
```

也可以在全局设置关闭连接:

```
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	tr := &http.Transport{DisableKeepAlives: true}
	client := &http.Client{Transport: tr}

	resp, err := client.Get("http://golang.org")
	if resp != nil {
		defer resp.Body.Close()
	}

	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(resp.StatusCode)

	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(len(string(body)))
}
```

如果你需要向同一个服务器发送大量的请求, 那么保持连接的重用是合适的; 如果在短时间内需要连接多个不同的服务器, 那么主动关闭连接是合适的, 或者提高打开文件的数量限制.

### 第三十八条: 将 JSON 中的数值解码为 interface ###

默认的情况下, Go 将 JSON 中的数值解码为 float64 类型的值, 如果你没有显示指定类型或者指定的是 interface 的话.

```
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	var data = []byte(`{"status": 200}`)

	var result map[string]interface{}
	if err := json.Unmarshal(data, &result); err != nil {
		fmt.Println("error:", err)
		return
	}

	var status = result["status"].(int)   // panic
	fmt.Println("status value:", status)
}
```

有以下几个方式来处理这个问题:

1. 转换为 float64 类型: result["status"].(float64)
2. 将浮点值转换为需要的类型: uint64(result["status"].(float64))
3. 使用 Decoder:

```
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
)

func main() {
	var data = []byte(`{"status": 200}`)

	var result map[string]interface{}
	var decoder = json.NewDecoder(bytes.NewReader(data))
	decoder.UseNumber()

	if err := decoder.Decode(&result); err != nil {
		fmt.Println("error:", err)
		return
	}

	var status, _ = result["status"].(json.Number).Int64() //ok
	fmt.Println("status value:", status)
}
```

4. 使用 struct 来解码

```
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
)

func main() {
	var data = []byte(`{"status": 200}`)

	var result struct {
		Status uint64 `json:"status"`
	}

	if err := json.NewDecoder(bytes.NewReader(data)).Decode(&result); err != nil {
		fmt.Println("error:", err)
		return
	}

	fmt.Printf("result => %+v", result)
	//prints: result => {Status:200}
}
```

5. 使用 json.RawMessage

```
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
)

func main() {
	records := [][]byte{
		[]byte(`{"status": 200, "tag":"one"}`),
		[]byte(`{"status":"ok", "tag":"two"}`),
	}

	for idx, record := range records {
		var result struct {
			StatusCode uint64
			StatusName string
			Status     json.RawMessage `json:"status"`
			Tag        string          `json:"tag"`
		}

		if err := json.NewDecoder(bytes.NewReader(record)).Decode(&result); err != nil {
			fmt.Println("error:", err)
			return
		}

		var sstatus string
		if err := json.Unmarshal(result.Status, &sstatus); err == nil {
			result.StatusName = sstatus
		}

		var nstatus uint64
		if err := json.Unmarshal(result.Status, &nstatus); err == nil {
			result.StatusCode = nstatus
		}

		fmt.Printf("[%v] result => %+v\n", idx, result)
	}
}
```

### 第三十九条: 比较结构体, 数组, slice 和 map 类型的值 ###

当 struct 的每一个属性都可以使用 == 进行比较时, struct 也可以使用 == 进行比较. 对数组来说, 如果其中的元素可以进行比较那么数组也可以进行比较.

更好一点的方式是使用 reflect.DeepEqual 进行比较:

```
package main

import (
	"fmt"
	"reflect"
)

type data struct {
	num    int               //ok
	checks [10]func() bool   //not comparable
	doit   func() bool       //not comparable
	m      map[string]string //not comparable
	bytes  []byte            //not comparable
}

func main() {
	v1 := data{}
	v2 := data{}
	fmt.Println("v1 == v2:", reflect.DeepEqual(v1, v2)) //prints: v1 == v2: true

	m1 := map[string]string{"one": "a", "two": "b"}
	m2 := map[string]string{"two": "b", "one": "a"}
	fmt.Println("m1 == m2:", reflect.DeepEqual(m1, m2)) //prints: m1 == m2: true

	s1 := []int{1, 2, 3}
	s2 := []int{1, 2, 3}
	fmt.Println("s1 == s2:", reflect.DeepEqual(s1, s2)) //prints: s1 == s2: true
}
```

但是 DeepEqual 函数比较慢, 而且也有一些自身的问题:

```
package main

import (
	"encoding/json"
	"fmt"
	"reflect"
)

func main() {
	var b1 []byte = nil
	b2 := []byte{}
	fmt.Println("b1 == b2:", reflect.DeepEqual(b1, b2)) //prints: b1 == b2: false

	var str string = "one"
	var in interface{} = "one"
	fmt.Println("str == in:", str == in, reflect.DeepEqual(str, in))
	//prints: str == in: true true

	v1 := []string{"one", "two"}
	v2 := []interface{}{"one", "two"}
	fmt.Println("v1 == v2:", reflect.DeepEqual(v1, v2))
	//prints: v1 == v2: false (not ok)

	data := map[string]interface{}{
		"code":  200,
		"value": []string{"one", "two"},
	}
	encoded, _ := json.Marshal(data)
	var decoded map[string]interface{}
	json.Unmarshal(encoded, &decoded)
	fmt.Println("data == decoded:", reflect.DeepEqual(data, decoded))
	//prints: data == decoded: false
}
```

1. DeepCopy 不认为 nil slice 等于空的 slice, 这和 bytes.Equal 的行为是不同的

```
package main

import (
	"bytes"
	"fmt"
)

func main() {
	var b1 []byte = nil
	b2 := []byte{}
	fmt.Println("b1 == b2:", bytes.Equal(b1, b2)) //prints: b1 == b2: true
}
```

2. 当 DeepCopy 对 slice 进行对比时可能得到预期之外的结果

当对 []byte 或者 string 进行忽略大小写的比较时, 应该先使用 ToUpper 或 ToLower 函数进行处理(在使用 ==, bytes.Equal 和 bytes.Compare 之前); 这只对英文有效, 对其他类型的文字应该使用 bytes.EqualFold 和 strings.EqualFold 函数.

如果你需要对用户提供的密码或其他机密数据进行验证, 不要使用 reflect.DeepEqual, bytes.Equal 和 bytes.Compare, 这些函数可以导致 [时序攻击](https://en.wikipedia.org/wiki/Timing_attack), 为了避免泄露定时信息, 应该使用 "crypto/subtle" package 中的函数, 例如 subtle.ConstantTimeCompare.

### 第四十条: 从 panic 中恢复 ###

recover 函数可以用来捕获 panic 并进行恢复, 注意 recover 函数只有直接在 defer 中调用才能生效.

```
package main

import "fmt"

func main() {
	defer func() {
		fmt.Println("recovered:", recover())
	}()

	panic("not good")
}
```

### 第四十一条: 在 range 子句中更新和引用值 ###

在 range 子句中生成的数据是实际元素的副本, 不是对实际元素的引用.

```
package main

import "fmt"

func main() {
	data := []int{1, 2, 3}
	for _, v := range data {
		v *= 10 //original item is not changed
	}

	fmt.Println("data:", data) //prints data: [1 2 3]
}
```

### 第四十二条: 在 slice 中隐藏数据 ###

当对 slice 重新进行切片处理时, 新的 slice 会引用原 slice 的底层数组. 这可能会导致意料之外的内存泄露.

```
package main

import "fmt"

func get() []byte {
	raw := make([]byte, 10000)
	fmt.Println(len(raw), cap(raw), &raw[0]) //prints: 10000 10000 <byte_addr_x>
	return raw[:3]
}

func get2() []byte {
	raw := make([]byte, 10000)
	fmt.Println(len(raw), cap(raw), &raw[0]) //prints: 10000 10000 <byte_addr_x>
	res := make([]byte, 3)
	copy(res, raw[:3])
	return res
}

func main() {
	data := get()
	fmt.Println(len(data), cap(data), &data[0]) //prints: 3 10000 <byte_addr_x>

	data = get2()
	fmt.Println(len(data), cap(data), &data[0]) //prints: 3 10000 <byte_addr_x>
}
```

为了避免内存泄露, 可以从临时切片中重新复制数据以释放引用.

### 第四十三条: 意外的 slice 数据覆写 ###

假设你需要重写一个存储在 slice 中的路径:

```
package main

import (
	"bytes"
	"fmt"
)

func main() {
	path := []byte("AAAA/BBBBBBBBB")
	sepIndex := bytes.IndexByte(path, '/')
	dir1 := path[:sepIndex]
	dir2 := path[sepIndex+1:]
	fmt.Println("dir1 =>", string(dir1)) //prints: dir1 => AAAA
	fmt.Println("dir2 =>", string(dir2)) //prints: dir2 => BBBBBBBBB

	dir1 = append(dir1, "suffix"...)
	path = bytes.Join([][]byte{dir1, dir2}, []byte{'/'})

	fmt.Println("dir1 =>", string(dir1)) //prints: dir1 => AAAAsuffix
	fmt.Println("dir2 =>", string(dir2)) //prints: dir2 => uffixBBBB (not ok)

	fmt.Println("new path =>", string(path))
}
```

如上的代码不会得到预期的结果, 因为 dir1 和 dir2 都引用了原始的数组数据, 当对 dir1 和 dir2 进行修改的同时也修改了原始的数据.

可以通过分配新的切片并复制数据解决以上问题, 或者使用完整的切片表达式:

```
package main

import (
	"bytes"
	"fmt"
)

func main() {
	path := []byte("AAAA/BBBBBBBBB")
	sepIndex := bytes.IndexByte(path, '/')
	dir1 := path[:sepIndex:sepIndex] //full slice expression
	dir2 := path[sepIndex+1:]
	fmt.Println("dir1 =>", string(dir1)) //prints: dir1 => AAAA
	fmt.Println("dir2 =>", string(dir2)) //prints: dir2 => BBBBBBBBB

	dir1 = append(dir1, "suffix"...)
	path = bytes.Join([][]byte{dir1, dir2}, []byte{'/'})

	fmt.Println("dir1 =>", string(dir1)) //prints: dir1 => AAAAsuffix
	fmt.Println("dir2 =>", string(dir2)) //prints: dir2 => BBBBBBBBB (ok now)

	fmt.Println("new path =>", string(path))
}
```

切片表达式的第三个参数控制新切片的容量, 当追加数据到切片时会触发重新分配内存, 而不是覆盖原始数据.

### 第四十四条: slice 的持久化 ###

当多个切片引用相同的数据时, 要小心意外的数据覆盖:

```
package main

import "fmt"

func main() {
	s1 := []int{1, 2, 3}
	fmt.Println(len(s1), cap(s1), s1) //prints 3 3 [1 2 3]

	s2 := s1[1:]
	fmt.Println(len(s2), cap(s2), s2) //prints 2 2 [2 3]

	for i := range s2 {
		s2[i] += 20
	}

	//still referencing the same array
	fmt.Println(s1) //prints [1 22 23]
	fmt.Println(s2) //prints [22 23]

	s2 = append(s2, 4)

	for i := range s2 {
		s2[i] += 10
	}

	//s1 is now "stale"
	fmt.Println(s1) //prints [1 22 23]
	fmt.Println(s2) //prints [32 33 14]
}
```

### 第四十五条: 类型定义和方法 ###

从现有非接口类型定义新类型时, 新类型不会继承现有类型的方法集.

```
package main

import "sync"

type myMutex sync.Mutex

func main() {
	var mtx myMutex
	mtx.Lock()   //error
	mtx.Unlock() //error
}
```

如果需要原始类型得方法, 可以将原始类型作为嵌入得匿名字段.

```
type myLocker struct {  
    sync.Mutex
}
```

如果现有类型为接口, 则新类型会保留方法集:

```
package main

import "sync"

type myLocker sync.Locker

func main() {
	var lock myLocker = new(sync.Mutex)
	lock.Lock()   //ok
	lock.Unlock() //ok
}
```

### 第四十六条: 跳出 for-switch 和 for-select 语句 ###

break 语句可以使用标签跳出指定的循环:

```
package main

import "fmt"

func main() {
loop:
	for {
		switch {
		case true:
			fmt.Println("breaking out...")
			break loop
		}
	}

	fmt.Println("out!")
}
```

也可以使用 goto 语句达到同一效果.

### 第四十七条: for 语句中的迭代变量和闭包 ###

for 语句中的迭代变量在每次迭代中都会被重用, 这表示在 for 循环中创建的每个闭包都会引用相同的变量:

```
package main

import (
	"fmt"
	"time"
)

func main() {
	data := []string{"one", "two", "three"}

	for _, v := range data {
		go func() {
			fmt.Println(v)
		}()
	}

	time.Sleep(3 * time.Second)
	//goroutines print: three, three, three
}
```

有以下的方案可以解决该问题:

1. 将当前迭代变量的值复制一份

```
	for _, v := range data {
		vcopy := v //
		go func() {
			fmt.Println(vcopy)
		}()
	}
```

2. 将当前迭代变量作为参数传递给 goroutine, 也相当于复制一份

```
	for _, v := range data {
		go func(in string) {
			fmt.Println(in)
		}(v)
	}
```

以下代码也存在一个同样原因的错误:

```
package main

import (
	"fmt"
	"time"
)

type field struct {
	name string
}

func (p *field) print() {
	fmt.Println(p.name)
}

func main() {
	data := []field{{"one"}, {"two"}, {"three"}}

	for _, v := range data {
		go v.print()
	}

	time.Sleep(3 * time.Second)
	//goroutines print: three, three, three
}
```

但是思考下如下的代码是否是正确的呢?

```
package main

import (
	"fmt"
	"time"
)

type field struct {
	name string
}

func (p *field) print() {
	fmt.Println(p.name)
}

func main() {
	data := []*field{{"one"}, {"two"}, {"three"}}

	for _, v := range data {
		go v.print()
	}

	time.Sleep(3 * time.Second)
}
```

### 第四十八条: defer 语句中的函数参数求值时机 ###

defer 语句将在本语句被执行时对函数参数进行求值:

```
package main

import "fmt"

func main() {
	var i int = 1

	defer fmt.Println("result =>", func() int { return i * 2 }())
	i++
	//prints: result => 2 (not ok if you expected 4)
}
```

### 第四十九条: defer 语句的执行规则 ###

defer 函数会在当前函数结束时执行, 而不是在包含的代码块退出时执行, 这和变量的作用域规则时不同的. 因此当你有一个长期执行的函数, 并在 for 循环里面使用 defer, 可能会造成资源的清理延后:

```
package main

import (
	"fmt"
	"os"
	"path/filepath"
)

func main() {
	if len(os.Args) != 2 {
		os.Exit(-1)
	}

	start, err := os.Stat(os.Args[1])
	if err != nil || !start.IsDir() {
		os.Exit(-1)
	}

	var targets []string
	filepath.Walk(os.Args[1], func(fpath string, fi os.FileInfo, err error) error {
		if err != nil {
			return err
		}

		if !fi.Mode().IsRegular() {
			return nil
		}

		targets = append(targets, fpath)
		return nil
	})

	for _, target := range targets {
		f, err := os.Open(target)
		if err != nil {
			fmt.Println("bad target:", target, "error:", err) //prints error: too many open files
			break
		}
		defer f.Close() //will not be closed at the end of this code block
		//do something with the file...
	}
}
```

### 第五十条: 失败的类型断言 ###

当类型断言失败时会返回目标类型的零值:

```
package main

import "fmt"

func main() {
	var data interface{} = "great"

	if data, ok := data.(int); ok {
		fmt.Println("[is an int] value =>", data)
	} else {
		fmt.Println("[not an int] value =>", data)
		//prints: [not an int] value => 0 (not "great")
	}
}
```

### 第五十一条: 阻塞的 goroutines 和资源泄露 ###

有如下一段代码:

```
func First(query string, replicas ...Search) Result {  
    c := make(chan Result)
    searchReplica := func(i int) { c <- replicas[i](query) }
    for i := range replicas {
        go searchReplica(i)
    }
    return <-c
}
```

上面的代码在有多个 Search 时会造成 goroutine 泄露的问题. 因为使用的 channel 是无缓冲的, 更多的 goroutine 会阻塞在向 channel 发送数据的地方.

为了避免泄露, 需要保证所有的 goroutine 都会退出, 有如下几个方式:

1. 使用一个含有足够大的缓存区的 channel
2. 使用 select 避免发送阻塞

```
func First(query string, replicas ...Search) Result {
	c := make(chan Result, 1)
	searchReplica := func(i int) {
		select {
		case c <- replicas[i](query):
		default:
		}
	}
	for i := range replicas {
		go searchReplica(i)
	}
	return <-c
}
```

3. 使用取消 channel 通知 goroutine 退出

```
func First(query string, replicas ...Search) Result {
	c := make(chan Result)
	done := make(chan struct{})
	defer close(done)
	searchReplica := func(i int) {
		select {
		case c <- replicas[i](query):
		case <-done:
		}
	}
	for i := range replicas {
		go searchReplica(i)
	}

	return <-c
}
```

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