# Go 内存管理总结


本文是对 Golang 内存模型与内存管理的一个总结，基本内容来源于网络的学习，以及自己观摩了下源码。

所以学习的书籍与文章见 [**参考**](#reference)。

下面代码都是基于 go 1.15.6。

## 1 Linux 内存模型
所有语言的内存管理，在 Linux 上都是在以基本的进程内存模型基础上实现的，首先需要知道 Linux 进程内存布局。

在进程角度，看到的所有内存就是 [虚拟地址空间]^(virtual address space) ，整个是一个线性的存储地址。其中一部分高地址区域用户态无法访问，是内核地址空间。而另一部分就是由栈、mmap、堆等内存区域组成的用户地址空间。
{{< find_img "img1.png" >}}
上面进程可以自己分配与管理的进程，就是 mmap 与 堆，对应的系统调用为 **`mmap()`** 与 **`brk()`**，因此所有语言的内存管理都是基于这两个内存区域在进一步实现的（包括 glibc 的 malloc() 与 free()）。

mmap 最基本有两个用途：
* `文件映射` ：申请一块内存区域，映射到文件系统上一个文件（这也是 page_cache 的基本原理，所以他们在内核中都使用 address_space 实现）
* `匿名映射` ：申请一块内存区域，但是没有映射到具体文件，相当于分配了一块内存区域（可以用于父子进程共享、或者自己管理内存的分配等功能）

而所有在内存上所说的地址，包括代码指令地址、变量地址都是上面地址空间的一个地址。



## 2 PC 与 SP

Goroutine 将进程的切换变为了协程间的切换，那么就需要**在用户空间负责执行代码与协程上下文的保留与切换**。因此，有两个关键的寄存器：PC 与 SP。

### 2.1 PC
[程序计数器 PC]^(Program Counter) 是 CPU 中的一个寄存器，**保存着下一个 CPU 执行的指令的位置**。顺序执行指令时，PC = PC + 1（一个指令）。而调用函数或者条件跳转时，会将跳到的指令地址设置到 PC 中。

所以，可以想到，当需要切换执行的 goroutine，调用 JMP 指令跳转到 G 对应的代码。

### 2.2 SP
[栈顶指针 SP]^(stack pointer) 是**保存栈顶地址的寄存器**，我们平时所说的临时变量在栈上，就是将临时变量的值写入 SP 保存的内存地址，然后 SP 保存的地址减小（栈是从高地址向低地址变化），然后临时变量销毁时，SP 地址又变为高地址。

不过，因为 goroutine 切换时，必须要保存当前 goroutine 的上下文，也就是栈里的变量。因此，goroutine 栈肯定是不能使用 Linux 进程栈了（因为进程栈有上限，也无法实现“保存”这种功能）。所以所说的**协程栈，都是基于 mmap 申请内存空间**（基于 Go 内存管理，内存管理基于 mmap），然后**切换时修改 SP 寄存器地址实现的**。

这也是为什么 goroutine 栈可以“无限大”的原因了。



## 3 Goroutine 栈
整体的一个 G 的栈如下图所示：
{{< find_img "img2.png" >}}
* `stack.lo`： G 栈的最大低地址（也就是上限）；
* `stack.hi`：G 栈的初始地址；
* `stackguard0`：阈值地址，用于判断 G 栈是否需要扩容；
* StackGuard：常量，栈的保护区，也就是预留的地址；
* StackSmall：常量，用于小函数调用的优化；

先看一下 g 的实现中包含的 stack 属性（runtime/runtime2.go），其实注释写的就很明白了：
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

    // ...
}
```
* `stack` 属性就是 G 对应的栈了（这也表明了不是使用的进程栈）；

{{< admonition note Note>}}
stack 与 stackguard0 属性一定要在 g 结构的开头，因为汇编中会使用指定的偏移(0x10)来获取对应的值;
{{< /admonition >}}

具体看一下 stack 结构（runtime/runtime2.go）：
```go
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
type stack struct {
	lo uintptr
	hi uintptr
}
```

<a id="new-g-stack"></a>

### 3.1 新 G 的栈
在 **`malg`** 函数中，可以看到对于新 G 的栈的分配（一开始为 2KB）：
```go
// Allocate a new g, with a stack big enough for stacksize bytes.
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		
		// 在公共的 goroutine(g0) 上调用函数
		systemstack(func() {
		    // 分配一个 stack
			newg.stack = stackalloc(uint32(stacksize))
		})
		
		// 设置 stackguard0 地址
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
		// Clear the bottom word of the stack. We record g
		// there on gsignal stack during VDSO on ARM and ARM64.
		*(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0
	}
	return newg
}
```

{{< admonition note Note>}}
注意 **`systemstack()`**，用于将当前栈切换到 M 的 g0 协程栈上执行命令。

Why? 因为 G 用于执行用户逻辑，而某些管理操作不方便在 G 栈上执行（例如 G 可能中途停止，垃圾回收时 G 栈空间也有可能被回收），所以需要执行管理命令时，都会通过 systemstack 方法将线程栈切换为 g0 的栈执行，与用户逻辑隔离。
{{< /admonition >}}

### 3.2 栈的分配
`stackalloc()` 函数用于分配一个栈，无论是给新 G 还是扩容栈时都会用到，因此栈空间的分配与回收是一个比较频繁的操作，所以栈空间采取了缓存复用的方式。

主要逻辑如下：
1. 如果分配的栈空间不大，就走**缓存复用**这种方式分配。没有可以复用的就创建；
2. 如果分配的栈空间很大（大于 32KB），就直接从 heap 分配；

这里主要关注第 1 中方式，会调用 `stackpoolalloc()` 函数。
```go
// Allocates a stack from the free pool. Must be called with
// stackpool[order].item.mu held.
func stackpoolalloc(order uint8) gclinkptr {
	list := &stackpool[order].item.span
    s := list.first
    
	// ...
    
    if s == nil {
		// 没有可以复用的栈，走内存管理创建
		s = mheap_.allocManual(_StackCacheSize>>_PageShift, &memstats.stacks_inuse)
		// ... 
	}
	x := s.manualFreeList
	if x.ptr() == nil {
		throw("span has no free stacks")
	}
    s.manualFreeList = x.ptr().next
    
	// ...
	return x
}
```
可以看到，**首先尝试从 stackpool 缓存的空闲的 stack 获取，如果没有则走 Go 内存管理申请一个。**

再接下来就是 Go 内存管理模块负责的事了，~~不深入下去~~（后面再说）。底层创建都是使用 mmap 系统调用实现的，这里可以看下使用的参数：
```go
// Don't split the stack as this method may be invoked without a valid G, which
// prevents us from allocating more stack.
//go:nosplit
func sysAlloc(n uintptr, sysStat *uint64) unsafe.Pointer {
	p, err := mmap(nil, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
	if err != 0 {
		if err == _EACCES {
			print("runtime: mmap: access denied\n")
			exit(2)
		}
		if err == _EAGAIN {
			print("runtime: mmap: too much locked memory (check 'ulimit -l').\n")
			exit(2)
		}
		return nil
	}
	mSysStatInc(sysStat, n)
	return p
}
```
通过 mmap 调用的参数可以看到，申请了一个系统分配的**匿名内存映射**。

### 3.3 栈的扩容

#### 3.3.1 扩容判断
Go 编译器会在执行函数前，插入一些汇编指令，其中一个功能就是**检查 G 栈是否需要扩容**。看一个函数调用的实现：
```go
// main() 调用 test()
$ go build -gcflags "-l" -o test main.go
$ go tool objump -s "main\.test" test
TEXT main.test(SB) /root/yusihao/onething/BizImages/main.go
  main.go:3 0x45dc80 MOVQ FS:0xfffffff8, CX  // CX 为当前 G 地址		
  main.go:3 0x45dc89 CMPQ 0x10(CX), SP  	 // CX+0x10 执行 g.stackguard0 属性，与 SP 指针地址比较	
  main.go:3 0x45dc8d JBE 0x45dccf	 	     // 如果 SP <=stackguard0  跳转到 0x45dccf，也就是调用 runtime.morestack_noctxt(SB) 函数	
  main.go:3 0x45dc8f SUBQ $0x18, SP	

  // ...

  main.go:5 0x45dcce RET                                // 函数执行结束，RET 返回，不会执行后面两个指令				
  main.go:3 0x45dccf CALL runtime.morestack_noctxt(SB)  // 执行栈扩容	
  main.go:3 0x45dcd4 MP main.test(SB)                   // 执行结束后，重新执行当前函数	
```
逻辑很简单，如果 SP <= stackguard0，那么就执行栈的扩容，扩容结束重新执行当前函数。
{{< admonition note Note>}}
上面比较 SP 时候，没有考虑当前函数调用使用的空间大小。Why?

因为测试程序这个函数中使用的空间比较小，而 stackguard0  与 stack.lo 有一段**保护区**，所以编译器允许这里 "溢出" 一些，所以这里就没有让 SP 考虑函数使用空间。

如果函数中使用的空间大过保护区时，比较时就会让 SP 减去当前函数使用空间再比较了。
{{< /admonition >}}

#### 3.3.2 扩容
扩容逻辑大致分为三步：
1. 分配一个 2x 新栈；
1. 拷贝当前栈数据至新栈；
1. "释放"掉旧栈；

从上面扩容判断可以看到，会调用 **`morestack`** 的汇编代码：<a id="morestack"></a>
```go
// Called during function prolog when more stack is needed.
// R3 prolog's LR
// using NOFRAME means do not save LR on stack.
//
// The traceback routines see morestack on a g0 as being
// the top of a stack (for example, morestack calling newstack
// calling the scheduler calling newm calling gc), so we must
// record an argument size. For that purpose, it has no arguments.
TEXT runtime·morestack(SB),NOSPLIT|NOFRAME,$0-0
	// Cannot grow scheduler stack (m->g0).
	MOVW        g_m(g), R8
	MOVW        m_g0(R8), R4
	CMP        g, R4
	BNE        3(PC)
	BL        runtime·badmorestackg0(SB)
	B        runtime·abort(SB)

	// Cannot grow signal stack (m->gsignal).
	MOVW        m_gsignal(R8), R4
	CMP        g, R4
	BNE        3(PC)
	BL        runtime·badmorestackgsignal(SB)
	B        runtime·abort(SB)

	// Called from f.
	// Set g->sched to context in f.
	MOVW        R13, (g_sched+gobuf_sp)(g)
	MOVW        LR, (g_sched+gobuf_pc)(g)
	MOVW        R3, (g_sched+gobuf_lr)(g)
	MOVW        R7, (g_sched+gobuf_ctxt)(g)

	// Called from f.
	// Set m->morebuf to f's caller.
	MOVW        R3, (m_morebuf+gobuf_pc)(R8)        // f's caller's PC
	MOVW        R13, (m_morebuf+gobuf_sp)(R8)        // f's caller's SP
	MOVW        g, (m_morebuf+gobuf_g)(R8)

	// Call newstack on m->g0's stack.
	MOVW        m_g0(R8), R0
	BL        setg<>(SB)
	MOVW        (g_sched+gobuf_sp)(g), R13
	MOVW        $0, R0
	MOVW.W  R0, -4(R13)        // create a call frame on g0 (saved LR)
	BL        runtime·newstack(SB)

	// Not reached, but make sure the return PC from the call to newstack
	// is still in this function, and not the beginning of the next.
	RET

TEXT runtime·morestack_noctxt(SB),NOSPLIT|NOFRAME,$0-0
	MOVW        $0, R7
	B runtime·morestack(SB)
```
* 可以看到 **g0，gsignal 的栈都不会扩容**
* 在 g0 栈上会调用 `newstack()` 函数

调用的 **`newstack()`** 函数（runtime/stack.go），过程很复杂，只看一下关键点：
```go
// Called from runtime·morestack when more stack is needed.
// Allocate larger stack and relocate to new stack.
// Stack growth is multiplicative, for constant amortized cost.
//
// g->atomicstatus will be Grunning or Gscanrunning upon entry.
// If the scheduler is trying to stop this g, then it will set preemptStop.
//
// This must be nowritebarrierrec because it can be called as part of
// stack growth from other nowritebarrierrec functions, but the
// compiler doesn't check this.
//
//go:nowritebarrierrec
func newstack() {
    thisg := getg()
    gp := thisg.m.curg
    
    // ...

	// 新栈大小为当前两倍
	oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize * 2
    
    // ...

	// 改变 G 状态为 copy stack，gc 会跳过该状态的 G
	casgstatus(gp, _Grunning, _Gcopystack)

    // 分配新栈，拷贝数据，释放旧站
	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, newsize)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	// 执行 G 代码
	gogo(&gp.sched)
}

