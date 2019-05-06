# Concurrency in Go #

## An Introduction to Concurrency ##

### 摩尔定律 ###

在1965年，摩尔发表了一篇3页的论文, 预言半导体芯片上集成的晶体管和电阻数量将每年增加一倍; 1975年时, 他把摩尔定律修正为每两年增加一倍. 这个预测的准确度一直持续到了2012年.

很多公司预见到了摩尔定律的放缓, 为了进一步提高计算机的性能, 他们开发了多核CPU. 但是多核时代也有自己的限制: 阿姆达尔定律, 
表明在多核时代并发程序的开发或者说提升程序的并发度仍然具有十分重要的意义.

### 为什么并发很难 ###

#### Race Conditions ####

如果多个操作需要执行同样的路径或访问同样的变量, 但代码没有保障操作之间的执行顺序时就可能会出现竞态条件. 例如如下的代码示例1:

```
var data int
go func() {
    data++
}
if data == 0 {
    fmt.Println("the value is %v.\n", data)
}
```

在上面的代码中有多处访问data这个变量的操作, 有如下3中可能的输出:

- 没有任何输出, data++先于if执行
- 输出0, data++后于Println执行
- 输出1, data++后于if执行, 但是先于Println执行

#### 原子性 ####

原子性是指一个操作是不可中断的, 在提到原子性的时候有以下几个点需要注意:

- 一个原子性的操作是关乎上下文的, 例如在你的代码中的原子性操作对于操作系统来说可能就不是原子性的
- 在上下文中, 一个原子性操作是整体完成的, 中途不会有其他操作

#### 内存访问同步 ####

当两个并发的操作访问同一块内存时, 需要注意对内存的访问应该是原子性的.

#### 死锁,活锁和饥饿 ####

##### 死锁 #####

死锁是指所有的并发进程都在等待另外的进程, 在这种情况下程序不会自动恢复. 例如如下的示例代码2:

```
type value struct {
    mu sync.Mutex
    value int
}

var wg sync.WaitGroup
printSum := func(v1 *value, v2 *value) {
    defer wg.Done()
    v1.mu.Lock()
    defer v1.mu.Unlock()

    time.Sleep(2*time.Second)
    v2.mu.Lock()
    defer v2.mu.Unlock()

    fmt.Printf("sum=%v\n", v1.value + v2.value)
}

var a, b value
wg.Add(2)
go printSum(&a, &b)
go printSum(&b, &a)
wg.Wait()
```

在出现死锁时一般都是满足Coffman Condition, 这些条件如下:

- 相互排斥: 一个操作会持有一个资源的排他访问权
- 等待条件: 一个操作持有一个资源的排他访问权时去等待获取另一个资源的访问权
- 不可抢占: 一个操作持有的访问权不会被其他操作抢占
- 循环等待: 一个操作P1等待另一个操作P2, 而且P2又在等待P1

##### 活锁 #####

活锁是指任务或者执行者没有被阻塞, 但是由于某些条件没有满足而导致的一直重复尝试的过程. 例如如下的实例代码3:

```
cadence := sync.NewCond(&sync.Mutex{})
go func() {
    for range time.Tick(1*time.Millisecond) {
        cadence.Broadcast()
    }
}

takeStep := func() {
    cadence.L.Lock()
    cadence.Wait()
    cadence.L.Unlock()
}

tryDir := func(dirName string, dir *int32, out *bytes.Buffer) bool {
    fmt.Fprintf(out, " %v", dirName)
    atomic.AddInt32(dir, 1)
    takeStep()
    if atomic.LoadInt32(dir) == 1 {
        fmt.Fprint(out, ". Success!")
        return true
    }
    takeStep()
    atomic.AddInt32(dir, -1)
    return false
}

var left, right int32
tryLeft := func(out *bytes.Buffer) bool {
    return tryDir("left", &left, out)
}
tryRight := func(out *bytes.Buffer) bool {
    return tryDir("right", &right, out)
}

walk := func(walking *sync.WaitGroup, name string) {
    var out bytes.Buffer
    defer func() {
        fmt.Println(out.String())
    }()
    defer walking.Done()

    fmt.Fprintf(&out, "%v is trying to scoot:", name)
    for i := 0; i < 5; i++ {
        if tryLeft(&out) || tryRight(&out) {
            return
        }
    }
    fmt.Fprintf(&out, "%\n%v tosses her hand up in exasperation!", name)
}

var peopelInHallway sync.WaitGroup
peopelInHallway.Add(2)
go walk(&peopelInHallway, "Alice")
go walk(&peopelInHallway, "Barbara")
peopelInHallway.Wait()
```

##### 饥饿 #####

饥饿是指一个操作不能获取到所有所需资源去完成任务, 活锁是饥饿的一种特殊情况.

## Modeling Your Code: Communicating Sequential Processes ##

### 并发和并行的区别 ###

```
Concurrency is a property of the code; parallelism is a property of the running program.
```

### What is CSP ###

CSP是指Communicating Sequential Processes, 在上个世纪70年代提出, 用于描述两个独立的并发实体通过共享的通讯管道进行通信的并发模型.