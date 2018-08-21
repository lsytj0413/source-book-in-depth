# Go 程序的调试分析和优化 #

本文章来自于 [Brad Fitzpatrick](https://github.com/bradfitz) 的一篇[演讲](https://github.com/bradfitz/talk-yapc-asia-2015/blob/master/talk.md), 主要内容是对一个 Go 程序进行调试分析和优化的实例. 这里记录按照作者的思路进行一次实操的流程, 以便加深理解.

## 实验环境 ##

### 操作系统版本 ###

```
soren@SOREN-MIBOOK:~/workspace/knownsec/vip-server$ uname -a
Linux SOREN-MIBOOK 4.4.0-17134-Microsoft #137-Microsoft Thu Jun 14 18:46:00 PST 2018 x86_64 x86_64 x86_64 GNU/Linux
```

### Go 版本 ###

```
soren@SOREN-MIBOOK:~/workspace/knownsec/vip-server$ go version
go version go1.10.1 linux/amd64

soren@SOREN-MIBOOK:~/workspace/knownsec/vip-server$ go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/home/soren/.cache/go-build"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/soren/workspace/golang/lib:/home/soren/workspace/golang"
GORACE=""
GOROOT="/usr/lib/go-1.10"
GOTMPDIR=""
GOTOOLDIR="/usr/lib/go-1.10/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build872215374=/tmp/go-build -gno-record-gcc-switches"
```

### 待优化代码 ###

**main.go** 文件内容如下:

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"regexp"
)

var visitors int