// Copies gp's stack to a new stack of a different size.
// Caller must have changed gp status to Gcopystack.
func copystack(gp *g, newsize uintptr) {
    // 创建新 stack
	new := stackalloc(uint32(newsize))
	// ...
	// 拷贝数据
	memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)
	// ...
	gp.stack = new
	gp.stackguard0 = new.lo + _StackGuard 
	// 释放旧 stack
	if stackPoisonCopy != 0 {
		fillstack(old, 0xfc)
	}
	stackfree(old)
}
```

### 3.4 栈的释放
stackfree 栈的释放与申请相反，放入 stackpool，或者直接调用内存管理删除，重点还是内存管理的活，所以这里不展开。

### 3.5 栈的切换
切换应该是属于 goroutine 调度的内容，不过这里可以关注一下栈时如何切换的。
当 M 执行的 G 需要切换，或者一个新创建 G 执行时，最后都会调用 `execute()` 函数，而 execute() 函数会调用 gogo 汇编实现的函数。
```go
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		    // make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	// restore SP (关键!)
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX
	JMP	BX
```
`gobuf` 中保存着要执行的 G 的 sp、pc 指针，可以看到通过**将对应 gobuf.sp 写入到 SP 寄存器中**，也就是将使用的栈切换为了 G 的栈。

### 3.6 g0 的栈
在阅读网上的文章时，许多文章都说 g0 使用的是系统栈，我理解为使用的是进程的栈内存区域。但是思考一下，每个 M 对应一个 g0，也就是说有多个线程要同时共享系统栈，这是不可能的。例如在 pthread 实现中，对应新建的线程也是使用 mmap 分配一个内存区域，然后调用 clone() 系统调用时传入栈地址参数。

看一下代码，确认一下到底 g0 的栈到底是啥，找到一个新建 m 的地方：
```go
mp := new(m)
mp.mstartfn = fn
mcommoninit(mp, id)

