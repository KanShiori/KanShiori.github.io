# Go 调试方法总结


> **总结系列的文章**是自己的学习或使用后，对相关知识的一个总结，用于后续可以快速复习与回顾。

本文是对 Golang 调试方法的一个总结，基本内容来源于网络的学习，以及自己观摩了下源码。

所以学习的书籍与文章见 [**参考**](#参考)。

## 1 go pprof
Go pprof 是 Go 性能分析工具，在程序运行的过程中，可以记录运行的：CPU、Mem、goroutine 等情况。基本定位 Go 程序的问题第一反应就是使用 go pprof。

pprof 像基本的采集一样，分为采集、上报、分析三个部分。

### 1.1 采集
Go 标准库提供个各个数据的采集接口，可以主动开启 pprof 并将数据写入到 `io.writer` 中。
其包括：
* `pprof.Profiles()` ：返回支持的 profile 的快照。包括：goroutine、threadcreate、heap、allocs、block、mutex；
* `trace.Start()` / `trace.Stop()` ：开启各个事件的追踪，包括 goroutine 状态变更、GC 事件、mheap 大小变更等；
* runtime 包函数；

不过大多数情况下，我们通过使用 `net/http/pprof` 就行了，其注册了各个 HTTP 请求的处理函数，而其函数使用前面的接口进行的数据的采集。
```go
//  net/http/pprof/pprof.go
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```
* `/debug/pprof/` ：引导界面，包含各个子类别的访问；
* `/debug/pprof/cmdline` ：输出程序启动的命令行；
* `/debug/pprof/profile` ：输出大多数 profile 结果，调用 pprof.Profiles()；
* `/debug/pprof/trace` ：一定时间内 go runtime 事件的追踪，调用 trace.Start()；

### 1.2 上报
前面看到，`net/http/pprof` 包就 init 注册对应的 HTTP 请求处理函数，而我们只要在程序中导入 `net/http/pprof` 并**启动一个 HTTP Server** 就可以使用其提供的上报的功能。
```go
package main

import (
	"net/http"
	_ "net/http/pprof"
)

func main() {
	// …
	
	// 启动对应的 HTTP Server
	// 这里我的程序中 main 函数后面会阻塞, 所以用协程启动 server
	go func() {
		if err := http.ListenAndServe("0.0.0.0:12300", nil); err != nil {
			panic(err)
		}
	}()
	
	// …
}
```

#### 1.2.1 浏览器方式
浏览器打开指定的地址的 /debug/pprof/ 页面，可以看到引导页面：
{{< find_img "img1.png" >}}
各个支持的 profile 都有简要的解释，这里翻译一下：
* **`allocs`** ：过去所有的内存分配的样本；
* `block` ：
* `cmdline` ：输出程序执行的命令；
* `goroutine` ：所有 goroutine 当前执行的上下文；
* `heap` ：heap 内存使用的样本；
* `mutex` ：
* **`profile`** ：一定时间内的 CPU profile，可以通过参数指定采集多少时间；
* `threadcreate` ：创建新的系统线程的函数执行上下文，也许在 cgo 中很有用；
* **`trace`** ：一定时间内程序的执行 trace，可以通过参数指定采集时间。

当你点击其中的某一项时，就会得到一个用于 profile 的数据文件，通过 `go tool profile` 或者 `go tool trace` 等命令可以将其可视化。
{{< admonition tip Tip>}}
也许你点击 heap 或者其他会发现浏览器打开了一个新的页面，这是浏览器直接将获取的数据展示了出来，但是其数据可读性很差。

你可以使用 curl 保存数据文件，或者直接通过 `go tool profile <addr>/debug/profile/heap` 来获取文件并解析。
{{< /admonition >}}

### 1.3 解析
通过 go tool pprof 可以解析 pprof 得到的数据文件。`go tool pprof <file>` 执行后会进入一个交互式命令行，其中通过各个命令可以进行数据文件的解析与展示。
常用的命令包括：
* **`top <N>`** ：展示前 N 个数据最大的项；
* `list <packge>.<func>` ：列出某个函数的指标；
* `traces` ：展示各个 goroutine 的调用栈，以及对应的指标；
* **`web`** ：通过浏览器打开解析后的图片；

#### 1.3.1 heap
我们先来看一下 heap 项的数据文件，下载后通过 go tool pprof 的 web 命令打开：
```bash
$ go tool pprof http://10.9.195.138:12300/debug/pprof/heap
```

弹出的浏览器的图片中，展示**各个函数的申请的 heap 占用**。其中，框越大就代表使用的越多，所以我们很快可以找到内存使用最多的函数。
{{< find_img "img2.png" >}}

#### 1.3.2 alloc
对于 heap 项展示的是一瞬间的 heap 使用的快照。但是有一些情况从 heap 项的数据中看不出来，例如一直在快速申请无用内存，而导致 GC 频繁触发。这种情况虽然整体程序内存占用不大，但是是频繁 GC 之后的假象。

**而 alloc 就是用于展示申请内存大小**，这与 heap 展示当前占用内存大小是不一样的。
```bash
$  go tool pprof http://10.9.195.138:12300/debug/pprof/allocs
```
{{< find_img "img3.png" >}}

#### 1.3.3 goroutine
Go 自带内存回收，所以一般不会发生内存泄漏的情况。而**大部分所说的内存泄漏，都是由于 goroutine 泄漏导致的 goroutine 结构占用内存过多**。

如果直接点击浏览器的 goroutine 项中，会展示出所有 goroutine 的上下文。
{{< find_img "img4.png" >}}
但是这不利于分析整体的 goroutine 情况，还是通过 `go tool profile` 命令来看：
```bash
$  go tool pprof http://10.9.195.138:12300/debug/pprof/goroutine
```

在解析后的图片中，**可以看到各个函数创建的 goroutine 的数量**，因此可以很快找到 goroutine 泄漏的情况（图片不是泄漏，只是展示一下）。
{{< find_img "img5.png" >}}

#### 1.3.4 profile
profile 项用于**展示各个函数的 CPU 执行时间**，当你发现程序 CPU 使用异常时，这项非常关键。

默认情况会程序在收到情况后，采集 30s 内的各个函数的 CPU 执行时间。你可以通过设置参数来调整该时间。
```bash
$ go tool pprof http://10.9.195.138:12300/debug/pprof/profile?seconds=30
```

一眼就可以看到执行时间最长的函数流：
{{< find_img "img6.png" >}}


#### 1.3.5 trace
与前面几个不同，trace 可以用于跟踪程序的执行情况。go runtime 会上报特定的 trace 事件，而 trace 将其收集并展示出来。
```bash
$ curl http://10.9.195.138:12300/debug/pprof/trace\?seconds\=30 > trace.out
$ go tool trace trace.out
```

执行命令后，浏览器会弹出一个页面（必须是 Chrome），包含几个分析项，包括：
项目 | 作用
-|-
**View trace**                  | 按照时间维度观察各个事件；
**Goroutine analysis**          | 查看各个 G 的统计信息，包括执行时间，阻塞时间等；
**Network blocking profile**    | 网络接口阻塞情况；
**Synchronization blocking**    | profile：用户态阻塞情况，包括 select、chan 阻塞等；
**Syscall blocking profile**     | 系统调用导致的阻塞情况，最常见的就是 IO 系统调用；
**Scheduler latency profile**   | 各个函数的执行时间情况；
User defined task           | 
User defined regions        | 
Minimum mutator utilization | 展示出了 GC 外，应用能够获得的 CPU 资源的最小比例；

##### (1) View trace
选择 "View trace"，会出现一个以时间为横轴的图片。
{{< find_img "img7.png" >}}
按住 W 放大后可以看到，其中每个 P 执行的 G 的时间段与事件（事件执行是个竖线）都会展示出来。各个 G 直接存在事件，点击 Flow events 后还会展示出事件流。
{{< find_img "img8.png" >}}
* 上面 G76 这个 Goroutine，经过了 P0 执行 -> 系统调用阻塞 -> P2 执行

点击其中一个段，可以看到该 G 的执行的统计信息。
{{< find_img "img9.png" >}}
* `Title`：G 编号以及执行的函数
* `Start`：启动时间；
* `Wall Duration`：执行时间；
* `Start Stack Trace`：G 开始执行的函数栈；
* `End Stack Trace`：G 切换前或者结束的函数栈；

##### (2) Goroutine analysis
选择 "Goroutine analysis" 后，点击一个具体的 G，可以看到其执行时间的统计。
{{< find_img "img10.png" >}}

## 2 delve
pprof 是程序整体的运行情况的数据，如果想**对程序执行进行单步调试，就需要使用 delve**。

delve 类似于 gdb，但是 gdb 是针对于线程的，无法识别 goroutine。而 delve 是专门用于 Go 程序的，可以做到针对不同 goroutine 打断点，单步执行等操作。