func handleHi(w http.ResponseWriter, r *http.Request) {
	if match, _ := regexp.MatchString(`^\w*$`, r.FormValue("color")); !match {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}

	visitors++
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write([]byte("<h1 style='color: " + r.FormValue("color") + "'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitors) + "!"))
	return
}

func main() {
	log.Printf("Starting on port 8080")
	http.HandleFunc("/hi", handleHi)
	log.Fatal(http.ListenAndServe("127.0.0.1:8080", nil))
}
```

使用 **go run main.go** 启动该程序, 然后用浏览器访问 **http://localhost:8080/hi** 就可以看到返回的内容了.

添加测试代码 **main_test.go** 内容如下:

```
package main

import (
	"bufio"
	"io/ioutil"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func req(t *testing.T, v string) *http.Request {
	req, err := http.ReadRequest(bufio.NewReader(strings.NewReader(v)))
	if err != nil {
		t.Fatal(err)
	}

	return req
}

func TestHandleHi_Recorder(t *testing.T) {
	rw := httptest.NewRecorder()
	handleHi(rw, req(t, "GET / HTTP/1.0\r\n\r\n"))
	if !strings.Contains(rw.Body.String(), "visitor number") {
		t.Errorf("Unexpected output: %s", rw.Body)
	}
}

func TestHandleHi_TestServer(t *testing.T) {
	ts := httptest.NewServer(http.HandlerFunc(handleHi))
	defer ts.Close()

	res, err := http.Get(ts.URL)
	if err != nil {
		t.Error(err)
		return
	}

	if g, w := res.Header.Get("Content-Type"), "text/html; charset=utf-8"; g != w {
		t.Errorf("Content-Type = %q; want %q", g, w)
	}

	slurp, err := ioutil.ReadAll(res.Body)
	defer res.Body.Close()

	if err != nil {
		t.Error(err)
		return
	}

	t.Logf("Got: %s", slurp)
}
```

执行 **go test -v** 命令, 获得如下输出:

```
=== RUN   TestHandleHi_Recorder
--- PASS: TestHandleHi_Recorder (0.00s)
=== RUN   TestHandleHi_TestServer
--- PASS: TestHandleHi_TestServer (0.00s)
        main_test.go:51: Got: <h1 style='color: '>Welcome!</h1>You are visitor number 2!
PASS
ok      github.com/lsytj0413/ena/example/profile        0.021s
```

至此, 测试代码准备完毕.

## 竞态分析 ##

在 Go 中当你在多个 goroutine 中访问共享的数据时(至少一个 goroutine 执行了写操作), 会存在竞态情况.

我们首先使用 **go test -race** 命令来检查代码中是否有竞态:

```
PASS
ok      github.com/lsytj0413/ena/example/profile        1.028s
```

看起来所有情况都是正常的是吧? 其实不是的, 以上命令是进行运行时分析, 会存在实际存在竞态但没有发现的情况(但是不会存在误报). 我们需要让测试代码并发起来, 向 **main_test.go** 中加入如下代码:

```
func TestHandleHi_TestServer_Parallel(t *testing.T) {
	ts := httptest.NewServer(http.HandlerFunc(handleHi))
	defer ts.Close()

	var wg sync.WaitGroup
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			res, err := http.Get(ts.URL)
			if err != nil {
				t.Error(err)
				return
			}

			if g, w := res.Header.Get("Content-Type"), "text/html; charset=utf-8"; g != w {
				t.Errorf("Content-Type = %q; want %q", g, w)
			}

			slurp, err := ioutil.ReadAll(res.Body)
			defer res.Body.Close()

			if err != nil {
				t.Error(err)
				return
			}

			t.Logf("Got: %s", slurp)
		}()
	}

	wg.Wait()
}
```

再次执行 **go test -race** 命令, 获取的输出如下:

```
==================
WARNING: DATA RACE
Read at 0x000000a66338 by goroutine 28:
  github.com/lsytj0413/ena/example/profile.handleHi()
      /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go:18 +0xaf
  net/http.HandlerFunc.ServeHTTP()
      /usr/lib/go-1.10/src/net/http/server.go:1947 +0x51
  net/http.serverHandler.ServeHTTP()
      /usr/lib/go-1.10/src/net/http/server.go:2694 +0xb9
  net/http.(*conn).serve()
      /usr/lib/go-1.10/src/net/http/server.go:1830 +0x7dc

Previous write at 0x000000a66338 by goroutine 27:
  github.com/lsytj0413/ena/example/profile.handleHi()
      /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go:18 +0xcb
  net/http.HandlerFunc.ServeHTTP()
      /usr/lib/go-1.10/src/net/http/server.go:1947 +0x51
  net/http.serverHandler.ServeHTTP()
      /usr/lib/go-1.10/src/net/http/server.go:2694 +0xb9
  net/http.(*conn).serve()
      /usr/lib/go-1.10/src/net/http/server.go:1830 +0x7dc

Goroutine 28 (running) created at:
  net/http.(*Server).Serve()
      /usr/lib/go-1.10/src/net/http/server.go:2795 +0x364
  net/http/httptest.(*Server).goServe.func1()
      /usr/lib/go-1.10/src/net/http/httptest/server.go:280 +0xa2

Goroutine 27 (running) created at:
  net/http.(*Server).Serve()
      /usr/lib/go-1.10/src/net/http/server.go:2795 +0x364
  net/http/httptest.(*Server).goServe.func1()
      /usr/lib/go-1.10/src/net/http/httptest/server.go:280 +0xa2
==================
--- FAIL: TestHandleHi_TestServer_Parallel (0.03s)
        main_test.go:82: Got: <h1 style='color: '>Welcome!</h1>You are visitor number 3!
        main_test.go:82: Got: <h1 style='color: '>Welcome!</h1>You are visitor number 4!
        testing.go:730: race detected during execution of test
FAIL
exit status 1
FAIL    github.com/lsytj0413/ena/example/profile        0.066s
```

从以上输出可以看到, 在 **main.go** 的第18行中存在竞态条件, 代码是:

```
visitor++
```

其中 visitor 被多个 goroutine 读写但未采用同步机制(每一个 request 的处理都在一个新的 goroutine 中). 有多种方式可以修复这个问题, 例如使用 channel, 使用 mutex, 或者使用 atomic 等. 此处我们修改如下然后再次运行, 即可看到竞态已经不存在了:

```
var visitors int64

func handleHi(w http.ResponseWriter, r *http.Request) {
	if match, _ := regexp.MatchString(`^\w*$`, r.FormValue("color")); !match {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}
	visitNum := atomic.AddInt64(&visitors, 1)

	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write([]byte("<h1 style='color: " + r.FormValue("color") + "'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
	return
}
```

## CPU 分析 ##

要做 CPU 分析我们需要 Benchmark 数据, 添加如下的 Benchmark 测试方法到 **main_test.go**:

```
func BenchmarkHi(b *testing.B) {
	b.ReportAllocs()
	r := req(b, "GET / HTTP/1.0\r\n\r\n")

	for i := 0; i < b.N; i++ {
		rw := httptest.NewRecorder()
		handleHi(rw, r)
	}
}
```

其中需要将 req 函数的第一个参数由 *testing.T 类型修改为 testing.TB. 使用 **go test -v -run=^$ -bench=.** 命令执行 Benchmark 测试, 输出如下:

```
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4             200000              7728 ns/op            4369 B/op         63 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        1.653s
```

### 第一次 CPU 分析 ###

执行以下命令:

```
go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -cpuprofile=prof.cpu
```

输出如下:

```
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4             500000              7095 ns/op            4369 B/op         63 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        3.650s
```

并且在当前目录生成了 **prof.cpu** 以及 **profile.test** 两个文件. 然后输入命令 **go tool pprof profile.test prof.cpu**:

```
File: profile.test.exe
Type: cpu
Time: Aug 21, 2018 at 2:53pm (CST)
Duration: 2.61s, Total samples = 2.81s (107.57%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1450ms, 51.60% of 2810ms total
Dropped 29 nodes (cum <= 14.05ms)
Showing top 10 nodes out of 129
      flat  flat%   sum%        cum   cum%
     370ms 13.17% 13.17%      580ms 20.64%  runtime.mallocgc
     260ms  9.25% 22.42%      300ms 10.68%  runtime.scanobject
     200ms  7.12% 29.54%      490ms 17.44%  runtime.growslice
     140ms  4.98% 34.52%      140ms  4.98%  runtime.heapBitsSetType
     120ms  4.27% 38.79%      120ms  4.27%  runtime.memclrNoHeapPointers
      80ms  2.85% 41.64%       80ms  2.85%  runtime.duffcopy
      80ms  2.85% 44.48%       80ms  2.85%  runtime.heapBitsForObject
      70ms  2.49% 46.98%      200ms  7.12%  runtime.makeslice
      70ms  2.49% 49.47%      300ms 10.68%  runtime.newobject
      60ms  2.14% 51.60%      120ms  4.27%  runtime.mapassign_faststr
(pprof) top --cum
Showing nodes accounting for 0.04s, 1.42% of 2.81s total
Dropped 29 nodes (cum <= 0.01s)
Showing top 10 nodes out of 129
      flat  flat%   sum%        cum   cum%
         0     0%     0%      2.02s 71.89%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%     0%      2.02s 71.89%  testing.(*B).launch
         0     0%     0%      2.02s 71.89%  testing.(*B).runN
         0     0%     0%      1.95s 69.40%  github.com/lsytj0413/ena/example/profile.handleHi
     0.01s  0.36%  0.36%      1.44s 51.25%  regexp.MatchString
         0     0%  0.36%      1.32s 46.98%  regexp.Compile
         0     0%  0.36%      1.32s 46.98%  regexp.compile
     0.02s  0.71%  1.07%      0.71s 25.27%  regexp.compileOnePass
     0.01s  0.36%  1.42%      0.69s 24.56%  runtime.systemstack
         0     0%  1.42%      0.68s 24.20%  runtime.mstart
(pprof) list handleHi
Total: 2.81s
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in D:\workspace\golang\src\github.com\lsytj0413\ena\example\profile\main.go
         0      1.95s (flat, cum) 69.40% of Total
         .          .      9:)
         .          .     10:
         .          .     11:var visitors int64
         .          .     12:
         .          .     13:func handleHi(w http.ResponseWriter, r *http.Request) {
         .      1.45s     14:   if match, _ := regexp.MatchString(`^\w*$`, r.FormValue("color")); !match {
         .          .     15:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     16:           return
         .          .     17:   }
         .          .     18:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     19:
         .      140ms     20:   w.Header().Set("Content-Type", "text/html; charset=utf-8")
         .      360ms     21:   w.Write([]byte("<h1 style='color: " + r.FormValue("color") + "'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
         .          .     22:   return
         .          .     23:}
         .          .     24:
         .          .     25:func main() {
         .          .     26:   log.Printf("Starting on port 8080")
(pprof) web
```

使用 web 命令会获取到对应的 svg 图片, 内容如下:

![pprof001.png](/images/go/pprof001.png)

从上面的输出可以看到, handleHi 函数一共运行了 1.95s, 其中 1.44s 都是消耗在 regexp.MatchString 函数中, 而 regexp.Compile 函数又占据了相当长的时间.

### 第一次优化 ###

从上面的分析中可知正则表达式多次的编译造成了性能的下降, 现在修改代码使其只编译一次:

```
var colorRx = regexp.MustCompile(`^\w*$`)

func handleHi(w http.ResponseWriter, r *http.Request) {
	if !colorRx.MatchString(r.FormValue("color")) {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}
```

再次执行 benchmark:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -v -run=^$ -bench=^BenchmarkHi$
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4            1000000              1513 ns/op            1152 B/op         12 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        2.378s
```

可以看到之前 3.6s 只能执行 50W 次测试, 现在 2.4s 就能执行 100W 次测试, 性能有了极大提高. 再看 cpuprofile 的结果:

```
PS D:\workspace\golang\src\github.com\lsytj0413\ena\example\profile> go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -cpuprofile=prof
goos: windows
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4            2000000              1748 ns/op            1152 B/op         12 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        5.556s
PS D:\workspace\golang\src\github.com\lsytj0413\ena\example\profile> go tool pprof profile.test.exe prof
File: profile.test.exe
Type: cpu
Time: Aug 21, 2018 at 3:20pm (CST)
Duration: 5.43s, Total samples = 6.19s (114.04%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 3460ms, 55.90% of 6190ms total
Dropped 77 nodes (cum <= 30.95ms)
Showing top 10 nodes out of 107
      flat  flat%   sum%        cum   cum%
    1010ms 16.32% 16.32%     1670ms 26.98%  runtime.mallocgc
     550ms  8.89% 25.20%      550ms  8.89%  runtime.heapBitsSetType
     450ms  7.27% 32.47%      450ms  7.27%  runtime.memclrNoHeapPointers
     410ms  6.62% 39.10%      510ms  8.24%  runtime.scanobject
     280ms  4.52% 43.62%      680ms 10.99%  runtime.mapassign_faststr
     180ms  2.91% 46.53%      180ms  2.91%  net/textproto.CanonicalMIMEHeaderKey
     170ms  2.75% 49.27%      180ms  2.91%  runtime.mapiternext
     170ms  2.75% 52.02%      170ms  2.75%  runtime.memmove
     130ms  2.10% 54.12%     1300ms 21.00%  runtime.newobject
     110ms  1.78% 55.90%      120ms  1.94%  runtime.scanblock
(pprof) top -cum
Showing nodes accounting for 1.32s, 21.32% of 6.19s total
Dropped 77 nodes (cum <= 0.03s)
Showing top 10 nodes out of 107
      flat  flat%   sum%        cum   cum%
     0.03s  0.48%  0.48%      4.22s 68.17%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%  0.48%      4.22s 68.17%  testing.(*B).launch
         0     0%  0.48%      4.22s 68.17%  testing.(*B).runN
     0.05s  0.81%  1.29%      3.56s 57.51%  github.com/lsytj0413/ena/example/profile.handleHi
     0.04s  0.65%  1.94%      1.75s 28.27%  runtime.systemstack
         0     0%  1.94%      1.71s 27.63%  runtime.mstart
     1.01s 16.32% 18.26%      1.67s 26.98%  runtime.mallocgc
     0.04s  0.65% 18.90%      1.39s 22.46%  net/http/httptest.(*ResponseRecorder).Write
     0.13s  2.10% 21.00%      1.30s 21.00%  runtime.newobject
     0.02s  0.32% 21.32%      1.21s 19.55%  net/http/httptest.(*ResponseRecorder).writeHeader
```

handleHi 函数的耗时有一定的下降.

## 内存分析 ##

执行以下命令获取内存分析统计:

```
 go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.out
```

获取到的输出如下:

```
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4            2000000              1562 ns/op            1152 B/op         12 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        4.940s
```

执行以下命令查看内存分析数据:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool pprof --alloc_space profile.test mem.out
Local symbolization failed for profile.test: open /tmp/go-build168902618/b001/profile.test: no such file or directory
Some binary filenames not available. Symbolization may be incomplete.
Try setting PPROF_BINARY_PATH to the search path for local binaries.
File: profile.test
Type: alloc_space
Time: Aug 21, 2018 at 3:25pm (DST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 3296.76MB, 100% of 3297.26MB total
Dropped 1 node (cum <= 16.49MB)
Showing top 10 nodes out of 12
      flat  flat%   sum%        cum   cum%
 1183.84MB 35.90% 35.90%  1183.84MB 35.90%  net/http/httptest.cloneHeader
 1054.84MB 31.99% 67.90%  1054.84MB 31.99%  net/textproto.MIMEHeader.Set
  628.55MB 19.06% 86.96%   628.55MB 19.06%  net/http/httptest.NewRecorder (inline)
  396.02MB 12.01% 98.97%  2668.21MB 80.92%  github.com/lsytj0413/ena/example/profile.handleHi
   33.50MB  1.02%   100%    33.50MB  1.02%  fmt.Sprint
         0     0%   100%  3297.26MB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%   100%  1054.84MB 31.99%  net/http.Header.Set
         0     0%   100%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).Write
         0     0%   100%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).WriteHeader
         0     0%   100%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).writeHeader
(pprof) top --cum
Showing nodes accounting for 2634.71MB, 79.91% of 3297.26MB total
Dropped 1 node (cum <= 16.49MB)
Showing top 10 nodes out of 12
      flat  flat%   sum%        cum   cum%
         0     0%     0%  3297.26MB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%     0%  3297.26MB   100%  testing.(*B).launch
         0     0%     0%  3297.26MB   100%  testing.(*B).runN
  396.02MB 12.01% 12.01%  2668.21MB 80.92%  github.com/lsytj0413/ena/example/profile.handleHi
         0     0% 12.01%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).Write
         0     0% 12.01%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).WriteHeader
         0     0% 12.01%  1183.84MB 35.90%  net/http/httptest.(*ResponseRecorder).writeHeader
 1183.84MB 35.90% 47.91%  1183.84MB 35.90%  net/http/httptest.cloneHeader
         0     0% 47.91%  1054.84MB 31.99%  net/http.Header.Set
 1054.84MB 31.99% 79.91%  1054.84MB 31.99%  net/textproto.MIMEHeader.Set
(pprof) list handleHi
Total: 3.22GB
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go
  396.02MB     2.61GB (flat, cum) 80.92% of Total
         .          .     16:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     17:           return
         .          .     18:   }
         .          .     19:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     20:
         .     1.03GB     21:   w.Header().Set("Content-Type", "text/html; charset=utf-8")
  396.02MB     1.58GB     22:   w.Write([]byte("<h1 style='color: " + r.FormValue("color") + "'>Welcome!</h1>You are visitor number " + fmt.Sprint(visitNum) + "!"))
         .          .     23:   return
         .          .     24:}
         .          .     25:
         .          .     26:func main() {
         .          .     27:   log.Printf("Starting on port 8080")
(pprof)
```

可以看到 handleHi 函数的 21及22 行使用了较多的内存. 优化方法如下:

1. 删除 21 行
2. 将 22 行替换为 fmt.Fprintf

优化后的代码如下:

```
func handleHi(w http.ResponseWriter, r *http.Request) {
	if !colorRx.MatchString(r.FormValue("color")) {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}
	visitNum := atomic.AddInt64(&visitors, 1)

	fmt.Fprintf(w, "<html><h1 style='color: \"%s\"'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum)

	return
}
```

重新执行一个分析, 结果如下:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.out
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4            2000000              1736 ns/op            1096 B/op         10 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        5.245s
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool pprof --alloc_space profile.test mem.out
Local symbolization failed for profile.test: open /tmp/go-build563508654/b001/profile.test: no such file or directory
Some binary filenames not available. Symbolization may be incomplete.
Try setting PPROF_BINARY_PATH to the search path for local binaries.
File: profile.test
Type: alloc_space
Time: Aug 21, 2018 at 3:37pm (DST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 3.08GB, 100% of 3.08GB total
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
    1.15GB 37.49% 37.49%     1.15GB 37.49%  net/http/httptest.cloneHeader
    1.01GB 32.80% 70.29%     1.01GB 32.80%  net/textproto.MIMEHeader.Set
    0.65GB 21.14% 91.43%     0.65GB 21.14%  net/http/httptest.NewRecorder (inline)
    0.24GB  7.84% 99.27%     0.24GB  7.84%  bytes.makeSlice
    0.02GB  0.73%   100%     2.43GB 78.86%  github.com/lsytj0413/ena/example/profile.handleHi
         0     0%   100%     0.24GB  7.84%  bytes.(*Buffer).Write
         0     0%   100%     0.24GB  7.84%  bytes.(*Buffer).grow
         0     0%   100%     2.40GB 78.13%  fmt.Fprintf
         0     0%   100%     3.08GB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%   100%     1.01GB 32.80%  net/http.Header.Set
(pprof) top --cum
Showing nodes accounting for 1.18GB, 38.22% of 3.08GB total
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
         0     0%     0%     3.08GB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%     0%     3.08GB   100%  testing.(*B).launch
         0     0%     0%     3.08GB   100%  testing.(*B).runN
    0.02GB  0.73%  0.73%     2.43GB 78.86%  github.com/lsytj0413/ena/example/profile.handleHi
         0     0%  0.73%     2.40GB 78.13%  fmt.Fprintf
         0     0%  0.73%     2.40GB 78.13%  net/http/httptest.(*ResponseRecorder).Write
         0     0%  0.73%     2.16GB 70.29%  net/http/httptest.(*ResponseRecorder).writeHeader
         0     0%  0.73%     1.15GB 37.49%  net/http/httptest.(*ResponseRecorder).WriteHeader
    1.15GB 37.49% 38.22%     1.15GB 37.49%  net/http/httptest.cloneHeader
         0     0% 38.22%     1.01GB 32.80%  net/http.Header.Set
(pprof) list handleHi
Total: 3.08GB
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go
      23MB     2.43GB (flat, cum) 78.86% of Total
         .          .     16:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     17:           return
         .          .     18:   }
         .          .     19:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     20:
      23MB     2.43GB     21:   fmt.Fprintf(w, "<html><h1 style='color: \"%s\"'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum)
         .          .     22:
         .          .     23:   return
         .          .     24:}
         .          .     25:
         .          .     26:func main() {
(pprof)
```

可以看出内存占用有一定程度的减少.

## Benchcmp ##

Golang 有一个 benchcmp 工具, 可以给出两次 benchmark 的结果对比.

在解决竞态后的代码中运行如下命令:

```
go test -bench=. -memprofile=prof.mem | tee mem.0
```

在最新的代码中运行如下命令:

```
go test -bench=. -memprofile=prof.mem | tee mem.2
```

使用 benchcmp 对比结果:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ benchcmp mem.0 mem.2
benchmark         old ns/op     new ns/op     delta
BenchmarkHi-4     6867          1642          -76.09%

benchmark         old allocs     new allocs     delta
BenchmarkHi-4     63             10             -84.13%

benchmark         old bytes     new bytes     delta
BenchmarkHi-4     4369          1096          -74.91%
```

可以看到优化后的内存分配和运行时间都大幅度减少.

## 内存还能优化吗 ##

首先, 我们在每次测试运行之后进行一次内存清理:

```
func BenchmarkHi(b *testing.B) {
	b.ReportAllocs()
	r := req(b, "GET / HTTP/1.0\r\n\r\n")

	for i := 0; i < b.N; i++ {
		rw := httptest.NewRecorder()
		handleHi(rw, r)
		reset(rw)
	}
}

func reset(rw *httptest.ResponseRecorder) {
	m := rw.HeaderMap
	for k := range m {
		delete(m, k)
	}

	body := rw.Body
	body.Reset()
	*rw = httptest.ResponseRecorder{
		Body:      body,
		HeaderMap: m,
	}
}
```

再次运行测试并查看结果:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -v -run=^$ -bench=^BenchmarkHi$ -benchtime=2s -memprofile=mem.out
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHi-4            1000000              2856 ns/op            1096 B/op         10 allocs/op
PASS
ok      github.com/lsytj0413/ena/example/profile        2.917s
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool pprof --alloc_space profile.test mem.out
Local symbolization failed for profile.test: open /tmp/go-build739441264/b001/profile.test: no such file or directory
Some binary filenames not available. Symbolization may be incomplete.
Try setting PPROF_BINARY_PATH to the search path for local binaries.
File: profile.test
Type: alloc_space
Time: Aug 21, 2018 at 3:56pm (DST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1044.25MB, 100% of 1044.25MB total
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
  403.62MB 38.65% 38.65%   403.62MB 38.65%  net/http/httptest.cloneHeader
  344.61MB 33.00% 71.65%   344.61MB 33.00%  net/textproto.MIMEHeader.Set
  217.02MB 20.78% 92.43%   217.02MB 20.78%  net/http/httptest.NewRecorder (inline)
   72.51MB  6.94% 99.38%    72.51MB  6.94%  bytes.makeSlice
    6.50MB  0.62%   100%   827.23MB 79.22%  github.com/lsytj0413/ena/example/profile.handleHi
         0     0%   100%    72.51MB  6.94%  bytes.(*Buffer).Write
         0     0%   100%    72.51MB  6.94%  bytes.(*Buffer).grow
         0     0%   100%   820.73MB 78.60%  fmt.Fprintf
         0     0%   100%  1044.25MB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%   100%   344.61MB 33.00%  net/http.Header.Set
(pprof) top -cum
Showing nodes accounting for 410.12MB, 39.27% of 1044.25MB total
Showing top 10 nodes out of 15
      flat  flat%   sum%        cum   cum%
         0     0%     0%  1044.25MB   100%  github.com/lsytj0413/ena/example/profile.BenchmarkHi
         0     0%     0%  1044.25MB   100%  testing.(*B).launch
         0     0%     0%  1044.25MB   100%  testing.(*B).runN
    6.50MB  0.62%  0.62%   827.23MB 79.22%  github.com/lsytj0413/ena/example/profile.handleHi
         0     0%  0.62%   820.73MB 78.60%  fmt.Fprintf
         0     0%  0.62%   820.73MB 78.60%  net/http/httptest.(*ResponseRecorder).Write
         0     0%  0.62%   748.23MB 71.65%  net/http/httptest.(*ResponseRecorder).writeHeader
         0     0%  0.62%   403.62MB 38.65%  net/http/httptest.(*ResponseRecorder).WriteHeader
  403.62MB 38.65% 39.27%   403.62MB 38.65%  net/http/httptest.cloneHeader
         0     0% 39.27%   344.61MB 33.00%  net/http.Header.Set
(pprof) list handleHi
Total: 1.02GB
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go
    6.50MB   827.23MB (flat, cum) 79.22% of Total
         .          .     16:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     17:           return
         .          .     18:   }
         .          .     19:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     20:
    6.50MB   827.23MB     21:   fmt.Fprintf(w, "<html><h1 style='color: \"%s\"'>Welcome!</h1>You are visitor number %d!", r.FormValue("color"), visitNum)
         .          .     22:
         .          .     23:   return
         .          .     24:}
         .          .     25:
         .          .     26:func main() {
(pprof) disasm handleHi
Total: 1.02GB
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi
    6.50MB   827.23MB (flat, cum) 79.22% of Total
         .          .     666430: MOVQ FS:0xfffffff8, CX                         ;main.go:14
         .          .     666439: LEAQ -0x18(SP), AX
         .          .     66643e: CMPQ 0x10(CX), AX
         .          .     666442: JBE 0x666654
         .          .     666448: SUBQ $0x98, SP
         .          .     66644f: MOVQ BP, 0x90(SP)
         .          .     666457: LEAQ 0x90(SP), BP
         .          .     66645f: MOVQ 0xb0(SP), AX
         .          .     666467: MOVQ AX, 0(SP)                                 ;main.go:15
         .          .     66646b: LEAQ 0xa5b18(IP), CX
         .          .     666472: MOVQ CX, 0x8(SP)
         .          .     666477: MOVQ $0x5, 0x10(SP)
         .          .     666480: CALL net/http.(*Request).FormValue(SB)
         .          .     666485: MOVQ github.com/lsytj0413/ena/example/profile.colorRx(SB), AX
         .          .     66648c: MOVQ 0x18(SP), CX
         .          .     666491: MOVQ 0x20(SP), DX
         .          .     666496: MOVQ AX, 0(SP)
         .          .     66649a: MOVQ CX, 0x8(SP)
         .          .     66649f: MOVQ DX, 0x10(SP)
         .          .     6664a4: CALL regexp.(*Regexp).MatchString(SB)
         .          .     6664a9: MOVZX 0x18(SP), AX
         .          .     6664ae: TESTL AL, AL
         .          .     6664b0: JE 0x666608
         .          .     6664b6: MOVL $0x1, AX                           ;main.go:19
         .          .     6664bb: LEAQ github.com/lsytj0413/ena/example/profile.visitors(SB), CX
         .          .     6664c2: LOCK XADDQ AX, 0(CX)
         .          .     6664c7: MOVQ AX, 0x58(SP)
         .          .     6664cc: MOVQ 0xb0(SP), CX
         .          .     6664d4: MOVQ CX, 0(SP)                                 ;main.go:21
         .          .     6664d8: LEAQ 0xa5aab(IP), CX
         .          .     6664df: MOVQ CX, 0x8(SP)
         .          .     6664e4: MOVQ $0x5, 0x10(SP)
         .          .     6664ed: CALL net/http.(*Request).FormValue(SB)
         .          .     6664f2: MOVQ 0x18(SP), AX
         .          .     6664f7: MOVQ 0x20(SP), CX
         .          .     6664fc: MOVQ AX, 0x60(SP)
         .          .     666501: MOVQ CX, 0x68(SP)
         .          .     666506: MOVQ 0x58(SP), AX
         .          .     66650b: INCQ AX                                       ;main.go:19
         .          .     66650e: MOVQ AX, 0x50(SP)                           ;main.go:21
         .          .     666513: XORPS X0, X0
         .          .     666516: MOVUPS X0, 0x70(SP)
         .          .     66651b: MOVUPS X0, 0x80(SP)
         .          .     666523: LEAQ 0x33436(IP), AX
         .          .     66652a: MOVQ AX, 0(SP)
         .          .     66652e: LEAQ 0x60(SP), AX
         .          .     666533: MOVQ AX, 0x8(SP)
         .          .     666538: CALL runtime.convT2Estring(SB)
         .          .     66653d: MOVQ 0x10(SP), AX
         .          .     666542: MOVQ 0x18(SP), CX
         .          .     666547: MOVQ AX, 0x70(SP)
         .          .     66654c: MOVQ CX, 0x78(SP)
         .          .     666551: LEAQ 0x329c8(IP), AX
         .          .     666558: MOVQ AX, 0(SP)
         .          .     66655c: LEAQ 0x50(SP), AX
         .          .     666561: MOVQ AX, 0x8(SP)
    6.50MB     6.50MB     666566: CALL runtime.convT2E64(SB)                 ;github.com/lsytj0413/ena/example/profile.handleHi main.go:21
         .          .     66656b: MOVQ 0x10(SP), AX                           ;main.go:21
         .          .     666570: MOVQ 0x18(SP), CX
         .          .     666575: MOVQ AX, 0x80(SP)
         .          .     66657d: MOVQ CX, 0x88(SP)
         .          .     666585: LEAQ 0x4f6b4(IP), AX
         .          .     66658c: MOVQ AX, 0(SP)
         .          .     666590: MOVQ 0xa0(SP), AX
         .          .     666598: MOVQ AX, 0x8(SP)
         .          .     66659d: MOVQ 0xa8(SP), AX
         .          .     6665a5: MOVQ AX, 0x10(SP)
         .          .     6665aa: CALL runtime.convI2I(SB)
         .          .     6665af: MOVQ 0x18(SP), AX
         .          .     6665b4: MOVQ 0x20(SP), CX
         .          .     6665b9: MOVQ AX, 0(SP)
         .          .     6665bd: MOVQ CX, 0x8(SP)
         .          .     6665c2: LEAQ 0xb5d5e(IP), AX
         .          .     6665c9: MOVQ AX, 0x10(SP)
         .          .     6665ce: MOVQ $0x45, 0x18(SP)
         .          .     6665d7: LEAQ 0x70(SP), AX
         .          .     6665dc: MOVQ AX, 0x20(SP)
         .          .     6665e1: MOVQ $0x2, 0x28(SP)
         .          .     6665ea: MOVQ $0x2, 0x30(SP)
         .   820.73MB     6665f3: CALL fmt.Fprintf(SB)                     ;github.com/lsytj0413/ena/example/profile.handleHi main.go:21
         .          .     6665f8: MOVQ 0x90(SP), BP                           ;main.go:23
         .          .     666600: ADDQ $0x98, SP
         .          .     666607: RET
         .          .     666608: MOVQ 0xa0(SP), AX
         .          .     666610: MOVQ AX, 0(SP)                                 ;main.go:16
         .          .     666614: MOVQ 0xa8(SP), AX
         .          .     66661c: MOVQ AX, 0x8(SP)
         .          .     666621: LEAQ 0xaaecd(IP), AX
         .          .     666628: MOVQ AX, 0x10(SP)
         .          .     66662d: MOVQ $0x17, 0x18(SP)
         .          .     666636: MOVQ $0x190, 0x20(SP)
         .          .     66663f: CALL net/http.Error(SB)
         .          .     666644: MOVQ 0x90(SP), BP                           ;main.go:17
         .          .     66664c: ADDQ $0x98, SP
         .          .     666653: RET
         .          .     666654: CALL runtime.morestack_noctxt(SB)           ;main.go:14
         .          .     666659: ?
         .          .     66665a: SARL CL, CH
         .          .     66665c: ?
(pprof)
```

内存有一定程度的降低, 并且可以看到内存基本来自于 fmt.Fprintf 函数.

1. A Go interface is 2 words of memory: (type, pointer).
2. A Go string is 2 words of memory: (base pointer, length)
3. A Go slice is 3 words of memory: (base pointer, length, capacity)

fmt.Fprintf 函数的描述如下:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go doc fmt.Fprintf
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
    Fprintf formats according to a format specifier and writes to w. It returns
    the number of bytes written and any write error encountered.
```

在每次调用 fmt.Fprintf 函数时会为参数分配内存, 这就是此处内存分配较多的原因.

### 消除所有内存分配 ###

修改代码如下:

```
var bufPool = sync.Pool{
	New: func() interface{} {
		return new(bytes.Buffer)
	},
}

func handleHi(w http.ResponseWriter, r *http.Request) {
	if !colorRx.MatchString(r.FormValue("color")) {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}
	visitNum := atomic.AddInt64(&visitors, 1)

	buf := bufPool.Get().(*bytes.Buffer)
	defer bufPool.Put(buf)

	buf.Reset()
	buf.WriteString("<h1 style='color: ")
	buf.WriteString(r.FormValue("color"))
	buf.WriteString(">Welcome!</h1>You are visitor number ")
	b := strconv.AppendInt(buf.Bytes(), int64(visitNum), 10)
	b = append(b, '!')
	w.Write(b)

	return
}
```

注: 此处的测试没有得到和文章一样的结果, 可能是不同的 Go 版本的原因.

## 竞争优化 ##

首先写一个并行版本的 banchmark 测试:

```
func BenchmarkHiParallel(b *testing.B) {
	r := req(b, "GET / HTTP/1.0\r\n\r\n")
	b.RunParallel(func(pb *testing.PB) {
		rw := httptest.NewRecorder()
		for pb.Next() {
			handleHi(rw, r)
			reset(rw)
		}
	})
}
```

运行测试并进行分析:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -bench=Parallel -blockprofile=prof.block
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHiParallel-4            2000000               851 ns/op
PASS
ok      github.com/lsytj0413/ena/example/profile        2.621s
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool pprof profile.test  prof.block
Fetched 1 source profiles out of 2
Local symbolization failed for profile.test: open /tmp/go-build706902925/b001/profile.test: no such file or directory
Local symbolization failed for profile.test: open /tmp/go-build706902925/b001/profile.test: no such file or directory
Some binary filenames not available. Symbolization may be incomplete.
Try setting PPROF_BINARY_PATH to the search path for local binaries.
File: profile.test
Type: delay
Time: Aug 21, 2018 at 4:16pm (DST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top -cum 10
Showing nodes accounting for 2.49s, 48.78% of 5.11s total
Dropped 67 nodes (cum <= 0.03s)
Showing top 10 nodes out of 23
      flat  flat%   sum%        cum   cum%
         0     0%     0%      4.97s 97.31%  testing.(*B).runN
     2.49s 48.78% 48.78%      2.49s 48.78%  runtime.chanrecv1
         0     0% 48.78%      2.49s 48.77%  main.main
         0     0% 48.78%      2.49s 48.77%  runtime.main
         0     0% 48.78%      2.49s 48.77%  testing.(*M).Run
         0     0% 48.78%      2.49s 48.69%  testing.(*B).Run
         0     0% 48.78%      2.49s 48.69%  testing.runBenchmarks
         0     0% 48.78%      2.49s 48.69%  testing.runBenchmarks.func1
         0     0% 48.78%      2.49s 48.68%  testing.(*B).doBench
         0     0% 48.78%      2.49s 48.68%  testing.(*B).run
(pprof) list BenchmarkHiParallel
Total: 5.11s
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.BenchmarkHiParallel in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main_test.go
         0      2.48s (flat, cum) 48.62% of Total
         .          .    111:   }
         .          .    112:}
         .          .    113:
         .          .    114:func BenchmarkHiParallel(b *testing.B) {
         .          .    115:   r := req(b, "GET / HTTP/1.0\r\n\r\n")
         .      2.48s    116:   b.RunParallel(func(pb *testing.PB) {
         .          .    117:           rw := httptest.NewRecorder()
         .          .    118:           for pb.Next() {
         .          .    119:                   handleHi(rw, r)
         .          .    120:                   reset(rw)
         .          .    121:           }
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.BenchmarkHiParallel.func1 in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main_test.go
         0   125.99ms (flat, cum)  2.47% of Total
         .          .    114:func BenchmarkHiParallel(b *testing.B) {
         .          .    115:   r := req(b, "GET / HTTP/1.0\r\n\r\n")
         .          .    116:   b.RunParallel(func(pb *testing.PB) {
         .          .    117:           rw := httptest.NewRecorder()
         .          .    118:           for pb.Next() {
         .   125.99ms    119:                   handleHi(rw, r)
         .          .    120:                   reset(rw)
         .          .    121:           }
         .          .    122:   })
         .          .    123:}
(pprof) list handleHi
Total: 5.11s
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go
         0   125.99ms (flat, cum)  2.47% of Total
         .          .     17:           return new(bytes.Buffer)
         .          .     18:   },
         .          .     19:}
         .          .     20:
         .          .     21:func handleHi(w http.ResponseWriter, r *http.Request) {
         .   113.38ms     22:   if !colorRx.MatchString(r.FormValue("color")) {
         .          .     23:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     24:           return
         .          .     25:   }
         .          .     26:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     27:
         .    12.34ms     28:   buf := bufPool.Get().(*bytes.Buffer)
         .          .     29:   defer bufPool.Put(buf)
         .          .     30:
         .          .     31:   buf.Reset()
         .          .     32:   buf.WriteString("<h1 style='color: ")
         .          .     33:   buf.WriteString(r.FormValue("color"))
         .          .     34:   buf.WriteString(">Welcome!</h1>You are visitor number ")
         .          .     35:   b := strconv.AppendInt(buf.Bytes(), int64(visitNum), 10)
         .          .     36:   b = append(b, '!')
         .          .     37:   w.Write(b)
         .          .     38:
         .   273.08us     39:   return
         .          .     40:}
         .          .     41:
         .          .     42:func main() {
         .          .     43:   log.Printf("Starting on port 8080")
         .          .     44:   http.HandleFunc("/hi", handleHi)
(pprof)
```

可以看到 colorRx.MatchString 函数处的耗时较多, 修改代码为如下内容进行优化:

```
var colorRxPool = sync.Pool{
	New: func() interface{} {
		return regexp.MustCompile(`^\w*$`)
	},
}

func handleHi(w http.ResponseWriter, r *http.Request) {
	if !colorRxPool.Get().(*regexp.Regexp).MatchString(r.FormValue("color")) {
		http.Error(w, "Optional color is valid", http.StatusBadRequest)
		return
	}
```

再次运行测试并分析:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -bench=Parallel -blockprofile=prof.block
goos: linux
goarch: amd64
pkg: github.com/lsytj0413/ena/example/profile
BenchmarkHiParallel-4             500000              3480 ns/op
PASS
ok      github.com/lsytj0413/ena/example/profile        1.915s
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool pprof profile.test  prof.block
Local symbolization failed for profile.test: open /tmp/go-build241274106/b001/profile.test: no such file or directory
Some binary filenames not available. Symbolization may be incomplete.
Try setting PPROF_BINARY_PATH to the search path for local binaries.
File: profile.test
Type: delay
Time: Aug 21, 2018 at 4:22pm (DST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top -cum 10
Showing nodes accounting for 1.79s, 49.72% of 3.59s total
Dropped 67 nodes (cum <= 0.02s)
Showing top 10 nodes out of 20
      flat  flat%   sum%        cum   cum%
         0     0%     0%      3.56s 99.11%  testing.(*B).runN
     1.79s 49.72% 49.72%      1.79s 49.72%  runtime.chanrecv1
         0     0% 49.72%      1.79s 49.71%  main.main
         0     0% 49.72%      1.79s 49.71%  runtime.main
         0     0% 49.72%      1.79s 49.71%  testing.(*M).Run
         0     0% 49.72%      1.78s 49.60%  testing.(*B).Run
         0     0% 49.72%      1.78s 49.60%  testing.runBenchmarks
         0     0% 49.72%      1.78s 49.60%  testing.runBenchmarks.func1
         0     0% 49.72%      1.78s 49.58%  testing.(*B).doBench
         0     0% 49.72%      1.78s 49.58%  testing.(*B).run
(pprof) list BenchmarkHiParallel
Total: 3.59s
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.BenchmarkHiParallel in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main_test.go
         0      1.78s (flat, cum) 49.51% of Total
         .          .    111:   }
         .          .    112:}
         .          .    113:
         .          .    114:func BenchmarkHiParallel(b *testing.B) {
         .          .    115:   r := req(b, "GET / HTTP/1.0\r\n\r\n")
         .      1.78s    116:   b.RunParallel(func(pb *testing.PB) {
         .          .    117:           rw := httptest.NewRecorder()
         .          .    118:           for pb.Next() {
         .          .    119:                   handleHi(rw, r)
         .          .    120:                   reset(rw)
         .          .    121:           }
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.BenchmarkHiParallel.func1 in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main_test.go
         0    21.34ms (flat, cum)  0.59% of Total
         .          .    114:func BenchmarkHiParallel(b *testing.B) {
         .          .    115:   r := req(b, "GET / HTTP/1.0\r\n\r\n")
         .          .    116:   b.RunParallel(func(pb *testing.PB) {
         .          .    117:           rw := httptest.NewRecorder()
         .          .    118:           for pb.Next() {
         .    21.34ms    119:                   handleHi(rw, r)
         .          .    120:                   reset(rw)
         .          .    121:           }
         .          .    122:   })
         .          .    123:}
(pprof) list handleHi
Total: 3.59s
ROUTINE ======================== github.com/lsytj0413/ena/example/profile.handleHi in /home/soren/workspace/golang/src/github.com/lsytj0413/ena/example/profile/main.go
         0    21.34ms (flat, cum)  0.59% of Total
         .          .     21:           return regexp.MustCompile(`^\w*$`)
         .          .     22:   },
         .          .     23:}
         .          .     24:
         .          .     25:func handleHi(w http.ResponseWriter, r *http.Request) {
         .    20.56ms     26:   if !colorRxPool.Get().(*regexp.Regexp).MatchString(r.FormValue("color")) {
         .          .     27:           http.Error(w, "Optional color is valid", http.StatusBadRequest)
         .          .     28:           return
         .          .     29:   }
         .          .     30:   visitNum := atomic.AddInt64(&visitors, 1)
         .          .     31:
         .   772.58us     32:   buf := bufPool.Get().(*bytes.Buffer)
         .          .     33:   defer bufPool.Put(buf)
         .          .     34:
         .          .     35:   buf.Reset()
         .          .     36:   buf.WriteString("<h1 style='color: ")
         .          .     37:   buf.WriteString(r.FormValue("color"))
         .          .     38:   buf.WriteString(">Welcome!</h1>You are visitor number ")
         .          .     39:   b := strconv.AppendInt(buf.Bytes(), int64(visitNum), 10)
         .          .     40:   b = append(b, '!')
         .          .     41:   w.Write(b)
         .          .     42:
         .     3.66us     43:   return
         .          .     44:}
         .          .     45:
         .          .     46:func main() {
         .          .     47:   log.Printf("Starting on port 8080")
         .          .     48:   http.HandleFunc("/hi", handleHi)
(pprof)
```

可以看到 MatchString 函数的由 110ms 降低到了 20 ms.

## 覆盖率 ##

可以使用以下命令查看测试覆盖率:

```
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go test -cover -coverprofile=cover
PASS
coverage: 73.7% of statements
ok      github.com/lsytj0413/ena/example/profile        0.024s
soren@SOREN-MIBOOK:~/workspace/golang/src/github.com/lsytj0413/ena/example/profile$ go tool cover -html=cover
```