// In case of cgo or Solaris or illumos or Darwin, pthread_create will make us a stack.
// Windows and Plan 9 will layout sched stack on OS stack.
if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
        mp.g0 = malg(-1)
} else {
        mp.g0 = malg(8192 * sys.StackGuardMultiplier)
}
mp.g0.m = mp
```
可以看到，m 的 g0 属性还是使用的 [**malg() 函数**](#new-g-stack) 去创建的，与普通的 g 创建一样，只不过初始大小为 8KB。malg() 流程上面有说到，就是走内存管理分配 mspan 作为栈的方式。

不过，g0 的栈还是有些不同的，不会进行栈的扩容（因为仅仅内部管理时用到，不需要进行自动扩容），在栈扩容的 [**morestack 汇编代码**](#morestack)里可以看到。


## 4 内存模型

### 4.1 概览
Golang 内存管理包含四个组件：
1. **`object`** ：object 代表用户代码申请的一个对象，没有实际的数据结构，而是在 mspan 中以逻辑切分的方式分配；
1. page：切分内存的单元，mheap 将内存以 8KB page 切分，然后组合成为 mspan；
1. **`runtime.mspan`** ：内存管理的最小单元，由多个 8KB 大小的 page 构成，按照固定大小来切分为多个 object；
1. **`runtime.mcache`** ：单个 P 对应的 mspan 的**缓存**，无锁分配；
1. **`runtime.mcentral`** ：按照不同大小的 mspan 分组的管理链表，为 mcache 提供空闲 mspan
1. **`runtime.mheap`** ：保存闲置的 mspan 与 largerspan 链表，与操作系统申请与释放内存；

上面的组件也可以看做分层，普通对象（object）的申请与释放就是按照上下层顺序申请与释放的。
{{< admonition note Note>}}
下面~~不会说~~（后面再说）具体的 object 分配流程，而是说明各个层次时的申请与释放操作。
{{< /admonition >}}

### 4.2 mspan
每个 mspan 由多个 8KB 的 page 组成，所有的 mspan 会以 list 的方式构建，而不同的模块（mcache、mcentral）通过引用指针，来不同方式来组织不同的 mspan。

每个 mspan 管理多个固定大小的 object，通过编号(index)方式来寻找 object 的地址。

结构如下图所示：
{{< find_img "img3.png" >}}

其数据结构如下（省略部分）：
```go
type mspan struct {
        next *mspan     // next span in list, or nil if none
        prev *mspan     // previous span in list, or nil if none

        startAddr uintptr // address of first byte of span aka s.base()
        npages    uintptr // number of pages in span

        manualFreeList gclinkptr // list of free objects in mSpanManual spans
        freeindex uintptr
        nelems uintptr // number of object in the span.
        
        allocCache uint64
        allocBits  *gcBits
        gcmarkBits *gcBits
        
		// sweep generation:
		// if sweepgen == h->sweepgen - 2, the span needs sweeping
		// if sweepgen == h->sweepgen - 1, the span is currently being swept
		// if sweepgen == h->sweepgen, the span is swept and ready to use
		// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
		// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
		// h->sweepgen is incremented by 2 after every GC
        sweepgen    uint32
        spanclass   spanClass     // size class and noscan (uint8)
        allocCount  uint16        // number of allocated objects
        elemsize    uintptr       // computed from sizeclass or from npages
}
```
* next、prev ：链表前后 span；
* **`startAddr`** ：span 在 arena 区域的起始地址;
* **`npages`** ：占用 page(8KB) 数量；
* manualFreeList ：空闲 object 链表；
* **`freeindex`** ：下一个空闲的 object 的编号，如果 freeindex == nelem，表明没有空闲 object 可以分配
* nelems ：当前 span 中分配的 object 的上限；
* **`allocCache`** ：freeindex 的 cache，通过 bitmap 的方式记录对应编号的 object 内存是否是空闲的；
* **`sweepgen`** ：mspan 的状态, 见注释；
* spanclass ：mspan 大小类别；
* allocCount ：已经分配的 object 数量；
* **`elemsize`** ：管理的 object 的固定大小；

可以看到，每个 mspan 管理着固定大小的 object，并通过一个 freeindex+allocCache 来记录空闲的 object 的编号。由此可以得出:
* **mspan 的地址区域**: **`[startAddr, startAddr + npages*8*1024)`**
* **某个 object 的起始地址**: **`<index>*elemsize + startAddr`**

#### 4.2.1 object 分配
在创建新的 object 时，对于普通大小的 object 分配（16<size<32KB)，会在从 mcache 中选出具有空闲空间的 mspan，然后记录到 mspan.allocCache 中。

具体代码如下，`nextFreeIndex()` 函数就是用于得到下一个空闲 object，并移动 `freeindex`（runtime/mbitmap.go)：
```go
// nextFreeIndex returns the index of the next free object in s at
// or after s.freeindex.
// There are hardware instructions that can be used to make this
// faster if profiling warrants it.
func (s *mspan) nextFreeIndex() uintptr {
        sfreeindex := s.freeindex
        snelems := s.nelems
        if sfreeindex == snelems {
                return sfreeindex
        }

        aCache := s.allocCache

        bitIndex := sys.Ctz64(aCache)
        for bitIndex == 64 {
                // Move index to start of next cached bits.
                sfreeindex = (sfreeindex + 64) &^ (64 - 1)
                if sfreeindex >= snelems {
                        s.freeindex = snelems
                        return snelems
                }
                whichByte := sfreeindex / 8
                // Refill s.allocCache with the next 64 alloc bits.
                s.refillAllocCache(whichByte)
                aCache = s.allocCache
                bitIndex = sys.Ctz64(aCache)
                // nothing available in cached bits
                // grab the next 8 bytes and try again.
        }
        result := sfreeindex + uintptr(bitIndex)
        if result >= snelems {
                s.freeindex = snelems
                return snelems
        }

        s.allocCache >>= uint(bitIndex + 1)
        sfreeindex = result + 1

        if sfreeindex%64 == 0 && sfreeindex != snelems {
                // We just incremented s.freeindex so it isn't 0.
                // As each 1 in s.allocCache was encountered and used for allocation
                // it was shifted away. At this point s.allocCache contains all 0s.
                // Refill s.allocCache so that it corresponds
                // to the bits at s.allocBits starting at s.freeindex.
                whichByte := sfreeindex / 8
                s.refillAllocCache(whichByte)
        }
        s.freeindex = sfreeindex
        return result
}
```
注意：目前跳过了 "nextFreeFast" 实现，该获取 span 比 "nextFree" 更快，使用了 `mspan.allocCache`。
#### 4.2.2 mspan 的清理
TODO

### 4.3 mcache
每个 P 拥有一个 mcache，mcache 中保存着具有空闲空间的 mspan，用于分配 object 时，不需要加锁即可从 mspan 分配对象。

有两种 object 走 mcache 分配：
* **`tiny object`**：mcache 还单独使用一个 mspan 进行**非指针微小对象**的分配。与普通 object 对象分配不同的是，tiny object 不是固定大小分配的，而是通过 mcache 记录其 offset 偏移量，让 tiny object "挤在" 同一个 mspan 中。
* **`normal object`**：普通大小的 object，会使用 `mcache.alloc` 进行分配。`mcache.alloc` 包含 134 个数组项（67 sizeclass * 2），对于每个大小规格的 mspan 有着两个类型：
	* **scan**：包含指针的对象
	* **noscan**：不包含指针的对象，GC 时无需进一步扫描是否引用着其他活跃对象

mcache "永远" 有空闲的 mspan 用于 object 的分配，当 mcache 缓存的 mspan 没有空闲空间时，就会**找 mcentral 去申请新的 mspan 用于使用**。
{{< find_img "img4.png" >}}

数据结构如下（runtime/mcache.go）：
```go
// Per-thread (in Go, per-P) cache for small objects.
// No locking needed because it is per-thread (per-P).
//
// mcaches are allocated from non-GC'd memory, so any heap pointers
// must be specially handled.
type mcache struct {
        // Allocator cache for tiny objects w/o pointers.
        // See "Tiny allocator" comment in malloc.go.

        // tiny points to the beginning of the current tiny block, or
        // nil if there is no current tiny block.
        //
        // tiny is a heap pointer. Since mcache is in non-GC'd memory,
        // we handle it by clearing it in releaseAll during mark
        // termination.
        tiny             uintptr
        tinyoffset       uintptr
        local_tinyallocs uintptr // number of tiny allocs not counted in other stats

        // The rest is not accessed on every malloc.

        alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass

        stackcache [_NumStackOrders]stackfreelist
}
```
* **`tiny`** **`tinyoffset`** ：用于小对象（<16）的分配。tiny 指向当前为 tiny object 准备的 span 的起始地址，tinyoffset 指向对象使用的偏移地址；
* **`alloc`** ：最重要的属性，保存着不同大小的 mspan 各一个。目前，包含固定 64 类 sizeclass：0、8 … 32768；

<a id="mcache-alloc_mspan"></a>

#### 4.3.1 mspan 的分配
先看下 tiny object 分配，在分配一个 object 时，如果大小小于 16 字节时，就会走 tiny object 逻辑。
```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = alignUp(off, 8)
			} else if size&3 == 0 {
				off = alignUp(off, 4)
			} else if size&1 == 0 {
				off = alignUp(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			...
		}
	} else {
		...
	}
}
```
1. 如果当前 **tinyoffset+size < 16B**（因为每个 tiny span 的大小为 32B，而每个 tiny object 最大为 16，所以比较 16 即可），那么表明**当前的 tiny span 肯定可以放下**，那么移动 `c.tinyoffset` 偏移即可；
2. 如果没有，那么就需要**重新申请** tinySpanClass=5 的 span（32B），并替换当前 `c.tiny` 与 `c.tinyoffset`（当前 object 可以放在老的 span 或者新的 span）。

接着看下普通大小的 object 的分配，在外层函数计算好 spanClass 后，就会调用 **`nextFree()`** 函数（runtime/malloc.go）：  
```go
// nextFree returns the next free object from the cached span if one is available.
// Otherwise it refills the cache with a span with an available object and
// returns that object along with a flag indicating that this was a heavy
// weight allocation. If it is a heavy weight allocation the caller must
// determine whether a new GC cycle needs to be started or if the GC is active
// whether this goroutine needs to assist the GC.
//
// Must run in a non-preemptible context since otherwise the owner of
// c could change.
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
        s = c.alloc[spc]
        shouldhelpgc = false
        freeIndex := s.nextFreeIndex()
        if freeIndex == s.nelems {
                // The span is full.
                if uintptr(s.allocCount) != s.nelems {
                        println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
                        throw("s.allocCount != s.nelems && freeIndex == s.nelems")
                }
                c.refill(spc)
                shouldhelpgc = true
                s = c.alloc[spc]

        freeIndex = s.nextFreeIndex()
        }

        if freeIndex >= s.nelems {
                throw("freeIndex is not valid")
        }

        v = gclinkptr(freeIndex*s.elemsize + s.base())
        s.allocCount++
        if uintptr(s.allocCount) > s.nelems {
                println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
                throw("s.allocCount > s.nelems")
        }
        return
}
```
1. 得到 `c.alloc` 对应大小的 mspan；
1. 如果 mspan 没有空闲空间了（freeIndex == s.nelems），那么走 `c.refill()` **重新向 mcentral 申请一个 有空闲空间 mspan**；
3. `mspan.nextFreeIndex()` 下一个 index，并计算出对应的内存地址返回；


<a id="mcache-alloc"></a>

#### 4.3.2 mspan 的获取
前面分配 object 中可以看到，当 mcache 当前大小的 mspan 没有空闲空间后，就会通过 **`c.refill()`** **向 mcentral 重新申请一个 mspan**（runtime/mcache.go）：
```go
// refill acquires a new span of span class spc for c. This span will
// have at least one free object. The current span in c must be full.
//
// Must run in a non-preemptible context since otherwise the owner of
// c could change.
func (c *mcache) refill(spc spanClass) {
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		if go115NewMCentralImpl {
			mheap_.central[spc].mcentral.uncacheSpan(s)
		} else {
			atomic.Store(&s.sweepgen, mheap_.sweepgen)
		}
	}

	// Get a new cached span from the central lists.
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

	c.alloc[spc] = s
}
```
1. 标记当前 mspan 不会在被缓存（猜想，垃圾回收后会改变这个标志位），go 1.15 后的会将 mspan 从 mcache 还给 mcentral；
2. 通过 mheap **对应大小的 mcentral 获取一个空闲的 mspan**，并置位 `mspan.sweepgen`（表明被 mcache 使用）；
3. 将 `c.alloc` 对应大小的 mspan 赋值为刚获取到的；

可以看到，通过 refill 动态的申请 mspan，mcache 不同大小的 mspan 在 **申请->使用->替换** 中不断的循环，而 cache 一直能保存着有着空闲空间的 mspan 供 P 使用。


### 4.4 mcentral
**`mcentral`** 是内存分配器的中心缓存，**用于给 mcache 提供空闲的 mspan**。因为不是 P 对应的，所以访问也需要锁。

mheap 会创建 64 个 sizeClass 的 mcentral，每个 mcentral 管理相同大小的所有 mspan，以两个链表结构管理：
* **`nonempty`** ：包含空闲空间的 mspan 组成的链表；
* **`empty`** ：不包含空闲空间，或者被 mcache 申请的 mspan 组成的链表（判断是否被 mcache 使用是通过 mspan.sweepgen 属性来判断）；

当 mcache 要申请某大小的 mspan 时，会回去指定大小的 mcentral 实例上申请。
{{< find_img "img5.png" >}}

数据结构如下：
```go
// Central list of free objects of a given size.
//
//go:notinheap
type mcentral struct {
	lock      mutex
	spanclass spanClass

	// For !go115NewMCentralImpl.
	nonempty mSpanList // list of spans with a free object, ie a nonempty free list
	empty    mSpanList // list of spans with no free objects (or cached in an mcache)

	…
}
```
* `lock` ：访问需要加的锁；
* `spanclass` ：当前 mcentral 管理的 mspan 大小；
* `nonempty` ：包含空闲空间的 mspan 链表；
* `empty` ：不包含空闲空间，或者被 mcache 申请了的 mspan 链表；

{{< admonition note Note>}}
源码中存在 go115NewMCentralImpl 的注释，对 mcentral 结构做了很大的改动，但是在 go1.15 release 页面上并没有看到对应的说明。

其 commit 见: [runtime: add new mcentral implementation](https://github.com/golang/go/commit/a13691966ad571ed9e434d591a2d612c51349fd1)
{{< /admonition >}}

#### 4.4.1 从 mcentral 申请 mspan
[**在 mcache 中**](#mcache-alloc)，可以看到 mcache 通过调用 `mcentral.cacheSpan()` 申请新的空闲 mspan。在 go1.15 中，因为有新版 mcentral 的实现，因此双链表方式移动到了 `mcentral.oldCacheSpan()` 方法中。
```go
// Allocate a span to use in an mcache.
func (c *mcentral) cacheSpan() *mspan {
	if !go115NewMCentralImpl {
		return c.oldCacheSpan()
	}
	…
}

