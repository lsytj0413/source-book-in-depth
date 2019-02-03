# Go语言中的5个高级测试技巧 #

Go 有一个强大的内置测试库. 如果你使用过 Go 语言, 那你应该已经知道了这一点. 在这篇文章中, 我们将讨论一些有效的技巧来帮助你在 Go 语言中进行更好的测试, 这些技巧是我们从我们的大型 Go 代码库中获得的经验, 这些技巧可以节省维护代码的时间和精力.

## 使用测试套件 ##

如果你只能从这篇文章中学到一件事, 那么就应该是: 使用测试套件. 对于那些不熟悉这种模式的人来说, 测试套件就是针对一个通用的接口开发一个测试过程, 这个测试过程可以用来对这个接口的多个实现进行测试. 下面的代码演示了如果针对不同 Thinger 的实现使用相同的测试:

```
type Thinger interface {
    DoThing(input string) (Result, error)
}

// Suite tests all the functionality that Thingers should implement
func Suite(t *testing.T, impl Thinger) {
    res, _ := impl.DoThing("thing")
    if res != expected {
        t.Fail("unexpected result")
    }
}

// TestOne tests the first implementation of Thinger
func TestOne(t *testing.T) {
    one := one.NewOne()
    Suite(t, one)
}

// TestOne tests another implementation of Thinger
func TestTwo(t *testing.T) {
    two := two.NewTwo()
    Suite(t, two)
}
```

有些读者也许已经在代码中使用了这种测试技术. 这种技巧在基于插件的系统中广泛使用, 通常是针对接口编写适用于该接口的所有实现的测试, 以确定实现是否满足行为要求.

使用这种技巧可以节省数小时, 数天甚至更多的时间. 而且, 这可以在交换两个底层系统时避免编写(很多)额外的测试, 也可以确保不会破坏应用程序的正确性. 这隐含的要求你提供一个方式来指定需要测试的实现, 使用依赖注入你可以将需要测试的实现传递给测试套件.

