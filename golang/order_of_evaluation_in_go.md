# Golang 中的求值顺序 #

Golang 支持在同一行中定义多个变量, 并且变量可以有不同的类型:

```
var i int
var U, V, W float64
var k = 0
var x, y float32 = -1, -2
var (
	i       int
	u, v, s = 2.0, 3.0, "bar"
)
var re, im = complexSqrt(-1)
var _, found = entries[name]  // map lookup; only interested in "found"
```

但是这样却给我们带来了一个问题, 那就是如下的代码的结果是如何呢?

```
n0, n1 = n0 + n1, n0
```

## 官方文档 ##

在 Golang 的[文档](https://tip.golang.org/ref/spec#Order_of_evaluation) 中描述了求值顺序的定义:

1. 在同一个包中, 变量声明的求值顺序按照依赖顺序来确定
2. 在非变量初始化中按照从左到右的顺序进行求值

例如如下代码:

```
y[f()], ok = g(h(), i()+x[j()], <-c), k()
```

求值顺序按照 **f(), h(), i(), j(), <-c, g(), k()** 的顺序进行求值. 但是对 x 和 y 的索引求值顺序是未定义的. 例如如下代码:

```
	a := 1
	f := func() int { a++; return a }
	x := []int{a, f()} // x may be [1, 2] or [2, 2]: evaluation order between a and f() is not specified
	m := map[int]int{a: 1, a: 2} // m may be {2: 1} or {2: 2}: evaluation order between the two map assignments is not specified
	n := map[int]int{a: f()} // n may be {2: 3} or {3: 3}: evaluation order between the key and the value is not specified
```

在如上的代码中, x, m 和 n 的值是未定义的.

在求值的过程中, 如果有依赖于其他未初始化变量的值, 它会在该依赖初始化之后再求值.

```
var a, b, c = f() + v(), g(), sqr(u()) + v()

func f() int        { return c }
func g() int        { return a }
func sqr(x int) int { return x*x }

// functions u and v are independent of all other variables and functions
```

如上代码的求值顺序是: **u(), sqr(), v(), f(), v(), g()**.

更多内容可以查看[理解Golang语句中的求值顺序](https://tonybai.com/2015/08/27/understanding-go-statements-evaluating-order/#comments).