// Allocate a span to use in an mcache.
//
// For !go115NewMCentralImpl.
func (c *mcentral) oldCacheSpan() *mspan {
        lock(&c.lock)
        sg := mheap_.sweepgen
        
retry:
        var s *mspan
        // 走 nonempty 链表找
        for s = c.nonempty.first; s != nil; s = s.next {
                if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
                        c.nonempty.remove(s)
                        c.empty.insertBack(s)
                        unlock(&c.lock)
                        s.sweep(true)
                        goto havespan
                }
                if s.sweepgen == sg-1 {
                        // the span is being swept by background sweeper, skip
                        continue
                }
                // we have a nonempty span that does not require sweeping, allocate from it
                c.nonempty.remove(s)
                c.empty.insertBack(s)
                unlock(&c.lock)
                goto havespan
        }

        // 走 empty 链表找
        for s = c.empty.first; s != nil; s = s.next {
                if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
                        // we have an empty span that requires sweeping,
                        // sweep it and see if we can free some space in it
                        c.empty.remove(s)
                        // swept spans are at the end of the list
                        c.empty.insertBack(s)
                        unlock(&c.lock)
                        s.sweep(true)
                        freeIndex := s.nextFreeIndex()
                        if freeIndex != s.nelems {
                                s.freeindex = freeIndex
                                goto havespan
                        }
                        lock(&c.lock)
                        // the span is still empty after sweep
                        // it is already in the empty list, so just retry
                        goto retry
                }
                if s.sweepgen == sg-1 {
                        // the span is being swept by background sweeper, skip
                        continue
                }
                // already swept empty span,
                // all subsequent ones must also be either swept or in process of sweeping
                break
        }
        unlock(&c.lock)

        // 向 heap 申请新的 mspan
        // Replenish central list if empty.
        s = c.grow()
        if s == nil {
                return nil
        }
        lock(&c.lock)
        c.empty.insertBack(s)
        unlock(&c.lock)

        // At this point s is a non-empty span, queued at the end of the empty list,
        // c is unlocked.