[这里](https://github.com/segmentio/testdemo) 提供了一个完整的例子. 虽然这个例子是故意设计的, 但是你可以想象其中的一个实现是远程数据库, 另一个实现是内存数据库.

在标准库中有一个很好的例子是 golang.org/x/net/nettest 包, 它提供了一个满足 net.Conn 的接口来进行测试验证.

## 空接口污染 ##

在 Go 语言中没有接口就没有测试.

接口在测试环境中非常重要, 因为它们是我们测试库中最强大的工具, 所以正确的使用接口是非常重要的. 包中经常导出接口提供给使用者使用, 这一般会出现两种情况: 使用者提供他们自己的实现或者该包提供自己的实现.

> The bigger the interface, the weaker the abstraction.
> -- Rob Pike, Go Proverbs

在导出之前, 应该仔细的考虑接口的定义. 开发者经常导出接口让使用者来实现他自己的行为. 相反的, 应该在文档中描述你的结构满足哪些接口, 这样你就不会在使用者和你自己的包之间创建一个强的依赖关系. errors 包就是一个很好的例子.

当我们的程序中有一个我们不想导出的接口时, 可以使用一个[内部包/子树](https://golang.org/doc/go1.4#internalpackages)来隐藏它们. 通过这种方式, 可以避免其他使用者依赖于这些接口, 因此可以灵活的改变这些接口来适应新的需求. 我们通常围绕外部依赖创建接口, 并使用依赖注入的方式来运行本地测试.

这允许使用者能够实现自己的小接口, 并且提供给自己测试. 有关这些概念的更多细节, 可以参考 [rakyll 的文章](https://rakyll.org/interface-pollution/).

## 不要导出并发原语 ##

Go 提供了易于使用的并发原语, 但是有时也会被过度使用. 我们主要关注 channel 和 sync 包. 有些时候从使用者中导出 channel 是看上去很美好的事情. 另外, 包含 sync.Mutex 而不把它作为私有成员是一个常见的错误. 当然这并不总是很糟糕的一件事, 但是在测试程序时确实会带来一些挑战.

如果你正在导出 channel 给你的使用者, 那你会为使用者带来他们不应该关心的额外的复杂度. 只要从一个包中导出了 channel, 就会为使用者的测试编写带来麻烦, 如果要做好这个测试, 使用者需要知道:

- 什么时候需要在 channel 上发送数据发送?
- 接收数据时是否有任何错误?
- 在 channel 使用完成后如何进行清理?
- 怎样才能包装出一些 API , 以避免直接调用 channel ?

考虑下面这个示例库中的一个消费队列的例子, 它读取消息并暴露一个 channel 供使用者订阅:

```
type Reader struct {...}
func (r *Reader) ReadChan() <-chan Msg {...}
```

一个使用者需要像下面这样编写他的测试:

```
func TestConsumer(t testing.T) {
    cons := &Consumer{
        r: libqueue.NewReader(),
    }
    for msg := range cons.r.ReadChan() {
        // Test thing.
    }
}
```

使用者可能会使用依赖注入并写下如下的测试代码:

```
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    for msg := range cons.r.ReadChan() {
        // Test thing.
    }
}
```

如果考虑到错误呢?

```
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    for {
        select {
        case msg := <-cons.r.ReadChan():
            // Test thing.
        case err := <-cons.r.ErrChan():
            // What caused this again?
        }
    }
}
```

现在, 我们如何生成事件来复制我们正在使用的实际的库的行为? 如果这个库只是一个简单的同步 API, 那么我们可以在客户端添加所有的并发处理, 然后使测试更为简单:

```
func TestConsumer(t testing.T, q queueIface) {
    cons := &Consumer{
        r: q,
    }
    msg, err := cons.r.ReadMsg()
    // handle err, test thing
}
```

如果有疑问, 请记住在包中添加并发的代码总是容易的, 但是移除是非常苦难甚至是不可能的. 最后, 不要忘记在文档中提示一个包或者结构体是否是并发安全的.

有时候, 导出一个 channel 也是可取的或者必要的. 为了在这种情况下缓解上述的问题, 可以通过访问者模式替代直接暴露 channel, 并强制在访问者中申明 channel 是只读或者只写的.

## 使用 net/http/httptest 测试 http 代码 ##

httptest 运行你在不启动一个服务器或者绑定到一个端口的情况下运行你的 http.Handler 代码. 这可以加快测试速度, 并允许它们尽量的并行.

以下是使用两种方法实现的相同测试的示例代码, 它看起来不多, 但是它为您节省了大量的代码和资源:

```
func TestServe(t *testing.T) {
    // The method to use if you want to practice typing
    s := &http.Server{
        Handler: http.HandlerFunc(ServeHTTP),
    }
    // Pick port automatically for parallel tests and to avoid conflicts
    l, err := net.Listen("tcp", ":0")
    if err != nil {
        t.Fatal(err)
    }
    defer l.Close()
    go s.Serve(l)

    res, err := http.Get("http://" + l.Addr().String() + "/?sloths=arecool")
    if err != nil {
        log.Fatal(err)
    }
    greeting, err := ioutil.ReadAll(res.Body)
    res.Body.Close()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(greeting))
}

func TestServeMemory(t *testing.T) {
    // Less verbose and more flexible way
    req := httptest.NewRequest("GET", "http://example.com/?sloths=arecool", nil)
    w := httptest.NewRecorder()

    ServeHTTP(w, req)
    greeting, err := ioutil.ReadAll(w.Body)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(greeting))
}
```

使用 httptest 最大的优势就是可以只运行你想要测试的功能. 不会有你以前认为的是好的, 从服务器或其他可憎的事物中带来的路由, 中间件或者其他的副作用,

你可以从 Mark Berger 的[这篇文章](http://markjberger.com/testing-web-apps-in-golang/)中看到更多的这样的模式.

## 使用一个单独的 _test包 ##

在 Go 语言中大多数的测试都是在相同的包中创建一个 pkg\_test.go 文件并在该文件中编写测试代码. 一个单独的测试包是指在你想要测试的包 foo 的目录中新建一个 foo\_test 文件, 并且这个文件在包 foo\_test 中. 在这种情况下, 你可以从 github.com/example/foo 那里导入你的额依赖. 这个方式提供了以下好处: 这是测试中存在循环依赖关系的推荐解决办法, 可以防止脆弱的测试, 并允许开发人员感受使用自己的软件包的感觉. 如果你的软件包很难被使用, 那么使用这种方法也可能很难进行测试.

这个技巧通过限制对私有变量的访问来防止脆弱的测试. 特别是, 如果你的测试出错了, 而且你使用的是一个单独的测试包, 那么使用这个包的客户端也会出现错误.

最后, 这种方式可以避免循环依赖. 大多数包都会依赖于在其他包, 因此可能会遇到循环依赖的情况. 外部程序包位于包结构中的两个包之上. 以 Go Programming Language (Chp. 11 Sec 2.4) 为例, net/url 实现了 net/http 包导入使用的 URL 解析器, 但是 net/url 想要通过导入 net/http 来进行测试, 于是 net/url_test 包出现了.

如果你正在使用单独的测试包, 那么你可能需要访问在被测试包中没有导出的实体. 大多数人都在测试基于时间的值时(例如 time.Now) 首先碰到这个问题. 在这种情况下, 我们可以使用额外的文件在测试期暴露它们, 因为 _test.go 文件在常规构建时是被排除在外的.

## 一些其他的点 ##

需要注意的是没有银弹, 最好的解决方案始终是对情况进行批判性分析, 并确定适合问题的最佳解决方案.

如果想要了解更多的测试技术, 可以查看以下帖子:

- [Writing Table Driven Tests in Go by Dave Cheney](https://dave.cheney.net/2013/06/09/writing-table-driven-tests-in-go)
- [The Go Programming Language chapter on Testing.](http://www.gopl.io/)

或者这些视频:

- [Hashimoto’s Advanced Testing With Go talk from Gophercon 2017](https://www.youtube.com/watch?v=yszygk1cpEc)
- [Andrew Gerrand's Testing Techniques talk from 2014](https://talks.golang.org/2014/testing.slide#1)

# 资料 #

- 原文 [5 Advanced Testing Techniques in Go](https://segment.com/blog/5-advanced-testing-techniques-in-go/)
