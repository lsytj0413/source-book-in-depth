# Go 中的反射 #

Golang中的reflect包提供了一些必要的API，可以根据正在处理的对象的类型来更改程序的控制流. reflect包提供了两个重要的类型——Type和Value.

- Type是一个Go类型的表示，可以用于编码任何的Go类型
- Value是Go语言中值的表示，可以用于编码任何值

## Type 和 Kind ##

Type 和 Kind 有一个小小的区别，考虑以下代码：

```
func demo01() {
	type example struct {
		field1 string
		field2 int
	}
	
	r := reflect.TypeOf(example{})
	fmt.Println(r.String())
	fmt.Println(r.Kind())
	
	type VersionType string
	var Version VersionType
	r = reflect.TypeOf(Version)
	fmt.Println(r.String())
	fmt.Println(r.Kind())
}
```

输出为：

```
main.example
struct
main.VersionType
string
```

可以看到，Kind可以视为Type的Type，在Go中所有的结构体都是相同的Kind，不同的Type。

- 复合类型，例如Pointer，Array等具有不同的Type和Kind
- 简单类型，例如int，float等具有相同的Type和Kind

## 从Type创建对象 ##

一个对象同时具有Type和Value，我们可以从 reflect.Type 创建一个同类型的对象。

### 从原始对象的零值创建对象 ###

对于简单类型，reflect包提供了Zero函数创建该类型的零值对象：

```
func demo02() {
	v := reflect.Zero(reflect.TypeOf(int(1000)))
	fmt.Println(v.Type().Name())
	fmt.Println(v.Type().Kind())
	fmt.Println(v.Int())
}
```

### 提取对应的值 ###

reflect.Zero 返回的是 reflect.Value 类型，如果需要进一步提取对应类型的值，需要调用响应的函数进一步处理。例如：

```
func demo03() {
	v := reflect.Zero(reflect.TypeOf(int32(1000)))
	
	switch v.Type().Kind() {
	case reflect.Int32:
		fmt.Println(int32(v.Int()))
	case reflect.String:
		fmt.Println(v.String())
	}
}
```

### 指针值 ###

Go 中有Uintptr和UnsafePointer两种指针类型, 他们都是一个代表虚拟地址的uint值. Uintptr将由Go runtime进行类型检查, 而UnsafePointer不会, 可以通过UnsafePointer在任意的Go类型之间转换, 只要他们的内存布局是相同的.

可以通过reflect.Value的Addr和UnsafeAddr函数来得到对应类型的值, 需要注意的是UnsafeAddr返回的是一个uintptr类型的值, 需要进一步转换为unsafe.Pointer.

## 进一步解释 Type 和 Kind ##

假设有如下代码:

```
type VersionType string
var Version VersionType
```

其中Version的Type是VersionType, Kind是string. 可以理解为: Type 是程序中定义的类型元数据, Kind是编译器和Go runtime定义的元数据, 用于为变量分配内存布局.

## 从复合类型创建复合对象 ##

复合类型是包含其他类型的类型, 例如 Map, Struct 和 Array 等都是复合类型.

### 创建 Array ###

```
func demo04() {
	v := reflect.Zero(reflect.TypeOf([4]int{}))
	fmt.Println(v.Type().String())
	
	arrType := reflect.ArrayOf(5, reflect.TypeOf(1))
	v = reflect.Zero(arrType)
	fmt.Println(v.Type().String())
	
	switch v.Type().Kind() {
	case reflect.Array:
		fmt.Println(v.Interface().([5]int))
	}
}
```

### 创建 Channel ###

```
func demo05() {
	var ch chan int
	v := reflect.Zero(reflect.TypeOf(ch))
	fmt.Println(v.Type().String())
	
	chanType := reflect.ChanOf(reflect.BothDir, reflect.TypeOf(1))
	v = reflect.MakeChan(chanType, 100)
	fmt.Println(v.Type().String())
	
	switch v.Type().Kind() {
	case reflect.Chan:
		var ch chan int
		ch = v.Interface().(chan int)
		ch <- 5
		fmt.Println(<- ch)
	}
}
```

### 创建函数 ###

不能使用 reflect.Zero 函数来创建函数对象.

```
func doubler(input int) int {
	return input * 2
}

func doublerReflect(args []reflect.Value) []reflect.Value {
	if len(args) != 1 {
		panic("expect 1 arg")
	}

	if args[0].Kind() != reflect.Int {
		panic("expect int arg")
	}

	var intVal int64
	intVal = args[0].Int()

	v := doubler(int(intVal))
	return []reflect.Value{reflect.ValueOf(v)}
}

func demo06() {
	fType := reflect.TypeOf(doubler)
	var ins, outs *[]reflect.Type
	ins = new([]reflect.Type)
	outs = new([]reflect.Type)

	for i := 0; i < fType.NumIn(); i++ {
		*ins = append(*ins, fType.In(i))
	}

	for i := 0; i < fType.NumOut(); i++ {
		*outs = append(*outs, fType.Out(i))
	}

	v := reflect.MakeFunc(reflect.FuncOf(*ins, *outs, fType.IsVariadic()), doublerReflect)
	input := 42
	insv := []reflect.Value{reflect.ValueOf(input)}
	outsv := v.Call(insv)
	fmt.Println(outsv[0].Interface().(int))
}
```

### 创建Map ##

```
func demo07() {
	v := reflect.Zero(reflect.TypeOf(map[string]string{}))
	fmt.Println(v.Type().String())

	mapType := reflect.MapOf(reflect.TypeOf(""), reflect.TypeOf(""))
	v = reflect.MakeMap(mapType)
	fmt.Println(v.Type().String())

	switch v.Type().Kind() {
	case reflect.Map:
		fmt.Println(v.Interface().(map[string]string))
	}
}
```

也可以使用 MakeMapWithSize 来创建具有初始大小的 map.

### 创建指针 ###

func demo08() {
	var p *int
	v := reflect.Zero(reflect.TypeOf(p))
	fmt.Println(v.Type().String())
	
	v = reflect.Zero(reflect.PtrTo(reflect.TypeOf(0)))
	fmt.Println(v.Type().String())

	var i int
	i = 100
	p = &i
	v = reflect.ValueOf(p)
	switch v.Kind() {
	case reflect.Ptr:
		fmt.Println(v.Elem().Interface().(int))
	}
}

### 创建切片 ###

```
func demo09() {
	v := reflect.Zero(reflect.TypeOf([]int{}))
	fmt.Println(v.Type().String())

	sliceType := reflect.SliceOf(reflect.TypeOf(0))
	v = reflect.MakeSlice(sliceType, 5, 100)

	switch v.Kind() {
	case reflect.Slice:
		var s []int
		s = v.Interface().([]int)
		fmt.Println(len(s), "  ", cap(s))
	}
}
```

### 创建结构体 ###

```
func demo10() {
	type example struct {
		Field1 string
		Field2 int
	}

	v := reflect.Zero(reflect.TypeOf(example{}))
	fmt.Println(v.Type().String())

	typ := reflect.StructOf([]reflect.StructField{
		{
			Name: "Field1",
			Type: reflect.TypeOf(""),
			Tag:  "",
		},
		{
			Name: "Field2",
			Type: reflect.TypeOf(int(0)),
			Tag:  "",
		},
	})
	v = reflect.New(typ).Elem()
	v.Field(0).SetString("field1")
	v.Field(1).SetInt(2)
	s := v.Addr().Interface()
	fmt.Printf("%v\n", s)
}
```

# 参考资料 #

- [Go Reflection](https://medium.com/kokster/go-reflection-creating-objects-from-types-part-i-primitive-types-6119e3737f5d)