havespan:
        …
        freeByteBase := s.freeindex &^ (64 - 1)
        whichByte := freeByteBase / 8
        // Init alloc bits cache.
        s.refillAllocCache(whichByte)

        // Adjust the allocCache so that s.freeindex corresponds to the low bit in
        // s.allocCache.
        s.allocCache >>= s.freeindex % 64

        return s
}
```
上面逻辑可以大致分为几个步骤：
1. 遍历 nonempty 链表，找到可用的 mspan（对于需要 sweep 的 mspan 先进行 sweep）；
1. 没找到，遍历 empty 链表，仅仅遍历需要 sweep 的 mspan，执行 sweep 并判断是否可用；
1. 还是没有，通过 `mcentral.grow()` 向 mheap 申请新的 mspan，mheap 中都没有，return nil；
1. 找到空闲 mspan 后，会放置到 empty 链表尾部，并返回；


<a id="mcentral-grow"></a>

#### 4.4.2 mcentral 扩容
在 mcentral 没有任何空闲 mspan 给 mcache 时，就会调用 `mcentral.grow()` 申请新的 mspan。
```go
// grow allocates a new empty span from the heap and initializes it for c's size class.
func (c *mcentral) grow() *mspan {
        npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
        size := uintptr(class_to_size[c.spanclass.sizeclass()])

        s := mheap_.alloc(npages, c.spanclass, true)
        if s == nil {
                return nil
        }

        // Use division by multiplication and shifts to quickly compute:
        // n := (npages << _PageShift) / size
        n := (npages << _PageShift) >> s.divShift * uintptr(s.divMul) >> s.divShift2
        s.limit = s.base() + size*n
        heapBitsForAddr(s.base()).initSpan(s)
        return s
}
```
1. 通过 `mheap.alloc()` 申请一个指大小的 mspan；
2. 执行 `mheapBit.initSpan()` 初始化 mspan；

#### 4.4.3 mcentral 回收 mspan
TODO

### 4.5 mheap
mheap 是最核心的组件了，runtime 只存在一个 mheap 对象，分配、初始化 mspan 都从 mheap 开始。

mheap 直接与虚拟内存打交道，并将在虚拟内存上创建 mspan 提供给上层使用。

mheap 的功能可以看做两个方面：
1. 与下层（虚拟内存）：**内存管理**（类似文件系统）。申请虚拟内存得到多个 heaparena，每个 heaparena 将可用内存区域切分为 page 单元，以倍数组成 mspan 分配给上层；
2. 与上层：**提供创建 mspan 的接口**。通过 mcentral 分类不同大小的 mspan，或者大内存需要直接走 mspan 分配；

其数据结构很大，省略了部分不会提到的属性（runtime/mheap.go），**`mheap_`** 就是 heap 的单实例对象：
```go
var mheap_ mheap

