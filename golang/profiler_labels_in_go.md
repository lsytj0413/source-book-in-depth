# Go 中的性能分析器标签 #

Go1.9 引入了[profiler标签](https://github.com/golang/proposal/blob/master/design/17280-profile-labels.md)功能, 可以向 CPU 分析器收集的样本添加任意键值. CPU分析器收集并输出CPU执行时花费最多时间的热点, 典型的CPU分析器主要报告这些位置的函数名称, 源代码行数等. 通过查看这些数据你可以检查代码的哪些部分调用了这些点, 并且可以按照调用者过滤来更细致的了解某些执行路径.

尽管位置数据在分析中是非常有用的, 但是它并不总是足够. 很多的Go程序运行在服务器上, 在服务器上分析性能问题更加的复杂, 很难将某些路径同其他路径隔离开, 或者难以理解它是否只是某个特殊路径的问题.

在 Go1.9 中你可以向执行路径中添加更多的上下文, 你可以使用任何[标签集](http://beta.golang.org/pkg/runtime/pprof/#LabelSet)作为分析数据的一部分, 然后使用这些标签更精确的检查分析器的输出.

一些显而易见的用法如下:

- 你不希望将软件的实现细节泄露到分析数据中, 例如对 Web Server 显示对应的 URL 路径比函数名称更有用
- 调用堆栈的信息不足以明确任务的信息, 例如消息队列中的消费者可以通过标签来识别消息的来源
- 待分析的问题需要上下文信息

## 增加标签 ##

**runtime/pprof**包导出了一些函数以允许用户增加标签. 常见的用法是使用 **Do** 函数来扩展一个Context, 然后在 f 函数执行时记录这些标签:

```
func Do(ctx context.Context, labels LabelSet, f func(context.Context))
```

Do函数只在当前 goroutine 执行的过程中设置标签, 如果你需要在 f 函数中使用 goroutine 并保存标签, 需要使用如下方式:

```
labels := pprof.Labels("worker", "purge")
pprof.Do(ctx, labels, func(ctx context.Context) {
    // Do some work
    go update(ctx)   // propagates labels in ctx.
})
```

上面的标签将被记为 worker:purge.

## 检查分析器输出 ##

本节将演示如何通过分析器标签检查记录的样本, 使用标签过滤器来分析和使用分析器数据.

使用 **net/http/pprof** 来进行数据采样, 代码如下:

```
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof"
	"runtime/pprof"
	"time"
)

func main() {
	ctx := context.Background()
	go func() {
		time.Sleep(time.Second)
		for i := 0; i < 10000000; i++ {
			labels := pprof.Labels("handler", "hello")
			pprof.Do(ctx, labels, func(ctx context.Context) {
				generate(1, 10)
			})
		}
		fmt.Println("handler:hello done")
	}()

	go func() {
		time.Sleep(time.Second)
		for i := 0; i < 10000000; i++ {
			labels := pprof.Labels("handler2", "hello")
			pprof.Do(ctx, labels, func(ctx context.Context) {
				generate(1, 10)
			})
		}
		fmt.Println("handler2:hello done")
	}()

	log.Fatal(http.ListenAndServe("localhost:5555", nil))
}

func generate(duration int, usage int) string {
	s := fmt.Sprintf("duration: %d", duration)
	for i := 0; i < usage; i++ {
		s += s
	}
	return s
}
```

使用以下命令来获取采样数据:

```
go tool pprof http://localhost:5555/debug/pprof/profile
```

通过以下命令来获取标签以及查看对应标签的数据:

```
PS C:\WINDOWS\system32> go tool pprof http://localhost:5555/debug/pprof/profile
Fetching profile over HTTP from http://localhost:5555/debug/pprof/profile
Saved profile in C:\Users\51112\pprof\pprof.samples.cpu.009.pb.gz
Type: cpu
Time: Aug 22, 2018 at 3:34pm (CST)
Duration: 30.16s, Total samples = 1.41mins (281.45%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 50.27s, 59.22% of 84.89s total
Dropped 222 nodes (cum <= 0.42s)
Showing top 10 nodes out of 101
      flat  flat%   sum%        cum   cum%
    14.97s 17.63% 17.63%     25.43s 29.96%  runtime.scanobject
    13.61s 16.03% 33.67%     13.61s 16.03%  runtime.memclrNoHeapPointers
     6.23s  7.34% 41.01%      6.23s  7.34%  runtime.memmove
     4.27s  5.03% 46.04%      4.66s  5.49%  runtime.heapBitsForObject
     2.39s  2.82% 48.85%      3.57s  4.21%  runtime.greyobject
     2.27s  2.67% 51.53%      2.27s  2.67%  runtime.heapBits.bits (inline)
     2.15s  2.53% 54.06%      8.22s  9.68%  runtime.mallocgc
     1.55s  1.83% 55.88%      1.55s  1.83%  runtime.nextFreeFast (inline)
     1.45s  1.71% 57.59%      1.45s  1.71%  runtime.stdcall2
     1.38s  1.63% 59.22%      2.72s  3.20%  runtime.gcmarknewobject
(pprof) tags
 handler: Total 8.7s
          8.7s (  100%): hello

 handler2: Total 8.6s
           8.6s (  100%): hello
(pprof) tagfocus="handler:hello"
(pprof) top
Active filters:
   tagfocus=handler:hello
Showing nodes accounting for 6.49s, 7.65% of 84.89s total
Dropped 41 nodes (cum <= 0.42s)
Showing top 10 nodes out of 39
      flat  flat%   sum%        cum   cum%
     2.97s  3.50%  3.50%      2.97s  3.50%  runtime.memmove
     0.74s  0.87%  4.37%      6.17s  7.27%  runtime.concatstrings
     0.71s  0.84%  5.21%      2.49s  2.93%  runtime.mallocgc
     0.60s  0.71%  5.91%      0.60s  0.71%  runtime.nextFreeFast (inline)
     0.40s  0.47%  6.38%      0.43s  0.51%  runtime.gcWriteBarrier
     0.35s  0.41%  6.80%      0.87s  1.02%  runtime.gcmarknewobject
     0.20s  0.24%  7.03%      2.34s  2.76%  runtime.rawstring
     0.18s  0.21%  7.24%      0.41s  0.48%  runtime.deferreturn
     0.18s  0.21%  7.46%      0.18s  0.21%  runtime.spanOfUnchecked (inline)
     0.16s  0.19%  7.65%      6.33s  7.46%  runtime.concatstring2
(pprof) tagfocus="handler2:hello"
(pprof) top
Active filters:
   tagfocus=handler2:hello
Showing nodes accounting for 6.50s, 7.66% of 84.89s total
Dropped 44 nodes (cum <= 0.42s)
Showing top 10 nodes out of 50
      flat  flat%   sum%        cum   cum%
     3.18s  3.75%  3.75%      3.18s  3.75%  runtime.memmove
     0.81s  0.95%  4.70%      2.68s  3.16%  runtime.mallocgc
     0.60s  0.71%  5.41%      6.38s  7.52%  runtime.concatstrings
     0.50s  0.59%  6.00%      0.50s  0.59%  runtime.nextFreeFast (inline)
     0.46s  0.54%  6.54%      0.89s  1.05%  runtime.gcmarknewobject
     0.27s  0.32%  6.86%      0.32s  0.38%  runtime.gcWriteBarrier
     0.20s  0.24%  7.09%      2.63s  3.10%  runtime.rawstringtmp
     0.20s  0.24%  7.33%      0.20s  0.24%  runtime.spanOfUnchecked (inline)
     0.14s  0.16%  7.49%      7.96s  9.38%  main.generate
     0.14s  0.16%  7.66%      0.14s  0.16%  runtime.acquirem (inline)
(pprof) tagfocus=
(pprof) tagignore="handler:hello"
(pprof) tags
TagHide expression matched no samples
 handler2: Total 8.6s
           8.6s (  100%): hello

(pprof)
```

从以上实例可以看到, 我们可以通过 tagfocus 来设置只显示含有对应标签的数据, 使用 tagignore 来过滤对应标签的数据. 其他的还有 tagshow, taghide 等命令.

对于 HTTP 路径, 可以使用 [pprofutil](https://godoc.org/github.com/rakyll/goutil/pprofutil) 包来自动的增加标签.