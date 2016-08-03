---
layout: blog
categories: go
title: go程序性能调优（工具篇）
tags: go test
excerpt: go程序性能调优
---

> 最近参加了Gopher China 2016大会，获益颇多，看到各种大拿把go玩的飞起，好是羡慕。于是便先把这两天的收获整理出来，这篇是关于go程序性能调优的。

想要提升程序性能，首先要知道程序的瓶颈在哪里。通过benchmark和pprof可以轻松的对go程序进行性能测试，在讨论性能测试之前，我们先看下普通的单元测试。

我们在fib目录下创建一个fib.go，注意go文件下的package必须与文件夹名一致，否则会识别不到。

```go
package fib

func Fib(n int) int {
    if n < 2 {
        return n
    }
    return Fib(n-1) + Fib(n-2)
}
```

下面我们创建单元测试用例文件fib\_test.go，文件名必须是\*\_test.go的类型，\*代表要测试的文件名，函数名必须以Test开头如：TestXxx或Test\_xxx。

```go
package fib

import "testing"

var fibTests = []struct {
    n        int // input
    expected int // expected result
}{
    {1, 1},
    {2, 1},
    {3, 2},
    {4, 3},
    {5, 5},
    {6, 8},
    {7, 13},
}

func TestFib(t *testing.T) {
    for _, tt := range fibTests {
        actual := Fib(tt.n)
        if actual != tt.expected {
            t.Errorf("Fib(%d): expected %d, actual %d", tt.n, tt.expected, actual)
        }
    }
}
```

输入`go test`就可以对该目录下的所有\*\_test.go进行测试了。我们还可以给该指令加一些参数，比如`-v`可以打印更详细的测试结果（包含Log和Logf等），参数`-cpuprofile=cpu.out`可以将cpu的使用信息写入cpu.out中，当然这些参数都可以通过输入`go help testflag`来查询。

接下来，我们通过benchmark做性能测试。benchmark的测试文件同样必须是以\_test.go结尾，为了区分也可以命名为\*\_b\_test.go，函数名必须以Benchmark开头如：BenchmarkXxx或Benchmark\_xxx。

```go
package fib

import "testing"

func BenchmarkFib(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Fib(40)
    }
}
```

输入`go test bench=.`就可以进行测试了，测试结果如下：

```
PASS
BenchmarkFib-4         2     697262729 ns/op
ok      github.com/terminalworld/fib    2.112s
```

我们还可以指定特定函数进行测试，比如`go test bench=Fib`就是对BenchmarkFib进行测试。上面我们提到可以通过`-cpuprofile=cpu.out`将cpu信息输出到cpu.out，我们来尝试下输入`go test -bench=Fib -cpuprofile=cpu.out`，这时该文件夹下出现两个新文件cpu.out、fib.test。下面我们就可以使用pprof对这两个文件进行分析了，输入`go tool pprof fib.test cpu.out`进入pprof的交互模式。pprof是一个非常强大的工具，可以对该程序的cpu、内存使用情况进行详细的分析。下面是我们输入top后的结果。

```
(pprof) top
1.81s of 1.81s total (100%)
Showing top 10 nodes out of 20 (cum >= 0.01s)
      flat  flat%   sum%        cum   cum%
     1.51s 83.43% 83.43%      1.51s 83.43%  github.com/terminalworld/fib.Fib
     0.29s 16.02% 99.45%      0.29s 16.02%  runtime.usleep
     0.01s  0.55%   100%      0.01s  0.55%  runtime.mach_semaphore_signal
         0     0%   100%      1.51s 83.43%  github.com/terminalworld/fib.BenchmarkFib
         0     0%   100%      0.01s  0.55%  runtime.ReadMemStats
         0     0%   100%      1.52s 83.98%  runtime.goexit
         0     0%   100%      0.01s  0.55%  runtime.mach_semrelease
         0     0%   100%      0.29s 16.02%  runtime.mstart
         0     0%   100%      0.29s 16.02%  runtime.mstart1
         0     0%   100%      0.01s  0.55%  runtime.notewakeup
```

我们还可以调用web（需要安装graphviz）来生成svg文件，然后使用浏览器查看svg文件！下面是该程序cpu使用情况。

<img src="/assets/img/go/pprof-svg.png" width="50%" height="50%">

benchmark和pprof简直不能更赞！通过benchmark和pprof，我们可以很容易找到程序的cpu和内存瓶颈（通过`-memprofile=mem.out`可得到内存使用信息）。比如，本例中绝大多cpu花费在fib.Fib上，我们就可以把关键点放在这个函数上。通过观察我们发现，这个函数是个dp问题，现有时间复杂度为O(2^n)，推导公式是`fib[n] = fib[n-1] + fib[n-2]`，通过记忆之前的结果或者将递归改成递推都可以将复杂度降为O(n)。

```go
package fib

func Fib(n int) int {

    a, b, tmp := 0, 1, 0
    for i := 1; i <= n; i++ {
        tmp = a + b
        a = b
        b = tmp
    }

    return a

}
```

增大数据量后，可以观察到效果更加明显。然而benchmark在面对大型系统时，就有点力不从心了。我们可以使用runtime/pprof包轻松解决这个问题。比如，

```go
import (
    "runtime/pprof"
    "os"
    "fmt"
)

func main() {

    f, err := os.Create("cpu.out")
    if err != nil {
        log.Fatal(err)
    }
    pprof.StartCPUProfile(f)        // 开始cpu profile，结果写到文件f中
    defer pprof.StopCPUProfile()    // 结束profile

    res := Fib(40)
    fmt.Println(res)
}
```

当然也可以使用dave.cheney大神包装好的包[github.com/pkg/profile](https://godoc.org/github.com/pkg/profile)。还有一点需要注意的是**OS X版本在El Capitan之前存在内核bug**，需要升级方能使用pprof。

参考：

[Writing table driven tests in Go](http://dave.cheney.net/2013/06/09/writing-table-driven-tests-in-go)

[How to write benchmarks in Go](http://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)

[Profiling Go Programs](http://blog.golang.org/profiling-go-programs)