// Main malloc heap.
// The heap itself is the "free" and "scav" treaps,
// but all the other global data is here too.
//
// mheap must not be heap-allocated because it contains mSpanLists,
// which must not be heap-allocated.
//
//go:notinheap
type mheap struct {
		// arenas is the heap arena map. It points to the metadata for
		// the heap for every arena frame of the entire usable virtual
		// address space.
		//
		// Use arenaIndex to compute indexes into this array.
		//
		// For regions of the address space that are not backed by the
		// Go heap, the arena map contains nil.
		//
		// Modifications are protected by mheap_.lock. Reads can be
		// performed without locking; however, a given entry can
		// transition from nil to non-nil at any time when the lock
		// isn't held. (Entries never transitions back to nil.)
		//
		// In general, this is a two-level mapping consisting of an L1
		// map and possibly many L2 maps. This saves space when there
		// are a huge number of arena frames. However, on many
		// platforms (even 64-bit), arenaL1Bits is 0, making this
		// effectively a single-level map. In this case, arenas[0]
		// will never be nil.
		arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena
		
		// central free lists for small size classes.
		// the padding makes sure that the mcentrals are
		// spaced CacheLinePadSize bytes apart, so that each mcentral.lock
		// gets its own cache line.
		// central is indexed by spanClass.
		central [numSpanClasses]struct {
				mcentral mcentral
				pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
		}
		
		pages     pageAlloc // page allocation data structure
		
		spanalloc             fixalloc // allocator for span*
		cachealloc            fixalloc // allocator for mcache*
		specialfinalizeralloc fixalloc // allocator for specialfinalizer*
		specialprofilealloc   fixalloc // allocator for specialprofile*
		speciallock           mutex    // lock for special record allocators.
		arenaHintAlloc        fixalloc // allocator for arenaHints
}
```
* **`arenas`** ：内存管理的元信息数组，对于虚拟内存的逻辑切割与管理就靠这个数组了；
* **`central`**  ：按照大小分类的各个 mcentral 对象；
* `pages` ：在 arena 区域上用于分配空闲的 pages，依旧使用空闲链表；
* `spanalloc`、`cachealloc` 等 ：各个数据结构的空闲链表分配器，通过连接空闲的 mspan、mcache 等对象，调用 fixalloc.alloc() 函数就获取下一个空闲的内存空间；

#### 4.5.1 虚拟内存布局
网上大部分文章还是说 mheap 管理的虚拟内存以 spans+bitmap+arena 管理，如下图：
{{< find_img "img6.png" >}}

但是从 go 1.11 开始，Go 开始使稀疏内存方式管理，即管理相互之间不连续的连续的内存区域，如下图（**图片来自 《Golang 设计与实现》**）：
{{< find_img "img7.png" >}}

使用的就是 `mheap.arenas`，一个二维的 heapArena 数组。

不同平台的 heapArena 管理的 arena 大小不同，在 Linux 64bit 平台下，每个 heapArena 管理着 64MB 的 arena 内存区域。
```go
// Currently, we balance these as follows:
//
//       Platform  Addr bits  Arena size  L1 entries   L2 entries
// --------------  ---------  ----------  ----------  -----------
//       */64-bit         48        64MB           1    4M (32MB)
// windows/64-bit         48         4MB          64    1M  (8MB)
//       */32-bit         32         4MB           1  1024  (4KB)
//     */mips(le)         31         4MB           1   512  (2KB)
```
{{< admonition note Note>}}
这里不太好理解，但是我觉可以简单理解就是，将原来的 spans+bitmap+arena 管理方式，变为了**多个 spans+bitmap+arena 实现**。而不同 arena 之间的地址不是连续的。

但是为什么要用二维数组？目前不知道，但是 Linux x86-64 架构上一维数组大小为 1，就是相当于 1 维数组。
{{< /admonition >}}

**`heapArena`** 数据结构如下（runtime/mheap.go）：
```go
// A heapArena stores metadata for a heap arena. heapArenas are stored
// outside of the Go heap and accessed via the mheap_.arenas index.
//
//go:notinheap
type heapArena struct {
        // bitmap stores the pointer/scalar bitmap for the words in
        // this arena. See mbitmap.go for a description. Use the
        // heapBits type to access this.
        bitmap [heapArenaBitmapBytes]byte

        // spans maps from virtual address page ID within this arena to *mspan.
        // For allocated spans, their pages map to the span itself.
        // For free spans, only the lowest and highest pages map to the span itself.
        // Internal pages map to an arbitrary span.
        // For pages that have never been allocated, spans entries are nil.
        //
        // Modifications are protected by mheap.lock. Reads can be
        // performed without locking, but ONLY from indexes that are
        // known to contain in-use or stack spans. This means there
        // must not be a safe-point between establishing that an
        // address is live and looking it up in the spans array.
        spans [pagesPerArena]*mspan

        // pageInUse is a bitmap that indicates which spans are in
        // state mSpanInUse. This bitmap is indexed by page number,
        // but only the bit corresponding to the first page in each
        // span is used.
        //
        // Reads and writes are atomic.
        pageInUse [pagesPerArena / 8]uint8

        // pageMarks is a bitmap that indicates which spans have any
        // marked objects on them. Like pageInUse, only the bit
        // corresponding to the first page in each span is used.
        //
        // Writes are done atomically during marking. Reads are
        // non-atomic and lock-free since they only occur during
        // sweeping (and hence never race with writes).
        //
        // This is used to quickly find whole spans that can be freed.
        //
        // TODO(austin): It would be nice if this was uint64 for
        // faster scanning, but we don't have 64-bit atomic bit
        // operations.
        pageMarks [pagesPerArena / 8]uint8

        // pageSpecials is a bitmap that indicates which spans have
        // specials (finalizers or other). Like pageInUse, only the bit
        // corresponding to the first page in each span is used.
        //
        // Writes are done atomically whenever a special is added to
        // a span and whenever the last special is removed from a span.
        // Reads are done atomically to find spans containing specials
        // during marking.
        pageSpecials [pagesPerArena / 8]uint8

        // zeroedBase marks the first byte of the first page in this
        // arena which hasn't been used yet and is therefore already
        // zero. zeroedBase is relative to the arena base.
        // Increases monotonically until it hits heapArenaBytes.
        //
        // This field is sufficient to determine if an allocation
        // needs to be zeroed because the page allocator follows an
        // address-ordered first-fit policy.
        //
        // Read atomically and written with an atomic CAS.
        zeroedBase uintptr
}
```
* `bitmap`：表示该 arena 区域中哪些地址保存了对象，每个字节的前 4bit 的每个 bit 表示一个 8B 内存（4个指针大小）是否被扫描，后 4bit 每个 bit 表示是否包含指针。<br>
因此，一个字节就代表了 32B（4个指针，每个 8B）内存的状态。**（图片来自《知乎：图解Go语言内存分配》）**
* `spans`：每个 mspan 对应的指针，因为管理 64 MB，所以数组长度为 8192（64MB / 8KB）。<br>
其数组编号就是对应的 page 编号，例如 spans\[0] 就代表第一个 page 内存区域大小，执行对应的 mspan 。当然，mspan 可能有多个 page 组成，那么对应的多个数组项就指向的同一个 mspan 对象。
* `zeroedBase` 记录管理的 arena 的内存基地址。
{{< find_img "img8.png" >}}

在一个 heapAreana 空间下，对于任意一个内存地址 addr，(addr - zeroedBase)/8KB 我们能够计算出对应的 page 编号，那么通过 heapAreana.spans\[index] 就可以找到对应的 mspan。

#### 4.5.2 mheap 初始化
在 runtime 初始化时，会镜像 mheap 的初始化，执行 `mheap.init()` 函数：
```go
// Initialize the heap.
func (h *mheap) init() {
        lockInit(&h.lock, lockRankMheap)
        lockInit(&h.sweepSpans[0].spineLock, lockRankSpine)
        lockInit(&h.sweepSpans[1].spineLock, lockRankSpine)
        lockInit(&h.speciallock, lockRankMheapSpecial)

        h.spanalloc.init(unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
        h.cachealloc.init(unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
        h.specialfinalizeralloc.init(unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
        h.specialprofilealloc.init(unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)
        h.arenaHintAlloc.init(unsafe.Sizeof(arenaHint{}), nil, nil, &memstats.other_sys)

        // Don't zero mspan allocations. Background sweeping can
        // inspect a span concurrently with allocating it, so it's
        // important that the span's sweepgen survive across freeing
        // and re-allocating a span to prevent background sweeping
        // from improperly cas'ing it from 0.
        //
        // This is safe because mspan contains no heap pointers.
        h.spanalloc.zero = false

        // h->mapcache needs no init

        for i := range h.central {
                h.central[i].mcentral.init(spanClass(i))
        }

        h.pages.init(&h.lock, &memstats.gc_sys)
}
```
1. 初始化各个类型的空闲链表分配器：`spanalloc`、`cachealloc` 等；
1. 初始化各个 `mcentral`；
1. 初始化 `page alloctor`；

<a id="mheap-alloc_mspan"></a>

#### 4.5.3 mheap 分配 mspan
在 [**mcentral 扩容流程**](#mcentral-grow)中看到，会调用 `mheap.alloc()` 申请一个新的 mspan。

而之前说的 large object 分配，也是直接会走 `mheap.alloc()` 分配到一个合适大小的 mspan，然后分配 object。
```go
// alloc allocates a new span of npage pages from the GC'd heap.
//
// spanclass indicates the span's size class and scannability.
//
// If needzero is true, the memory for the returned span will be zeroed.
func (h *mheap) alloc(npages uintptr, spanclass spanClass, needzero bool) *mspan {
        // Don't do any operations that lock the heap on the G stack.
        // It might trigger stack growth, and the stack growth code needs
        // to be able to allocate heap.
        var s *mspan
        systemstack(func() {
                // To prevent excessive heap growth, before allocating n pages
                // we need to sweep and reclaim at least n pages.
                if h.sweepdone == 0 {
                        h.reclaim(npages)
                }
                s = h.allocSpan(npages, false, spanclass, &memstats.heap_inuse)
        })

        if s != nil {
                if needzero && s.needzero != 0 {
                        memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)
                }
                s.needzero = 0
        }
        return s
}

// allocSpan allocates an mspan which owns npages worth of memory.
//
// If manual == false, allocSpan allocates a heap span of class spanclass
// and updates heap accounting. If manual == true, allocSpan allocates a
// manually-managed span (spanclass is ignored), and the caller is
// responsible for any accounting related to its use of the span. Either
// way, allocSpan will atomically add the bytes in the newly allocated
// span to *sysStat.
//
// The returned span is fully initialized.
//
// h must not be locked.
//
// allocSpan must be called on the system stack both because it acquires
// the heap lock and because it must block GC transitions.
//
//go:systemstack
func (h *mheap) allocSpan(npages uintptr, manual bool, spanclass spanClass, sysStat *uint64) (s *mspan) {
        // Function-global state.
        gp := getg()
        base, scav := uintptr(0), uintptr(0)

        // If the allocation is small enough, try the page cache!
        pp := gp.m.p.ptr()
        if pp != nil && npages < pageCachePages/4 {
                c := &pp.pcache

        // If the cache is empty, refill it.
        if c.empty() {
                lock(&h.lock)
                *c = h.pages.allocToCache()
                unlock(&h.lock)
        }

        // Try to allocate from the cache.
        base, scav = c.alloc(npages)
        if base != 0 {
                s = h.tryAllocMSpan()

        if s != nil && gcBlackenEnabled == 0 && (manual || spanclass.sizeclass() != 0) {
                goto HaveSpan
        }
        }
        }
        
        if base == 0 {
                // Try to acquire a base address.
                base, scav = h.pages.alloc(npages)
                if base == 0 {
                        if !h.grow(npages) {
                                unlock(&h.lock)
                                return nil
                        }
                        base, scav = h.pages.alloc(npages)
                        if base == 0 {
                                throw("grew heap, but no adequate free space found")
                        }
                }
        }
        
        … 
HaveSpan:
        // 初始化 mspan
        …
        return s
}
```
函数很长，这里只保留了最关键的步骤：
1. 小内存，从当前 `p.pcache`（pageCache结构）获取一个内存区域（又是一个缓存~）；
1. `p.pcache` 中没有获取到，从 p.mspancache 中获取  mspan（还是缓存~）；
1. 内存比较大（large mspan），或者上面还是没有获取到，那么只能从 mheap 中获取了，加锁操作：
	1. 从 `pagealloc.alloc()` 获取内存；
	1. 没有获取到，`mheap.grow()` 进行堆扩容，再尝试一次；
1. 还是没有，panic !

最后获取到之后，就会走 mspan 的初始化流程，包括初始 mspan 数据结构，记录 mspan 指针到 `mheap.arenas` 等行为。

#### 4.5.4 mheap 回收 mspan
TODO

#### 4.5.5 mheap 扩容
扩容的逻辑更加复杂，目前先不细致的分析了。

这里可以看一下申请内存使用的方式：
```go
func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
        p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
        if err != 0 {
                return nil
        }
        return p
}

func sysMap(v unsafe.Pointer, n uintptr, sysStat *uint64) {
        mSysStatInc(sysStat, n)

        p, err := mmap(v, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_FIXED|_MAP_PRIVATE, -1, 0)
        if err == _ENOMEM {
                throw("runtime: out of memory")
        }
        if p != v || err != 0 {
                throw("runtime: cannot map pages in arena address space")
        }
}
```
1. 会先调用 `sysReserve()` 来申请内存，但是不使用，预备状态；
1. 经过地址检查后，`sysMap()` 将其转为可读可写，就可以使用该内存了；


### 4.6 总结
粗略地看完整个内存模型后，大概内存的结构如下：
{{< find_img "img9.png" >}}

其中比较核心的就是：**内存被切分为 page，多个 page 组成不同大小的 mspan，而在 mspan 上又分割为固定大小的 object**。

而上层的 mcache、mcentral 只是以不同的方式组织 mspan，**通过多级缓存的思想，使得并发的获取一个可用的 mspan 更快**。
* mcache 将一部分 mspan 独立于 P 所有，使得不需要加锁既可以获取 mspan；
* mcentral 以大小来分类 mspan，将各个大小的 mspan 请求独立，缩小了锁的粒度；

mheap 作为最底层，就好像文件系统一样，管理着整个内存分配的骨架。而与上层的交互就是靠 mspan 作为单位。


## 5 对象分配流程
前面一直提到的，对象的分配分为三类：
* **`tiny object`** `(0, 16B)`: 使用 tiny allocator 分配，使用 mcahe 一个独立的 mspan，挤压式的；
* **`object`** `[16B, 32KB]`: 使用 mcache 分配；
* **`large object`** `(32KB, +∞)`: 直接通过 mheap 分配；

所有的分配逻辑在 `mallocgc()` 开始分叉，下面分别看下具体的分配代码。

### 5.1 tiny object 分配
tiny object 分配的代码在 [**mspan 分配**](#mcache-alloc_mspan)中已经说明了，这里再理一下大致步骤：
1. 不包含指针(noscan)并且小于 16B 的对象才走微小对象分配；
2. tiny object 分配仅仅是增大 mcache.tinyoffset 的值，所以是不同大小 tiny object 挤压在一个 mspan 中；
3. 如果当前的 mspan 没有空间了，通过 mcache.nextFree() 来获取新的指定大小的 mspan，而获取的流程就是前面所说的（走 mcentral->mheap);

### 5.2 object 分配
普通大小 object 分配流程就很简单了:
```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			...
		} else {
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		...
	}
}
```
1. 计算出对应的 sizeclass；
2. 从 `mcache.alloc[]` 得到对应的 mspan。如果没有，通过 `nextFree()` 申请；
3. 调用 `memclrNoHeapPointers()` 清理空闲内存中所有数据；

### 5.3 large object 分配
```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			...
		} else {
			...
		}
	} else {
		shouldhelpgc = true
		systemstack(func() {
			span = largeAlloc(size, needzero, noscan)
		})
		span.freeindex = 1
		span.allocCount = 1
		x = unsafe.Pointer(span.base())
		size = span.elemsize
	}
}

