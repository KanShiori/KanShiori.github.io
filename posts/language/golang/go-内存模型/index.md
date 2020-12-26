# Go 内存模型


本文是对 Golang 内存模型与内存管理的一个总结，基本内容来源于《Go 学习笔记》。
下面代码都是基于 go 1.15.6。

## 1 Linux 内存模型
所有语言的内存管理，在 Linux 上都是在以基本的进程内存模型基础上实现的，首先需要知道 Linux 进程内存布局。

在进程角度，看到的所有内存就是 **`虚拟地址空间`** ，整个是一个线性的存储地址。其中一部分高地址区域用户态无法访问，是内核地址空间。而另一部分就是由栈、mmap、堆等内存区域组成的用户地址空间。
{{< find_img "img1.png" >}}
上面进程可以自己分配与管理的进程，就是 mmap 与 堆，对应的系统调用为 **`mmap()`** 与 **`brk()`**，因此所有语言的内存管理都是基于这两个内存区域在进一步实现的（包括 glibc 的 malloc() 与 free()）。

mmap 最基本有两个用途：
* `文件映射` ：申请一块内存区域，映射到文件系统上一个文件（这也是 page_cache 的基本原理，所以他们在内核中都使用 address_space 实现）
* `匿名映射` ：申请一块内存区域，但是没有映射到具体文件，相当于分配了一块内存区域（可以用于父子进程共享、或者自己管理内存的分配等功能）

而所有在内存上所说的地址，包括代码指令地址、变量地址都是上面地址空间的一个地址。


## 2 PC 与 SP

Goroutine 将进程的切换变为了协程间的切换，那么就需要**在用户空间负责执行代码与协程上下文的保留与切换**。因此，有两个关键的寄存器：PC 与 SP。

### 2.1 PC
 **`程序计数器（Program Counter，PC）`** 是 CPU 中的一个寄存器，**保存着下一个 CPU 执行的指令的位置**。顺序执行指令时，PC = PC + 1（一个指令）。而调用函数或者条件跳转时，会将跳到的指令地址设置到 PC 中。

所以，可以想到，当需要切换执行的 goroutine，调用 JMP 指令跳转到 G 对应的代码。

### 2.2 SP
 **`SP`** 是**保存栈顶地址的寄存器**，我们平时所说的临时变量在栈上，就是将临时变量的值写入 SP 保存的内存地址，然后 SP 保存的地址减小（栈是从高地址向低地址变化），然后临时变量销毁时，SP 地址又变为高地址。

不过，因为 goroutine 切换时，必须要保存当前 goroutine 的上下文，也就是栈里的变量。因此，goroutine 栈肯定是不能使用 Linux 进程栈了（因为进程栈有上限，也无法实现“保存”这种功能）。所以所说的协程栈，都是基于 mmap 申请内存空间（基于 Go 内存管理，内存管理基于 mmap），然后切换时修改 SP 寄存器地址实现的。

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

### 3.1 新 G 的栈
在 **malg** 函数中，可以看到对于新 G 的栈的分配（一开始为 2KB）：
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
注意 **systemstack**，用于将当前栈切换到 M 的 g0 协程栈上执行命令。

Why? 因为 G 用于执行用户逻辑，而某些管理操作不方便在 G 栈上执行（例如 G 可能中途停止，垃圾回收时 G 栈空间也有可能被回收），所以需要执行管理命令时，都会通过 systemstack 方法将线程栈切换为 g0 的栈执行，与用户逻辑隔离。
{{< /admonition >}}

### 3.2 栈的分配
**stackalloc** 函数用于分配一个栈，无论是给新 G 还是扩容栈时都会用到，因此栈空间的分配与回收是一个比较频繁的操作，所以栈空间采取了缓存复用的方式。

主要逻辑如下：
1. 如果分配的栈空间不大，就走`缓存复用`这种方式分配。没有可以复用的就创建；
2. 如果分配的栈空间很大（大于 32KB），就直接从 heap 分配；

这里主要关注第 1 中方式，会调用 **stackpoolalloc** 函数。
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
可以看到，首先尝试从 stackpool 缓存的空闲的 stack 获取，如果没有则走 Go 内存管理申请一个。

再接下来就是 Go 内存管理模块负责的事了，不深入下去。底层创建都是使用 mmap 系统调用实现的，这里可以看下使用的参数：
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
通过 mmap 调用的参数可以看到，申请了一个系统分配的匿名内存映射。

### 3.3 栈的扩容

#### 3.3.1 扩容判断
Go 编译器会在执行函数前，插入一些汇编指令，其中一个功能就是**检查 G 栈是否需要扩容**。看一个函数调用的实现：
```text
$ go build -gcflags "-l" -o test main.go
$ go tool objump -s "main\.test" test
TEXT main.test(SB) /root/yusihao/onething/BizImages/main.go
  main.go:3 0x45dc80 MOVQ FS:0xfffffff8, CX  // CX 为当前 G 地址		
  main.go:3 0x45dc89 CMPQ 0x10(CX), SP  // CX+0x10 执行 g.stackguard0 属性，与 SP 指针地址比较	
  main.go:3 0x45dc8d JBE 0x45dccf	  // 如果 SP <=stackguard0  跳转到 0x45dccf，也就是调用 runtime.morestack_noctxt(SB) 函数	
  main.go:3 0x45dc8f SUBQ $0x18, SP	

  // ...

  main.go:5 0x45dcce RET  // 函数执行结束，RET 返回，不会执行后面两个指令				
  main.go:3 0x45dccf CALL runtime.morestack_noctxt(SB)  // 执行栈扩容	
  main.go:3 0x45dcd4 MP main.test(SB) // 执行结束后，重新执行当前函数	
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

具体会调用 newstack 函数（runtime/stack.go），过程很复杂，只看一下关键点：
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
当 M 执行的 G 需要切换，或者一个新创建 G 执行时，最后都会调用 execute() 函数，而 execute() 函数会调用 gogo 汇编实现的函数。
```text
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		// make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	// restore SP
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
`gobuf` 中保存着要执行的 G 的 sp、pc 指针，可以看到通过将对应 gobuf.sp 写入到 SP 寄存器中，也就是将使用的栈切换为了 G 的栈。
	


