# Go 并发调度总结


> **总结系列的文章**是自己的学习或使用后，对相关知识的一个总结，用于后续可以快速复习与回顾。

本文是对 Golang 并发调度实现的一个总结，基本内容来源于网络的学习，以及自己观摩了下源码。

所以学习的书籍与文章见 [**参考**](#参考)。

下面代码都是基于 go 1.15.6。

## 1 背景知识

### 1.1 进程、线程与协程
看协程的实现之前，绕不开的需要知道进程与线程，以及它们之间的比较。

[进程]^(progress) 就是程序执行的实例。在内核角度上看，进程是**分配系统资源的实体**。

而这也可以推出，所谓的进程切换的消耗，也就是切换分配的系统资源，包括**虚拟内存（页表等）**、**寄存器的值**等。
{{< admonition tip Tip>}}
据说每次上下文切换需要几十纳秒到微秒的 CPU 时间。
{{< /admonition >}}

[线程]^(thread) 是**内核调度的基本单元**，在 Linux 上，线程就是 “轻量级进程”，因为它与进程在内核的看来都一样，仅仅是共享了一些资源，包括：
* 内存地址空间；
* 进程的基础信息；
* 打开的文件描述符；
* 信号处理；
* 等等

所以相对于进程间切换，同一个进程下的线程切换，可以**省略这些共享的资源的切换**。因此，线程间切换速度快于进程间切换。
{{< admonition note Note>}}
所谓进程切换就是不同进程间的线程切换，所以下面所说的线程间的切换、线程上下文都是指同进程下的线程。
{{< /admonition >}}

上面所说的调度、切换都是站在内核的角度看线程。而站在线程的角度，它可以认为自己是 **“完全” 占用 CPU 的**。如果线程阻塞等待，就等于 CPU 在 ~~空闲~~ 浪费。

所以，写代码往往会使用异步 API，**通过回调/通知来使得线程阻塞的更加少**（典型的 epoll），这时候 CPU 原来阻塞的时间可以执行其他的代码。

但是，回调的代码不是同一个函数里线性执行的，会有一些缺点，具体例子，A 函数最初的执行流为：
* 执行 -> 读取数据 -> 执行

如果使用异步 API，其执行流程变为：
* A 函数：执行 -> 异步 -> 返回
* 回调函数：读取数据 -> 执行。

首先，单个函数的执行流变为了两个不同函数，代码写法上串行的思维要变为分割的思维。并且，如果这个操作还是有状态的，那么还涉及到了 “A 函数将状态传递到回调函数” 等问题。
{{< find_img "img1.png" >}}


因此，[协程]^(routine) 出现，上面的例子的执行流还是：
* 执行 -> 读取数据 -> 执行

但是，**在读取数据这一步，协程会主动让出 CPU，等待数据到来时再次切换到该协程**，继续执行。因此，写法不变，功能相同。

并且，在一些用户空间的生产消费模型实现上（channel），协程的阻塞不需要线程阻塞，而在用户空间就完成了协程的切换。

### 1.2 调度器
[调度器]^(scheduler) 是为了在决定在有限的 CPU 上，选择哪些任务（进程/线程/协程）执行使得系统运行更高效，所以主要有两个工作：
1. **决定为各个任务决定其运行多长时间，以及何时切换到下个任务执行**；
1. **在切换任务 A 至任务 B 时，必须保存任务 A 的运行环境，并且任务 B 的运行环境与上次其被切换时完全相同**；

第 1 个工作涉及到的就是 **`调度算法`**，如何调度使得系统运行的最高效。

第 2 个工作涉及到的就是 **`上下文切换`**，这里再说一下进程、线程、协程的上下文：
* 进程：虚拟内存、寄存器的值、内核对应进程信息等；（内核完成上下文切换）
* 线程：寄存器的值、内核对应的进程信息等；（内核完成上下文切换）
* 协程：寄存器的值、用户空间对应的协程信息等；（用户空闲 runtime 完成协程上下文切换）

{{< admonition note Note>}}
上面的切换没有说栈，因为栈的切换就是寄存器的切换。
{{< /admonition >}}

### 1.3 PC 与 SP
这一块在 [**内存管理总结**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/#2-pc-%E4%B8%8E-sp) 说过，因为对协程切换很重要，这个再次复制一下。

#### 1.3.1 PC
[程序计数器 PC]^(Program Counter) 是 CPU 中的一个寄存器，**保存着下一个 CPU 执行的指令的位置**。顺序执行指令时，PC = PC + 1（一个指令）。而调用函数或者条件跳转时，会将跳到的指令地址设置到 PC 中。

所以，可以想到，当需要切换执行的 goroutine，调用 JMP 指令跳转到 G 对应的代码。

#### 1.3.2 SP
[栈顶指针 SP]^(stack pointer) 是**保存栈顶地址的寄存器**，我们平时所说的临时变量在栈上，就是将临时变量的值写入 SP 保存的内存地址，然后 SP 保存的地址减小（栈是从高地址向低地址变化），然后临时变量销毁时，SP 地址又变为高地址。

不过，因为 goroutine 切换时，必须要保存当前 goroutine 的上下文，也就是栈里的变量。因此，goroutine 栈肯定是不能使用 Linux 进程栈了（因为进程栈有上限，也无法实现“保存”这种功能）。所以所说的**协程栈，都是基于 mmap 申请内存空间**（基于 Go 内存管理，内存管理基于 mmap），然后**切换时修改 SP 寄存器地址实现的**。

这也是为什么 goroutine 栈可以“无限大”的原因了。

## 2 GMP 模型
先从整体模型入手，整个协程实现有着三个最主要的对象：
* **`G`** ：表示 Goroutine，**保存并发任务的状态（上下文）**；
* **`M`** ：表示**系统线程，必须在绑定 P 后，执行 G 任务**；
* **`P`** ：处理器，作用类似于 CPU 核，**控制并发执行任务数量**；
* **`schedt`** ：全局的链表，包括**全局的 G 可运行链表、空闲 M 链表、空闲 P 链表**；

而大致的运行的模型如下：
{{< find_img "img2.png" >}}

* 每个 P 绑定一个 M，与一个正在运行的 G；
* 每个 P 包含自己本地的可运行的 G 的链表；
* 全局的 G 的可运行链表，用以提供给无 G 可以执行的 P；
* 一些 M 与 G 的绑定，因为系统调用阻塞而解绑了 P，M 阻塞结束后还是会进入调度循环；
* 一些 G 主动进入 waiting 状态，等待唤醒后重新加入某个可运行队列（典型的，读取 channel 阻塞，这是一个用户空间的阻塞，由发送 channel 时将其唤醒）；
* 空闲 M 链表，空闲 P 链表，空闲 G 链表等；

### 2.1 G
**`g 结构`**（runtime/runtime2.go）代表一个 G，包含了某个任务的上下文，M 切换 G 执行时，当前 G 的上下文就保存在了 g 结构中。
```go
type g struct {
        // Stack parameters.
        // stack describes the actual stack memory: [stack.lo, stack.hi).
        // stackguard0 is the stack pointer compared in the Go stack growth prologue.
        // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
        // stackguard1 is the stack pointer compared in the C stack growth prologue.
        // It is stack.lo+StackGuard on g0 and gsignal stacks.
        // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
        stack       stack   // offset known to runtime/cgo
        stackguard0 uintptr // offset known to liblink
        stackguard1 uintptr // offset known to liblink

        _panic       *_panic // innermost panic - offset known to liblink
        _defer       *_defer // innermost defer
        m            *m      // current m; offset known to arm liblink
        sched        gobuf
        
        atomicstatus uint32
        goid         int64
        
        preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
        preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
        preemptShrink bool // shrink stack at synchronous safe point

       lockedm        muintptr 
        …
}
```
* `stack`：G 对应栈空间的地址，见（内存管理模型-Goroutine 的栈）；
* **`stackguar0`**：扩容栈的地址，也可以用于判断 G 是否应该被抢占；当 stackguard0 == stackpreempt 就表明当前 G 被抢占了；
* `_panic`：G 下的 panic 链表；
* `_defer`：G 下的所有 defer 组成的链表；
* **`m`**：当前绑定的 M，为 nil 就表示当前 G 没有在执行；
* **`sched`**：G 的部分上下文，会提供给汇编代码；
* **`atomicstatus`**：G 的状态；
* `goid`：G 的唯一 ID，但是用户代码无法读取；
* `preempt`：抢占标志；
* **`lockedm`**：记录 G 被锁定的 M，实现 runtime.LockOSThread()

其中 g.sched 是一个非常重要的结构，需要看一下其 **`gobuf`** 的实现（runtime/runtime2.go）：
```go
type gobuf struct {
        // The offsets of sp, pc, and g are known to (hard-coded in) libmach.
        //
        // ctxt is unusual with respect to GC: it may be a
        // heap-allocated funcval, so GC needs to track it, but it
        // needs to be set and cleared from assembly, where it's
        // difficult to have write barriers. However, ctxt is really a
        // saved, live register, and we only ever exchange it between
        // the real register and the gobuf. Hence, we treat it as a
        // root during stack scanning, which means assembly that saves
        // and restores it doesn't need write barriers. It's still
        // typed as a pointer so that any other writes from Go get
        // write barriers.
        sp   uintptr
        pc   uintptr
        g    guintptr
        ctxt unsafe.Pointer
        ret  sys.Uintreg
        …
}
```
* **`sp`**：当前 G 的 SP 指针地址，在创建 G 时默认设置为 goexit 函数地址；
* **`pc`**：当前 G 的 PC 指针地址；
* `g`：gobuf 所属的 g 结构地址；
* `ret`：系统调用的返回值；

#### 2.1.1 G 的状态
目前 G 可能处于以下 9 种状态：
状态 | 值 | 含义
-|-|- 
_Gidle |0 | G 刚分配，并且还没有被初始化
_Grunnable |1 | G 处于可运行队列中，但是并没有被执行
_Grunning |2 | G 绑定了 M、P，不在可运行队列，并且可能正在执行
_Gsyscall |3 | G 正在执行系统调用而阻塞，不在可运行队列上，绑定了 M，没有绑定 P
_Gwaiting |4 | G 因为用户空间而阻塞，不在可运行队列，等待其他 G 唤醒，没有绑定 M、P
_Gdead |5 | G 当前没有被使用，其 g 结构可以被复用
_Gcopystack |6 | G 的栈正在被拷贝，没有执行，不再可运行队列
_Gpreempted |7 |由于抢占而被阻塞，没有执行，等待唤醒
_Gscan |8 | GC 正在扫描 G 的栈，可与其他状态同时存在

其状态轮转如下图所示：
{{< find_img "img3.png" >}}


#### 2.1.2 G 的创建
在代码中调用 go 语句时，编译器会将其翻译为 `newproc()` 调用，这也就是创建 G 的开端：
```go
func newproc(siz int32, fn *funcval) {
        argp := add(unsafe.Pointer(&fn), sys.PtrSize)
        gp := getg()
        pc := getcallerpc()
        systemstack(func() {
                // 创建 g 结构
                newg := newproc1(fn, argp, siz, gp, pc)

        // 放入当前 P 或者全局可运行队列
        _p_ := getg().m.p.ptr()
        runqput(_p_, newg, true)

        if mainStarted {
                wakep()
        }
        })
}

// Create a new g in state _Grunnable, starting at fn, with narg bytes
// of arguments starting at argp. callerpc is the address of the go
// statement that created this. The caller is responsible for adding
// the new g to the scheduler.
//
// This must run on the system stack because it's the continuation of
// newproc, which cannot split the stack.
//
//go:systemstack
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
        // 调用 go 语句的 G
        _g_ := getg()
        siz := narg
        siz = (siz + 7) &^ 7

        // 从 P.gFree 获取一个可复用的 g 对象
        _p_ := _g_.m.p.ptr()
        newg := gfget(_p_)
        
        // 没有可复用的，重新分配一个 g 对象
        if newg == nil {
                newg = malg(_StackMin)
                casgstatus(newg, _Gidle, _Gdead)
                allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
        }

        totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
        totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
        sp := newg.stack.hi - totalSize
        spArg := sp
        if usesLR {
                // caller's LR
                *(*uintptr)(unsafe.Pointer(sp)) = 0
                prepGoExitFrame(sp)
                spArg += sys.MinFrameSize
        }
        if narg > 0 {
                // 
                memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
                // This is a stack-to-stack copy. If write barriers
                // are enabled and the source stack is grey (the
                // destination is always black), then perform a
                // barrier copy. We do this *after* the memmove
                // because the destination stack may have garbage on
                // it.
                if writeBarrier.needed && !_g_.m.curg.gcscandone {
                        f := findfunc(fn.fn)
                        stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
                        if stkmap.nbit > 0 {
                                // We're in the prologue, so it's always stack map index 0.
                                bv := stackmapdata(stkmap, 0)
                                bulkBarrierBitmap(spArg, spArg, uintptr(bv.n)*sys.PtrSize, 0, bv.bytedata)
                        }
                }
        }

        // 初始化 G 的属性
        // pc 指针的设置是一个比较关键的，将 goexit 地址保存在 sched 中，用于 G 函数结束后可以平滑执行 goexit
        memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
        newg.sched.sp = sp
        newg.stktopsp = sp
        newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
        newg.sched.g = guintptr(unsafe.Pointer(newg))
        gostartcallfn(&newg.sched, fn)
        newg.gopc = callerpc
        newg.ancestors = saveAncestors(callergp)
        newg.startpc = fn.fn
        
        // G 由 _Gdead -> _Grunnable
        casgstatus(newg, _Gdead, _Grunnable)

        return newg
}
```
1. 构建 G 对应的 g 结构；
    1. 尝试**从 `P.gFree` 链表中获取** _GDead 的 G。如果没有，走 `mlag()` **分配一个新的 g**；
    1. 拷贝 fn 函数的所有参数到栈上；
    1. 设置 g 的属性（比较关键的就是 g.sched 属性），可以看到**将 g.sched.sp 设置为了 goexit 函数的地址，这在调度循环中至关重要**；
    1. G 状态由 _GDead -> _Grunnable；
1. `runqput()` 将新的 **G 放入 P 或者全局的可运行队列**；

#### 2.1.3 G 上下文的保存
在切换 G 之间，一个必要的操作就是将 G 的 PC/SP 指针保存到 `g.sched` 中，在源码中有两个函数负责这个行为。

**`mcall()`** 用于在 runtime 运行期间，将一个 M 绑定的 G 切换。**它将 M 切换到 g0 栈，然后执行 G 上下文的保存**，其实现是一段汇编代码：
```go
// func mcall(fn func(*g))
// Switch to m->g0's stack, call fn(g).
// Fn must never return. It should gogo(&g->sched)
// to keep running g.
TEXT runtime·mcall(SB), NOSPLIT, $0-8
	        MOVQ        fn+0(FP), DI

	        get_tls(CX)
	        MOVQ        g(CX), AX        // save state in g->sched
	        MOVQ        0(SP), BX        // caller's PC
	        MOVQ        BX, (g_sched+gobuf_pc)(AX)
	        LEAQ        fn+0(FP), BX        // caller's SP
	        MOVQ        BX, (g_sched+gobuf_sp)(AX)
	        MOVQ        AX, (g_sched+gobuf_g)(AX)
	        MOVQ        BP, (g_sched+gobuf_bp)(AX)

	        // switch to m->g0 & its stack, call fn
	        MOVQ        g(CX), BX
	        MOVQ        g_m(BX), BX
	        MOVQ        m_g0(BX), SI
	        CMPQ        SI, AX        // if g == m->g0 call badmcall
	        JNE        3(PC)
	        MOVQ        $runtime·badmcall(SB), AX
	        JMP        AX
	        MOVQ        SI, g(CX)        // g = m->g0
	        MOVQ        (g_sched+gobuf_sp)(SI), SP        // sp = m->g0->sched.sp
	        PUSHQ        AX
	        MOVQ        DI, DX
	        MOVQ        0(DI), DI
	        CALL        DI
	        POPQ        AX
	        MOVQ        $runtime·badmcall2(SB), AX
	        JMP        AX
	                RET
```
1. **保存当前的 SP、PC 至 `g.sched` 属性中**；
1. **切换到 m.g0 栈，执行传入的函数**；

`save()` 用于 G 进入系统调用阻塞前，保存 G 的上下文，**用于系统调用结束后，进行相关信息的恢复**。
```go
func save(pc, sp uintptr) {
        _g_ := getg()

        _g_.sched.pc = pc
        _g_.sched.sp = sp
        _g_.sched.lr = 0
        _g_.sched.ret = 0
        _g_.sched.g = guintptr(unsafe.Pointer(_g_))
        // We need to ensure ctxt is zero, but can't have a write
        // barrier here. However, it should always already be zero.
        // Assert that.
        if _g_.sched.ctxt != nil {
                badctxt()
        }
}
```

### 2.2 M
M 代表的就是一个操作系统线程，由 **`m 结构`** 表示（runtime/runtime2.go）：
```go
type m struct {
        g0      *g     // goroutine with scheduling stack
        
        gsignal       *g           // signal-handling g
        goSigStack    gsignalStack // Go-allocated signal handling stack
        
        curg          *g       // current running goroutine
        
        p             puintptr // attached p for executing go code (nil if not executing go code)
        nextp         puintptr
        oldp          puintptr // the p that was attached before executing a syscall
        id            int64
        
        lockedg       guintptr
        lockedExt     uint32      // tracking for external LockOSThread
        lockedInt     uint32      // tracking for internal lockOSThread
}
```
* **`g0`** : M 私有的 g0；
* `gsignal` ：用于处理操作系统信号的 G；
* **`curg`** ：当前 M 正在执行的 G；
* **`p`** : 当前 M 绑定的 P；
* `nextp` ：暂存的 nextp；
* `oldp` ：M 陷入系统调用前绑定的 P，用于系统调用结束后尝试重新绑定；
* `id` ：m 的 ID；
* **`lockedg`** ：保存 M 锁定的 G，实现 runtime.LockOSThread()；

#### 2.2.1 M 的创建
在创建 G 或者其他地方，当 G 变为 runnable 后，就会调用 `wakep()` 触发一次 P 执行 G 的过程。其会调用 `startm()` **选择/创建 一个 M 绑定 P**，并执行一个 G。
```go
// Tries to add one more P to execute G's.
// Called when a G is made runnable (newproc, ready).
func wakep() {
        if atomic.Load(&sched.npidle) == 0 {
                return
        }
        startm(nil, true)
}

// Schedules some M to run the p (creates an M if necessary).
// If p==nil, tries to get an idle P, if no idle P's does nothing.
// May run with m.p==nil, so write barriers are not allowed.
// If spinning is set, the caller has incremented nmspinning and startm will
// either decrement nmspinning or set m.spinning in the newly started M.
//go:nowritebarrierrec
func startm(_p_ *p, spinning bool) {
        // 选择一个空闲的 P
        lock(&sched.lock)
        if _p_ == nil {
                _p_ = pidleget()
                if _p_ == nil {
                        unlock(&sched.lock)
                        return
                }
        }
        
        // 选择一个空闲的 M
        mp := mget()
        if mp == nil {
                // No M is available, we must drop sched.lock and call newm.
                // However, we already own a P to assign to the M.
                //
                // Once sched.lock is released, another G (e.g., in a syscall),
                // could find no idle P while checkdead finds a runnable G but
                // no running M's because this new M hasn't started yet, thus
                // throwing in an apparent deadlock.
                //
                // Avoid this situation by pre-allocating the ID for the new M,
                // thus marking it as 'running' before we drop sched.lock. This
                // new M will eventually run the scheduler to execute any
                // queued G's.
                id := mReserveID()
                unlock(&sched.lock)

        var fn func()
        if spinning {
                // The caller incremented nmspinning, so set m.spinning in the new M.
                fn = mspinning
        }
        // 创建一个 M
        newm(fn, _p_, id)
        return
        }
        unlock(&sched.lock)
        
        // The caller incremented nmspinning, so set m.spinning in the new M.
        mp.spinning = spinning
        mp.nextp.set(_p_)
        notewakeup(&mp.park)
}
```
* 当存在空闲的 P，但是没有空闲的 M 时，就会调用 `newm()` 创建一个 M；

`newm()` 就是**创建 m 结构，以及启动系统线程的地方**（runtime/proc.go）：
```go
// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//
// id is optional pre-allocated m ID. Omit by passing -1.
//go:nowritebarrierrec
func newm(fn func(), _p_ *p, id int64) {
        // 分配 m 对象（复用 sched.freem 或者新创建）
        mp := allocm(_p_, fn, id)
        mp.nextp.set(_p_)
        mp.sigmask = initSigmask
        
        …
        newm1(mp)
}

func newm1(mp *m) {
        …
        execLock.rlock() // Prevent process clone.
        newosproc(mp)
        execLock.runlock()
}

// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrier
func newosproc(mp *m) {
        stk := unsafe.Pointer(mp.g0.stack.hi)
        
        // 给 thread 初始的栈来自于 g0
        // Disable signals during clone, so that the new thread starts
        // with signals disabled. It will enable them in minit.
        var oset sigset
        sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
        ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
        sigprocmask(_SIG_SETMASK, &oset, nil)

        if ret < 0 {
                throw("newosproc")
        }
}
```
1. 获取 m 对象，来自于 `sched.freem` 或者新创建一个；
1. 通过 `clone()` 系统调用创建一个系统线程，执行的函数为 `mstart()`；

看一下 clone 的 flags：
```go
var (
        cloneFlags = _CLONE_VM | /* share memory */
                _CLONE_FS | /* share cwd, etc */
                _CLONE_FILES | /* share fd table */
                _CLONE_SIGHAND | /* share sig handler table */
                _CLONE_SYSVSEM | /* share SysV semaphore undo lists (see issue #20763) */
                _CLONE_THREAD /* revisit - okay for now */
)
```

#### 2.2.2 M 的启动
前面看到，M 线程启动后执行 `mstart()` 函数：
```go
func mstart() {
        _g_ := getg()

        // 确定栈边界
        osStack := _g_.stack.lo == 0
        if osStack {
                // Initialize stack bounds from system stack.
                // Cgo may have left stack size in stack.hi.
                // minit may update the stack bounds.
                size := _g_.stack.hi
                if size == 0 {
                        size = 8192 * sys.StackGuardMultiplier
                }
                _g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
                _g_.stack.lo = _g_.stack.hi - size + 1024
        }
        _g_.stackguard0 = _g_.stack.lo + _StackGuard
        _g_.stackguard1 = _g_.stackguard0
        
        // 准备进入调度循环
        mstart1()

        // 线程退出
        // Exit this thread.
        switch GOOS {
        case "windows", "solaris", "illumos", "plan9", "darwin", "aix":
                osStack = true
        }
        mexit(osStack)
}

func mstart1() {
	    // 这是 g0
        _g_ := getg()

        // Record the caller for use as the top of stack in mcall and
        // for terminating the thread.
        // We're never coming back to mstart1 after we call schedule,
        // so other calls can reuse the current frame.
        save(getcallerpc(), getcallersp())
        asminit()
        minit()

        // m0 需要的信号初始化工作
        // Install signal handlers; after minit so that minit can
        // prepare the thread to be able to handle the signals.
        if _g_.m == &m0 {
                mstartm0()
        }

        if fn := _g_.m.mstartfn; fn != nil {
                fn()
        }

        // 绑定前面存下的 P
        if _g_.m != &m0 {
                acquirep(_g_.m.nextp.ptr())
                _g_.m.nextp = 0
        }
        
        // 进入调度循环，不再返回
        schedule()
}
```
上面的关键就是，**M 最后绑定 P，调用 schedule() 进入调度循环**，并且不会再返回。
{{< admonition tip m0>}}
m0 是程序启动的 main M
{{< /admonition >}}

#### 2.2.3 M 的退出
前面 `mexit()` 就是用于**释放 M 资源，m 结构记录在 `sched.freem`，并使得线程退出**。

但是，M 进入调度循环后，就不会返回了，所以需要使用调用 g0 保存的栈帧实现。

当然，M 线程退出只是在特定的场景（见 [**goexit()**](#33-goexit)）会出现：调用 `runtime.LockOSThread()` 锁定 M 后，没有调用 `runtime.UnlockOSThread()` 解锁， G 用户函数执行完毕后退出，M 对应线程退出。

{{< admonition note 线程是否会减少>}}
Linux 上 go 进程的线程数量可能减少，当你调用 LockOSThread() 而没有解锁，执行完 groutine 代码后，对应线程会退出。
{{< /admonition >}}
```go
// mexit tears down and exits the current thread.
//
// Don't call this directly to exit the thread, since it must run at
// the top of the thread stack. Instead, use gogo(&_g_.m.g0.sched) to
// unwind the stack to the point that exits the thread.
//
// It is entered with m.p != nil, so write barriers are allowed. It
// will release the P before exiting.
//
//go:yeswritebarrierrec
func mexit(osStack bool) 
```

#### 2.2.4 M 与 G 的锁定
通过 `runtime.LockOSThread()` 与 `runtime.UnlockOSThread()` 实现将 M 与 G 的锁定与解锁。

锁定后，该 M 只能执行锁定的 G，并且不会执行其他的 G 与被调度。
```go
func LockOSThread() {
        if atomic.Load(&newmHandoff.haveTemplateThread) == 0 && GOOS != "plan9" {
                // If we need to start a new thread from the locked
                // thread, we need the template thread. Start it now
                // while we're in a known-good state.
                startTemplateThread()
        }
        _g_ := getg()
        _g_.m.lockedExt++
        if _g_.m.lockedExt == 0 {
                _g_.m.lockedExt--
                panic("LockOSThread nesting overflow")
        }
        dolockOSThread()
}

func dolockOSThread() {
        if GOARCH == "wasm" {
                return // no threads on wasm yet
        }
        _g_ := getg()
        _g_.m.lockedg.set(_g_)
        _g_.lockedm.set(_g_.m)
}
```
将 G 记录在 `m.lockedg`，将 M 记录在 `g.lockedm` 实现了 M 与 G 的映射关系，实现锁定的功能。

### 2.3 P
P 是 M 与 G 的中间层，没有了 P，M 与 G 实际上就是一个线程池。**通过 P 来分片所有可运行的 G，使得运行效率更加的高**。

**`p`** 是 P 的结构体表示（runtime/runtime2.go）：
```go
type p struct {
        id          int32
        status      uint32 // one of pidle/prunning/...
        
        m           muintptr   // back-link to associated m (nil if idle)
        mcache      *mcache
        pcache      pageCache

        // Queue of runnable goroutines. Accessed without lock.
        runqhead uint32
        runqtail uint32
        runq     [256]guintptr
        // runnext, if non-nil, is a runnable G that was ready'd by
        // the current G and should be run next instead of what's in
        // runq if there's time remaining in the running G's time
        // slice. It will inherit the time left in the current time
        // slice. If a set of goroutines is locked in a
        // communicate-and-wait pattern, this schedules that set as a
        // unit and eliminates the (potentially large) scheduling
        // latency that otherwise arises from adding the ready'd
        // goroutines to the end of the run queue.
        runnext guintptr

        // Available G's (status == Gdead)
        gFree struct {
                gList
                n int32
        }

        sudogcache []*sudog
        sudogbuf   [128]*sudog
}
```
* `status` ：P 的状态；
* **`m`** ：当前绑定的 M；
* `mcache` ：P 唯一的 mcache，将【内存管理】；
* `runqhead` ：P 的可运行队列的 head index；
* `runqtail` ：P 的可以行队列的 tail index；
* **`runq`** ：P 的可运行队列，可以看到大小为 256；
* `runnext`  ：下一次优先执行的 G，优先级高于 runq；
* **`gFree`** ：可复用的 g 结构链表；

P 还包含大量与 GC 内存管理相关的字段，这里暂时省略。

#### 2.3.1 P 的状态
状态 | 值 | 含义
-|-|- 
_Pidle |0 | P 没有任何 G 可以执行，被空闲 P 链表持有着
_Prunning |1 | P 绑定了一个 M，并且正在执行 G 
_Psyscall |2 | P 绑定了 M，但是 M 陷入系统调用阻塞，P 可以被其他 M 获取
_Pgcstop |3 | P 绑定着 M，但是因为 STW 挂起
_Pdead |4 | P 不在被使用，由于 GOMAXPROCS 缩小

可以看到，_Pidle 与 _Psyscall 都属于 P 可以被其他 M 绑定的状态。

#### 2.3.2 P 的创建/销毁
前面可以看到，G 是由 `go` 命令创建的，而 M 是按需创建的。P 的创建不同，因为其代表的是并发个数，所以其创建是**在程序启动时创建**。

在执行用户 main 函数之间的 `scheinit()` 中进行 runtime 的初始化，其中一项就是初始化 P:
```go
// The bootstrap sequence is:
//
//        call osinit
//        call schedinit
//        make & queue new G
//        call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
        // …

        procs := ncpu
        if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
                procs = n
        }
        if procresize(procs) != nil {
                throw("unknown runnable goroutine during bootstrap")
        }

        // …
}
```

`procresize()` 用于改变 P 的数量（runtime/proc.go）：
```go
// Change number of processors. The world is stopped, sched is locked.
// gcworkbufs are not being modified by either the GC or
// the write barrier code.
// Returns list of Ps with local work, they need to be scheduled by the caller.
func procresize(nprocs int32) *p {
        old := gomaxprocs
        
        // 扩大 P 的数量
        // Grow allp if necessary.
        if nprocs > int32(len(allp)) {
                // Synchronize with retake, which could be running
                // concurrently since it doesn't run on a P.
                lock(&allpLock)
                if nprocs <= int32(cap(allp)) {
                        allp = allp[:nprocs]
                } else {
                        nallp := make([]*p, nprocs)
                        // Copy everything up to allp's cap so we
                        // never lose old allocated Ps.
                        copy(nallp, allp[:cap(allp)])
                        allp = nallp
                }
                unlock(&allpLock)
        }

        // 初始化新的 P
        // initialize new P's
        for i := old; i < nprocs; i++ {
                pp := allp[i]
                if pp == nil {
                        pp = new(p)
                }
                pp.init(i)
                atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
        }

        // 如果当前 M 绑定的 P 是要被释放的，那么 M 新选取一个可用的 P
        _g_ := getg()
        if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
                // continue to use the current P
                _g_.m.p.ptr().status = _Prunning
                _g_.m.p.ptr().mcache.prepareForSweep()
        } else {
                // release the current P and acquire allp[0].
                //
                // We must do this before destroying our current P
                // because p.destroy itself has write barriers, so we
                // need to do that from a valid P.
                if _g_.m.p != 0 {
                        _g_.m.p.ptr().m = 0
                }
                _g_.m.p = 0
                p := allp[0]
                p.m = 0
                p.status = _Pidle
                acquirep(p)
                if trace.enabled {
                        traceGoStart()
                }
        }

        // g.m.p is now set, so we no longer need mcache0 for bootstrapping.
        mcache0 = nil

        // 清理不使用的 P
        // release resources from unused P's
        for i := nprocs; i < old; i++ {
                p := allp[i]
                p.destroy()
                // can't free P itself because it can be referenced by an M in syscall
        }

        // Trim allp.
        if int32(len(allp)) != nprocs {
                lock(&allpLock)
                allp = allp[:nprocs]
                unlock(&allpLock)
        }

        var runnablePs *p
        for i := nprocs - 1; i >= 0; i-- {
                p := allp[i]
                if _g_.m.p.ptr() == p {
                        continue
                }
                p.status = _Pidle
                if runqempty(p) {
                        pidleput(p)
                } else {
                        p.m.set(mget())
                        p.link.set(runnablePs)
                        runnablePs = p
                }
        }
        stealOrder.reset(uint32(nprocs))
        var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
        atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
        return runnablePs
}
```

#### 2.3.3 P 数量调整
通过 `runtime.GOMAXPROCS()` 接口可以动态调整程序的并发数，也就是 P 的数量，看一下其实现（runtime/debug.go）：
```go
func GOMAXPROCS(n int) int {
        lock(&sched.lock)
        ret := int(gomaxprocs)
        unlock(&sched.lock)
        if n <= 0 || n == ret {
                return ret
        }
        
        // STW !
        stopTheWorldGC("GOMAXPROCS")

        // 调整全局变量
        // newprocs will be processed by startTheWorld
        newprocs = int32(n)
        
        // 恢复，恢复时会进行 P 的缩扩
        startTheWorldGC()
        return ret
}
```
1. stopTheWorld；
1. 调整全局变量 `newprocs` 的值，而该值后续在恢复时会被使用；
1. startTheWorld 恢复，恢复中一个步骤就是根据 `newprocs` 调用 `procresize()` 函数进行调整；

#### 2.3.4 P 的可运行队列
`p.runq` 保存着 P 独有的 runnable G，`p.runnext` 为最高优先级的 G。在下面源码中，经常可以看到调用 `runqget()` 从可运行队列得到一个 G:
```go
// Get g from local runnable queue.
// If inheritTime is true, gp should inherit the remaining time in the
// current time slice. Otherwise, it should start a new time slice.
// Executed only by the owner P.
func runqget(_p_ *p) (gp *g, inheritTime bool) {
        // 返回 runnext
        for {
                next := _p_.runnext
                if next == 0 {
                        break
                }
                if _p_.runnext.cas(next, 0) {
                        return next.ptr(), true
                }
        }
        
        // 从 runq 获取
        for {
                h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
                t := _p_.runqtail
                if t == h {
                        return nil, false
                }
                gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
                // head 变动
                if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
                        return gp, false
                }
        }
}
```

当一个 G 新创建，或者从 _Gwaiting -> _Grunning 时，G 会有加入到 P 的可运行队列，调用 `runqput()` 函数：
```go
// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(_p_ *p, gp *g, next bool) {
	// 放置到 runnext
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}

	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	// 放到 runq
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	
	// 本地 P 放不下了，放置到全局运行队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```
1. 如果指定，放置到 `q.runnext`；
1. 放置到 `q.runq`；
1. 如果**本地队列满了，那么放置到全局运行队列** `sched.runq`。为了优化效率，**每次会转移一半的本地队列 G**；

### 2.4 schedt
**`schedt`** 是一个单例全局变量，包含一些全局的链表（runtime/runtime2.go）：
```go
var sched schedt

type schedt struct {
        // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
        // sure to call checkdead().

        midle        muintptr // idle m's waiting for work
        nmidle       int32    // number of idle m's waiting for work

        pidle      puintptr // idle p's
        npidle     uint32

        // Global runnable queue.
        runq     gQueue
        runqsize int32

        // disable controls selective disabling of the scheduler.
        //
        // Use schedEnableUser to control this.
        //
        // disable is protected by sched.lock.
        disable struct {
                // user disables scheduling of user goroutines.
                user     bool
                runnable gQueue // pending runnable Gs
                n        int32  // length of runnable
        }

        // Global cache of dead G's.
        gFree struct {
                lock    mutex
                stack   gList // Gs with stacks
                noStack gList // Gs without stacks
                n       int32
        }

        // Central cache of sudog structs.
        sudoglock  mutex
        sudogcache *sudog

        // Central pool of available defer structs of different sizes.
        deferlock mutex
        deferpool [5]*_defer

        // freem is the list of m's waiting to be freed when their
        // m.exited is set. Linked through m.freelink.
        freem *m
}
```
* **`midle`** ：**空闲的 M 的链表**；
* **`pidle`** ：**空闲的 P 的链表**；
* **`runq`** ：**全局的可运行的 G 队列**；
* **`gFree`** ：全局的可复用的 g 结构队列；
* `freem` ：等待释放的 M 的链表，当 M 新建时会将其释放，并复用 m 对象；

## 3 调度循环
前面 M 的启动看到，每个 M 创建后会进入 `schedule()` 的调度循环，并且不会返回，每个 M 执行大致流程如下：
{{< find_img "img4.png" >}}
1. 执行 `schedule()`， **使得 M 找到一个可用的 G，并绑定**；
1. 执行 `execute()`，**完成执行 G.fn 的准备工作**；
1. **调用 G 的函数**；
1. G 函数调用退出后，调用 `goexit()` 函数**清理相关资源，并重新进入 `schedule()`**；

当然，上面是正常 G 执行并退出的逻辑，多数情况下 G 执行的过程中都会经历抢占与调度，也就是说会 M 可能会切换 P、切换 G 执行。

### 3.1 schedule()
`schedule()` 是调度循环的第一步，M 会在这里尽力寻找一个 runnable G，然后进入 `execute() `执行 G 的代码：
```go
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
        _g_ := getg()
        
        // 锁定了 G, 那么还是执行锁定的 G
        if _g_.m.lockedg != 0 {
                stoplockedm()
                execute(_g_.m.lockedg.ptr(), false) // Never returns.
        }

top:
        pp := _g_.m.p.ptr()
        pp.preempt = false

        // STW 中，等待结束
        if sched.gcwaiting != 0 {
                gcstopm()
                goto top
        }
        
        // 运行 timer
        checkTimers(pp, 0)

        // gp 为新调度到的 G
        var gp *g
        var inheritTime bool

        // Normal goroutines will check for need to wakeP in ready,
        // but GCworkers and tracereaders will not, so the check must
        // be done here instead.
        tryWakeP := false
        if trace.enabled || trace.shutdown {
                gp = traceReader()
                if gp != nil {
                        casgstatus(gp, _Gwaiting, _Grunnable)
                        traceGoUnpark(gp, 0)
                        tryWakeP = true
                }
        }
        
        // GC 扫描开始工作，尝试 GCWorker 的 G
        if gp == nil && gcBlackenEnabled != 0 {
                gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
                tryWakeP = tryWakeP || gp != nil
        }
        // 定期直接从 全局可运行队列 获取 G，防止饥饿
        if gp == nil {
                // Check the global runnable queue once in a while to ensure fairness.
                // Otherwise two goroutines can completely occupy the local runqueue
                // by constantly respawning each other.
                if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
                        lock(&sched.lock)
                        gp = globrunqget(_g_.m.p.ptr(), 1)
                        unlock(&sched.lock)
                }
        }
        // 从当前 P 的 可运行队列 获取一个 G
        if gp == nil {
                gp, inheritTime = runqget(_g_.m.p.ptr())
                // We can see gp != nil here even if the M is spinning,
                // if checkTimers added a local goroutine via goready.
        }
        // 上面都尝试了，尽可能去找到一个 G，这里会阻塞
        if gp == nil {
                gp, inheritTime = findrunnable() // blocks until work is available
        }

        // This thread is going to run a goroutine and is not spinning anymore,
        // so if it was marked as spinning we need to reset it now and potentially
        // start a new spinning M.
        if _g_.m.spinning {
                resetspinning()
        }

        // 帮忙唤醒别的 P
        // If about to schedule a not-normal goroutine (a GCworker or tracereader),
        // wake a P if there is one.
        if tryWakeP {
                wakep()
        }
        
        // 如果选出的 G 是被别的 M 锁定的，那么只能重新走流程
        if gp.lockedm != 0 {
                // Hands off own p to the locked m,
                // then blocks waiting for a new p.
                startlockedm(gp)
                goto top
        }

        // 执行新的 G
        execute(gp, inheritTime)
}
```
1. M 有锁定的 G，**执行锁定 G 代码**；
1. 从各个地方得到一个 runnable G，包括：
    1. **GCWork 的 G**，见：
    1. 定期直接从 **全局可运行的队列 获取 G**，防止全局队列的 G 长时间饥饿；
    1. 从绑定的 **P 的 可运行队列 获取 G**；
    1. 通过 `fundrunnable()` **尽可能获取一个 G**；
1. 最终，获取到一个 G 后，`execute()` 执行 G 的代码；

#### 3.1.1 fundrunnable()
为了找到可以运行的 G，findrunnable() 会尝试各个手段。但是这个代码比较复杂，这里捡最关键的点看（runtime/proc.go）：
```go
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
        _g_ := getg()

top:
        // 正在垃圾回收 STW，休眠轮询
        _p_ := _g_.m.p.ptr()
        if sched.gcwaiting != 0 {
                gcstopm()
                goto top
        }

        now, pollUntil, _ := checkTimers(_p_, 0)

        if fingwait && fingwake {
                if gp := wakefing(); gp != nil {
                        ready(gp, 0, true)
                }
        }

        // 从绑定的 P 队列获取
        if gp, inheritTime := runqget(_p_); gp != nil {
                return gp, inheritTime
        }

        // 从全局队列获取
        if sched.runqsize != 0 {
                lock(&sched.lock)
                gp := globrunqget(_p_, 0)
                unlock(&sched.lock)
                if gp != nil {
                        return gp, false
                }
        }

        // 检查全局的 netpoll 的 G
        if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
                if list := netpoll(0); !list.empty() { // non-blocking
                        gp := list.pop()
                        injectglist(&list)
                        casgstatus(gp, _Gwaiting, _Grunnable)
                        if trace.enabled {
                                traceGoUnpark(gp, 0)
                        }
                        return gp, false
                }
        }

        // 从别的 P 偷取一半的任务
        // Steal work from other P's.
        procs := uint32(gomaxprocs)
        ranTimer := false
        for i := 0; i < 4; i++ {
                for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
                        if sched.gcwaiting != 0 {
                                goto top
                        }
                        stealRunNextG := i > 2 // first look for ready queues with more than 1 g
                        p2 := allp[enum.position()]
                        if _p_ == p2 {
                                continue
                        }
                        if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
                                return gp, false
                        }

	        // Consider stealing timers from p2.
	        // This call to checkTimers is the only place where
	        // we hold a lock on a different P's timers.
	        // Lock contention can be a problem here, so
	        // initially avoid grabbing the lock if p2 is running
	        // and is not marked for preemption. If p2 is running
	        // and not being preempted we assume it will handle its
	        // own timers.
	        // If we're still looking for work after checking all
	        // the P's, then go ahead and steal from an active P.
	        if i > 2 || (i > 1 && shouldStealTimers(p2)) {
	                tnow, w, ran := checkTimers(p2, now)
	                now = tnow
	                if w != 0 && (pollUntil == 0 || w < pollUntil) {
	                        pollUntil = w
	                }
	                if ran {
	                        // Running the timers may have
	                        // made an arbitrary number of G's
	                        // ready and added them to this P's
	                        // local run queue. That invalidates
	                        // the assumption of runqsteal
	                        // that is always has room to add
	                        // stolen G's. So check now if there
	                        // is a local G to run.
	                        if gp, inheritTime := runqget(_p_); gp != nil {
	                                return gp, inheritTime
	                        }
	                        ranTimer = true
	                }
	        }
        }
        if ranTimer {
                // Running a timer may have made some goroutine ready.
                goto top
        }

stop:

        // …
        // 依旧按照顺序尝试各个地方获取
        
        // 一无所获，进入休眠
        stopm()
        goto top
}
```
1. 从绑定的 **P 本地队列获取**；
1. 从**全局队列获取**；
1. **检查 netpoll 是否有就绪的 G**（会执行一次非阻塞的 epollwait()）；
1. 通过 `runtime.runqsteal()` 从**其他的 P 的运行中偷取 G**，还可能同时把 P 的 timer 获取了；

### 3.2 execute()
`execute()` 让当前线程执行 G 的代码，并且不会返回：
```go
// Schedules gp to run on the current M.
// If inheritTime is true, gp inherits the remaining time in the
// current time slice. Otherwise, it starts a new time slice.
// Never returns.
func execute(gp *g, inheritTime bool) {
        _g_ := getg()

        // Assign gp.m before entering _Grunning so running Gs have an
        // M.
        _g_.m.curg = gp
        gp.m = _g_.m
        casgstatus(gp, _Grunnable, _Grunning)
        gp.waitsince = 0
        gp.preempt = false
        gp.stackguard0 = gp.stack.lo + _StackGuard
        if !inheritTime {
                _g_.m.p.ptr().schedtick++
        }
        
        // …

        gogo(&gp.sched)
}
```

不过 `execute()` 执行的是一些初始化的操作，切换 PC、SP 等操作只能通过汇编的 `gogo` 实现，注意关键的 `g.sched` 结构（`gobuf` 结构）的传参。
```go
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
        MOVQ        buf+0(FP), BX                // gobuf 内容放到 BX 寄存器
        MOVQ        gobuf_g(BX), DX
        MOVQ        0(DX), CX                // make sure g != nil
        get_tls(CX)
        MOVQ        DX, g(CX)
        MOVQ        gobuf_sp(BX), SP        // 将 SP 地址设置为 gobuf.sp。第一次执行 G 时，这里是 goexit 函数地址
        MOVQ        gobuf_ret(BX), AX
        MOVQ        gobuf_ctxt(BX), DX
        MOVQ        gobuf_bp(BX), BP
        MOVQ        $0, gobuf_sp(BX)        // clear to help garbage collector
        MOVQ        $0, gobuf_ret(BX)
        MOVQ        $0, gobuf_ctxt(BX)
        MOVQ        $0, gobuf_bp(BX)
        MOVQ        gobuf_pc(BX), BX  // 将 BX 寄存器设置为 gobuf.pc。第一次执行 G 时，这里是 G 的函数地址
        JMP        BX  // JMP gobuf.pc，开始执行
```
这里最关键的就是切换为 G 的执行上下文：
* 将 **SP 设置为 `gobuf.sp`**。如果 G 没有执行过，那么值就是创建 g 结构时填入的 `goexit()` 函数的地址。
* 代码流通过 **JMP 指令跳转到 `gobuf.pc`**。如果 G 没有执行过，那么就是 G 对应代码的地址。

这里我们也可以知道了，因为函数调用就是将 返回函数、参数 压栈的过程。而这里将栈顶设置为 `goexit()` 函数，**所以当 G 对应用户代码执行完后，就会继续执行 `goexit()` 函数**。这就是 M 不断执行调度循环的关键。
{{< find_img "img5.png" >}}

### 3.3 goexit()
`goexit()` 在 G 用户代码执行后，执行的汇编代码，其最后通过切换到 g0 栈调用 `goexit0()` 函数：
```go
// goexit continuation on g0.
func goexit0(gp *g) {
        _g_ := getg()
        
        // G running 变为 dead
        casgstatus(gp, _Grunning, _Gdead)
        
        // 重置 g 的属性
        if isSystemGoroutine(gp, false) {
                atomic.Xadd(&sched.ngsys, -1)
        }
        gp.m = nil
        locked := gp.lockedm != 0
        gp.lockedm = 0
        _g_.m.lockedg = 0
        gp.preemptStop = false
        gp.paniconfault = false
        gp._defer = nil // should be true already but just in case.
        gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
        gp.writebuf = nil
        gp.waitreason = 0
        gp.param = nil
        gp.labels = nil
        gp.timer = nil

        if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
                // Flush assist credit to the global pool. This gives
                // better information to pacing if the application is
                // rapidly creating an exiting goroutines.
                scanCredit := int64(gcController.assistWorkPerByte * float64(gp.gcAssistBytes))
                atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
                gp.gcAssistBytes = 0
        }

        // 解除 M 与 G 的绑定
        dropg()

        // 将 dead G 放到 p.gFree 或者 sched.gFree
        gfput(_g_.m.p.ptr(), gp)
        if locked {
                // 如果 M 与 G 是锁定的，那么 M 线程退出
                // The goroutine may have locked this thread because
                // it put it in an unusual kernel state. Kill it
                // rather than returning it to the thread pool.

                // Return to mstart, which will release the P and exit
                // the thread.
                if GOOS != "plan9" { // See golang.org/issue/22227.
                        gogo(&_g_.m.g0.sched)
                } else {
                        // Clear lockedExt on plan9 since we may end up re-using
                        // this thread.
                        _g_.m.lockedExt = 0
                }
        }
        
        // 重新进入调度循环
        schedule()
}
```
1. G 状态由 _Grunning 变为 _Gdead；
1. 重置 G 的属性；
1. 通过 `dropg()` **解除 M 与 G 的绑定**；
1. 将 g 对象放到 `p.gFree` 或者 `sched.gFree`，以便后续创建 G 时可以复用对象；
1. **如果 M 与 G 是锁定着的，而 G 执行完毕，让 m 回到 `mstart()` 函数继续执行，这样 M 线程会被销毁并退出**；
1. **重新进入 `schedule()` 调度循环**；

## 4 调度切换
当前，调度循环中描述的情况是一个 M 执行 G 不被抢占与调度的情况。大多数情况下，当 M 执行 G.fn 的过程中就会被切换，执行其他的 G 的情况。

### 4.1 切换时机
先大体总结一下可能的切换时机：
* **主动挂起**
    * 遇到 runtime 级别阻塞（例如 channel 读写阻塞）
* **主动调度**
    * 调用 `runtime.Gosched()` 主动进行调度
* **系统调用**
    * 系统调用结束后，M 可能进行 G 的切换；
* **抢占**
    * sysmon 判断 G 运行时间大于 10ms，进行抢占；

### 4.2 主动挂起
在 G 遇到非系统调用的阻塞前，就会调用 `gopark()`，**将 G 由 _Grunning -> _Gwaiting，而 M 解绑 G，重新进入调度循环**：
```go
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
        if reason != waitReasonSleep {
                checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
        }
        mp := acquirem()
        gp := mp.curg
        status := readgstatus(gp)
        mp.waitlock = lock
        mp.waitunlockf = unlockf
        gp.waitreason = reason
        mp.waittraceev = traceEv
        mp.waittraceskip = traceskip
        releasem(mp)
        // can't do anything that might move the G between Ms here.
        mcall(park_m)
}

// park continuation on g0.
func park_m(gp *g) {
        _g_ := getg()

        // 改变 G 状态
        casgstatus(gp, _Grunning, _Gwaiting)
        
        // M 解绑 G
        dropg()

        if fn := _g_.m.waitunlockf; fn != nil {
                ok := fn(gp, _g_.m.waitlock)
                _g_.m.waitunlockf = nil
                _g_.m.waitlock = nil
                if !ok {
                        if trace.enabled {
                                traceGoUnpark(gp, 2)
                        }
                        casgstatus(gp, _Gwaiting, _Grunnable)
                        execute(gp, true) // Schedule it back, never returns.
                }
        }
        
        // 重新进入调度循环
        schedule()
}
```
{{< admonition note Note>}}
上面没有任何地方记录 _Gwaiting 状态的 G，Why？

因为这时 `gopark()` 调用者的责任，例如 channel 读写阻塞时，会将 g 记录到 channel 时，在唤醒时将 G 重新加入到可运行队列。
{{< /admonition >}}

当进入 _Gwaiting 状态的 G 需要恢复时，调用 `goready()` / `goparkunlock()` 函数进行恢复：
```go
func goready(gp *g, traceskip int) {
        systemstack(func() {
                ready(gp, traceskip, true)
        })
}

// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
        status := readgstatus(gp)

        // 获取一个 M
        // Mark runnable.
        _g_ := getg()
        mp := acquirem() // disable preemption because it can be holding p in a local var

        // waiting -> runnable
        // status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
        casgstatus(gp, _Gwaiting, _Grunnable)
        
        // 放置到 M 对应 P 的 runq，等待调度执行
        runqput(_g_.m.p.ptr(), gp, next)
        wakep()
        releasem(mp)
}
```

调用 `gopark()` 的地方有许多，列出主要的几个地方：
* **channel 的发送/接受阻塞**；
* **select 所有 case 不满足，陷入阻塞**；
* **time.Sleep 使得 goroutine 进入阻塞**；
* **GC 工作的 gcwork 挂起等待唤醒**；
* **main goroutine 挂起并且不会被唤醒**；

### 4.3 主动调度
标准库接口 `runtime.Gosched()` 可以在用户代码中使 G 主动让出 M：
```go
func Gosched() {
        checkTimeouts()
        mcall(gosched_m)
}

// Gosched continuation on g0.
func gosched_m(gp *g) {
        goschedImpl(gp)
}

func goschedImpl(gp *g) {
        // running 状态变为 runnable
        casgstatus(gp, _Grunning, _Grunnable)
        
        // M 与 G 解绑
        dropg()
        
        // 放到 global runq
        lock(&sched.lock)
        globrunqput(gp)
        unlock(&sched.lock)

        schedule()
}
```

最后会调用 `goschedImpl()`，将 G 由 _Grunning -> _Grunnable 状态，M 与 G 解绑，并将其放到全局运行队列。
{{< admonition note Note>}}
因为调用 `goschedImpl()` 是要切换正在运行的 G，所以放到全局队列。
{{< /admonition >}}

### 4.4 系统调用
Go 通过 `syscall.Syscall` 和 `syscall.RawSyscall` 来封装系统的所有系统调用。
* `syscall.Syscall` **针对可能长时间阻塞的系统调用**，例如 IO 操作。<br>
使得在陷入系统调用之间，系统调用结束后，可以触发一些准备和情况工作。
* `syscall.RawSyscall` **针对不太会长时间阻塞的系统调用**，例如 读取时间等。<br>
直接进行系统调用，不做其他处理。

{{< admonition tip 系统调用分类>}}
大神将各个系统调用分类了，见 [**这里**](https://gist.github.com/draveness/50c88883f30fa99d548cf1163c98aeb1)
{{< /admonition >}}

```go
TEXT ·Syscall(SB),NOSPLIT,$0-56
        CALL        runtime·entersyscall(SB) // 调用 entersyscall()
        MOVQ        a1+8(FP), DI
        MOVQ        a2+16(FP), SI
        MOVQ        a3+24(FP), DX
        MOVQ        trap+0(FP), AX        // syscall entry
        SYSCALL
        CMPQ        AX, $0xfffffffffffff001
        JLS        ok
        MOVQ        $-1, r1+32(FP)
        MOVQ        $0, r2+40(FP)
        NEGQ        AX
        MOVQ        AX, err+48(FP)
        CALL        runtime·exitsyscall(SB)  // 结束调用 exitsyscall()
        RET
ok:
        MOVQ        AX, r1+32(FP)
        MOVQ        DX, r2+40(FP)
        MOVQ        $0, err+48(FP)
        CALL        runtime·exitsyscall(SB) // 结束调用 exitsyscall()
        RET
```
1. 系统调用前，执行 `entersyscall()`；
1. 执行系统调用；
1. 系统调用后，执行 `exitsyscall()`；

#### 4.4.1 系统调用前的准备
`entersyscall()` 会调用 `reentersyscall()` 函数，执行进入系统调用前的准备工作：
```go
func reentersyscall(pc, sp uintptr) {
        _g_ := getg()
        _g_.m.locks++
        _g_.stackguard0 = stackPreempt
        _g_.throwsplit = true

        save(pc, sp)
        _g_.syscallsp = sp
        _g_.syscallpc = pc
        casgstatus(_g_, _Grunning, _Gsyscall)

        _g_.m.syscalltick = _g_.m.p.ptr().syscalltick
        _g_.m.mcache = nil
        pp := _g_.m.p.ptr()
        pp.m = 0
        _g_.m.oldp.set(pp)
        _g_.m.p = 0
        atomic.Store(&pp.status, _Psyscall)
        if sched.gcwaiting != 0 {
                systemstack(entersyscall_gcwait)
                save(pc, sp)
        }
        _g_.m.locks--
}
```
1. 禁止线程上发生的抢占，防止出现内存不一致的问题；
1. 保证当前函数不会触发栈分裂或者增长；
1. 通过 `save()` 保存 PC、SP 值至 g.sched；
1. G 状态 _Grunning -> _Gsyscall；
1. `m.oldp` 设置为当前 P，`m.p` 设置为 0，这意味着记录但是解绑当前的 P，而 P 状态为 _Psyscall；

这里比较重要的就是让 **M 与 P 解绑，使得其他 M 可以获取到该 P 并执行 G**。

因此，P 代表的是并发数，而不是线程数。

#### 4.4.2 系统调用后的恢复
系统调用结束后，执行 `exitsyscall()` 进行恢复操作。
```go
func exitsyscall() {
        _g_ := getg()

        // M 尝试绑定阻塞前使用的 P，或者一个新的 P
        oldp := _g_.m.oldp.ptr()
        _g_.m.oldp = 0
        if exitsyscallfast(oldp) {
                _g_.m.p.ptr().syscalltick++
                casgstatus(_g_, _Gsyscall, _Grunning)
                ...

        return
        }

        // M 解绑 G，重新进入调用循环
        mcall(exitsyscall0)
        _g_.m.p.ptr().syscalltick++
        _g_.throwsplit = false
}
```
1. **M 尝试获取一个空闲的 P**，从两个地方：
    * 如果 `m.oldp` 还是为 _Psyscall 状态，说明没被人使用，那么记录使用阻塞前的 P；
    * 从全局空闲 P `sched.pidle` 链表中获取一个 P；
1. 如果没有 P，那么确实 M 无法执行当前 G，就 **M 解绑 G，将 G 放入全局队列**；
1. 无论如何，**最后 M 都会调用 `schedule()` 重启进入调度循环**，切换一个 G 执行。

### 4.5 抢占
每个运行中的 G 会有一个运行的时间片，而 sysmon 会**周期性检查部分 G，如果其执行时间大于 10ms，就会触发抢占**。

抢占包括两种方式：
* **`抢占标志`**：通过设置 `g.stackguard0=stackPreempt`。这必然会导致 G 执行下次函数调用时触发栈扩容逻辑，从而走到切换调度的逻辑；
* **`信号抢占`**（v1.14）：信号抢占会使得 M 线程触发信号中断，执行信号处理函数，从而重新进入调度循环。

#### 4.5.1 触发抢占时机
sysmon 会重启执行 `retake()` 函数，判断哪些正在运行的 G 是需要抢占的。
```go
func retake(now int64) uint32 {
        n := 0
        // Prevent allp slice changes. This lock will be completely
        // uncontended unless we're already stopping the world.
        lock(&allpLock)
        
        // 遍历所有 P
        for i := 0; i < len(allp); i++ {
                _p_ := allp[i]
                pd := &_p_.sysmontick
                s := _p_.status
                sysretake := false
                
                // 允许抢占 _Prunning 与 _Psyscall 状态的 P
                if s == _Prunning || s == _Psyscall {
                        t := int64(_p_.schedtick)
                        if int64(pd.schedtick) != t {
                                pd.schedtick = uint32(t)
                                pd.schedwhen = now
                        } else if pd.schedwhen+forcePreemptNS <= now {
                                // 到达抢占时间
                                preemptone(_p_)
                                // In case of syscall, preemptone() doesn't
                                // work, because there is no M wired to P.
                                sysretake = true
                        }
                }
                
                // 对于 _Psyscall，可以让其与 M 解绑，等其他的 M 绑定
                if s == _Psyscall {
                        t := int64(_p_.syscalltick)
                        if !sysretake && int64(pd.syscalltick) != t {
                                pd.syscalltick = uint32(t)
                                pd.syscallwhen = now
                                continue
                        }
                        if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
                                continue
                        }
                        unlock(&allpLock)
                        incidlelocked(-1)
                        if atomic.Cas(&_p_.status, s, _Pidle) {
                                n++
                                _p_.syscalltick++
                                handoffp(_p_)
                        }
                        incidlelocked(1)
                        lock(&allpLock)
                }
        }
        unlock(&allpLock)
        return uint32(n)
}
```
1. 运行中的 G 运行超过 10ms，调用 `preemptone()` 进行抢占；
1. 对于 _Psyscall 中的 P，将其解绑 M，使得其他 M 可以绑定该 P；

#### 4.5.2 触发抢占
`preemptone()` 触发抢占，通过上面所述的两种方式。
```go
func preemptone(_p_ *p) bool {
        mp := _p_.m.ptr()
        gp := mp.curg
        gp.preempt = true

        // 设置抢占标志
        // Every call in a go routine checks for stack overflow by
        // comparing the current stack pointer to gp->stackguard0.
        // Setting gp->stackguard0 to StackPreempt folds
        // preemption into the normal stack overflow check.
        gp.stackguard0 = stackPreempt

        // 信号抢占
        // Request an async preemption of this P.
        if preemptMSupported && debug.asyncpreemptoff == 0 {
                _p_.preempt = true
                preemptM(mp)
        }

        return true
}
```
1. **设置 G 的抢占标志，gp.stackguard0 = stackPreempt**；
1. **执行 `preemptM()` 进行信号抢占**；

#### 4.5.3 通过抢占标志抢占
前面看到，设置 `gp.stackguard0 = stackPreempt`，而这在每次 G 函数调用前的检查是否扩容栈时，**必然会触发 G 栈扩容逻辑** [**newstack()**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/#33-%E6%A0%88%E7%9A%84%E6%89%A9%E5%AE%B9)。

而在 [**内存管理**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/#top) 时，介绍了 `newstack()` 如果扩容 G 的栈，但是省略了一个重要的逻辑分支：**`newstack()` 函数中还会进行 G 的调度**：
```go
func newstack() {
        thisg := getg()
        gp := thisg.m.curg
        // …
        preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt
        // 特殊情况下不能抢占时，继续走 G 代码逻辑
        if preempt {
                if !canPreemptM(thisg.m) {
                        // Let the goroutine keep running for now.
                        // gp->preempt is set, so it will be preempted next time.
                        gp.stackguard0 = gp.stack.lo + _StackGuard
                        gogo(&gp.sched) // never return
                }
        
        // …
        if preempt {
                if gp == thisg.m.g0 {
                        throw("runtime: preempt g0")
                }
                if thisg.m.p == 0 && thisg.m.locks == 0 {
                        throw("runtime: g is running but p is not")
                }

        if gp.preemptShrink {
                // We're at a synchronous safe point now, so
                // do the pending stack shrink.
                gp.preemptShrink = false
                shrinkstack(gp)
        }

        if gp.preemptStop {
                preemptPark(gp) // never returns
        }

        // Act like goroutine called runtime.Gosched.
        gopreempt_m(gp) // never return
        }
        // …
}

func gopreempt_m(gp *g) {
        // 老朋友了
        goschedImpl(gp)
}
```
判断到 `gp.stackguard0 = stackPreempt` 后，无论如何都会走重新调度的逻辑，M 重新进入调度循环。

#### 4.5.4 信号抢占
执行 `preemptM()` 会对 M 进行信号抢占，**通过发送 SIGURG 信号触发 M 线程的信号处理函数**。
```go
const sigPreempt = _SIGURG

func preemptM(mp *m) {
        if GOOS == "darwin" && GOARCH == "arm64" && !iscgo {
                return
        }

        if atomic.Cas(&mp.signalPending, 0, 1) {
                if GOOS == "darwin" {
                        atomic.Xadd(&pendingPreemptSignals, 1)
                }

        // If multiple threads are preempting the same M, it may send many
        // signals to the same M such that it hardly make progress, causing
        // live-lock problem. Apparently this could happen on darwin. See
        // issue #37741.
        // Only send a signal if there isn't already one pending.
        signalM(mp, sigPreempt)
        }
}
```

M 会执行信号处理函数 `asyncPreempt()`，最后调用到 `asyncPreempt2()`，使得 M 进入重新调度：
```go
// asyncPreempt saves all user registers and calls asyncPreempt2.
//
// When stack scanning encounters an asyncPreempt frame, it scans that
// frame and its parent frame conservatively.
//
// asyncPreempt is implemented in assembly.
func asyncPreempt()

//go:nosplit
func asyncPreempt2() {
	gp := getg()
	gp.asyncSafePoint = true
	if gp.preemptStop {
		mcall(preemptPark)
	} else {
		mcall(gopreempt_m)
	}
	gp.asyncSafePoint = false
}
```
无论是走 `preemptPark()` 还是 `gopreempt_m()`，最后都会使得 **M 解绑当前 G，执行 `schedule()`，开始调度循环**。

{{< admonition tip 处理信号的时机>}}
**信号处理不是及时的**，内核会把信号记录在进程（线程）的 pending 队列上，然后触发软中断将进程变为 running 状态。

当进程被唤醒或者被内核调度得到 CPU 后，从内核态返回用户态时，会检查 pending 队列的信号，并执行信号处理函数。
{{< /admonition >}}

## 总结
相对于 [**内存管理**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/) 与 [**垃圾收集**](https://kanshiori.github.io/posts/language/golang/go-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E6%80%BB%E7%BB%93/)，竟然感觉并发调度的结构还算简单。

需要弄清楚的有以下几点：
* [**协程出现的意义**](#11-进程线程与协程)
* [**调度器的工作**](#12-调度器)
* [**GMP 模型整体框架**](#2-gmp-模型)
* [**G、M、P 代表的意义**](#2-gmp-模型)
* 调度循环的流程
    * [**进入调度循环 M 的行为**](#31-schedule)
    * [**找寻 G 的各个途径**](#311-fundrunnable)
    * [**M 执行 G 时，如何切换上下文**](#32-execute)
    * [**如何退出回到调度循环**](#32-execute)
* 调度切换
    * [**切换的各个时机**](#41-切换时机)
    * [**系统调用进行调度切换的行为**](#44-系统调用)
    * [**抢占的两种方式**](#45-抢占)

## 参考
* [《Golang 学习笔记》](https://github.com/qyuhen/book)
* [《Golang 设计与实现》：调度器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)