func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
	// print("largeAlloc size=", size, "\n")

	if size+_PageSize < size {
		throw("out of memory")
	}
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}

	// Deduct credit for this span allocation and sweep if
	// necessary. mHeap_Alloc will also sweep npages, so this only
	// pays the debt down to npage pages.
	deductSweepCredit(npages*_PageSize, npages)

	spc := makeSpanClass(0, noscan)
	s := mheap_.alloc(npages, spc, needzero)
	if s == nil {
		throw("out of memory")
	}
	if go115NewMCentralImpl {
		// Put the large span in the mcentral swept list so that it's
		// visible to the background sweeper.
		mheap_.central[spc].mcentral.fullSwept(mheap_.sweepgen).push(s)
	}
	s.limit = s.base() + size
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
```
1. large object 会切换到系统栈，然后走 mheap 申请；
2. 计算对象需要的 page 数量，然后调用 mheap.alloc() 申请空闲的 mspan；

而 [**mheap.alloc()**](#mheap-alloc_mspan) 就是 mcentral 申请 mspan 的方法。


## 参考
<a id="reference"></a>

* [《Golang 学习笔记》](https://github.com/qyuhen/book)
* [Blog：Go 内存管理可视化](https://medium.com/@ankur_anand/a-visual-guide-to-golang-memory-allocator-from-ground-up-e132258453ed)
* [《Golang 设计与实现》：内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#71-%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8)
* [知乎：图解Go语言内存分配](https://zhuanlan.zhihu.com/p/59125443)
