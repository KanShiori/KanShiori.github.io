# Go 垃圾收集总结

> **总结系列的文章**是自己的学习或使用后，对相关知识的一个总结，用于后续可以快速复习与回顾。

本文是对 Golang 垃圾收集的一个总结，基本内容来源于网络的学习，以及自己观摩了下源码。

所以学习的书籍与文章见 [**参考**](#参考)。

下面代码都是基于 go 1.15.6。

## 1 背景知识

### 1.1 术语
垃圾回收算法有一些基本的术语，首先需要知道对应的含义：
* **`Mutator`**：具有“改变对象”的意思，GC 中就是改动对象间引用关系的意思，也就是程序；
* **`堆`**：对象使用的内存空间，GC 就是将垃圾对象空间放回到堆中；
* **`根对象`**：对象的指针的“起点”部分，一般就是全局对象和栈对象；

### 1.2 三色标记算法
三色标记算法是 GC [标记清除算法]^(Mark-Sweep) 的一种，也是 Golang 中使用的算法。
{{< admonition tip 推荐阅读>}}
推荐阅读文章，写的非常详细：[**垃圾收集器**](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#72-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8)
{{< /admonition >}}

首先，三色标记算法的最基本逻辑为：
1. 标记的最开始，**所有对象默认为`白色`**；
1. 将 **`根对象`标记为灰色**，放入灰色集合；
1. 从 **`灰色集合`** 中取出灰色对象，将其**子对象标记为`灰色`**，加入灰色集合，该灰色对象标记为黑色；
1. 重复第 3 步，直到灰色集合为空；
1. **清理所有的白色对象**；

整个逻辑很简单，就是一个树的层次遍历，将所有可达的结点标记，然后清理未标记的不可达结点。
{{< find_img "img1.png" >}}

但是，仅仅是普通的三色标记算法要求执行时，Mutator 不能同时运行。因为如果 Mutator 并行时，某个扫描过的结点的引用关系变化，就可能导致 [悬挂指针]^(dangling pointer) 问题。

例如，上图中第 3 步将 A 指向 D，那么 D 还是无法被标记，被错误回收。

而想要让 Mutator 同时运行时，标记的结果还保持正确，那么每个时刻标记的结果要满足 [三色不变性]^(Tri-color invariant) <a id="三色不变性"></a>
* **`强三色不变性`**：**黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象**。<br>
  因为黑色对象不会再被扫描，如果黑色对象指向白色对象，那么肯定该白色对象会被错误回收。<br>
  当然，除非这种情况能够满足弱三色不变性。
* **`弱三色不变性`**：**黑色对象执行白色对象，那么必须包含一条灰色对象经由多个白色对象的可达路径**。<br>
  因为有了后面这个可达路径，也就是说白色对象还是可以被标记的，那么黑色对象可以指向该白色对象。

### 1.3 屏障技术
为了满足两个不变性，所以要在对象引用变更时，做出一些操作改变对象标记。这就是 GC 里 [屏障技术]^(barrier) 的作用。

Go 中使用了写屏障，即**在用户程序更新对象指针时，执行一些代码，使得继续满足不变性**。
{{< admonition note Note>}}
这里的屏障技术似乎和我知道的 CPU 的屏障技术含义不太类似，**更像是`回调函数`**，也挺困惑
{{< /admonition >}}

#### 1.3.1 插入写屏障
Dijkstra 提出的 **`插入写屏障`**，在更新对象指针时，将其被指向的对象重新加入扫描集合（三色标记中也就是变为灰色），这样接下来还是能够被扫描。
{{< find_img "img2.png" >}}

可以看到，这样黑色对象始终指向的是灰色对象，永远都会满足强三色不变性。

但是，Dijkstra 也有一些缺点：
* 对象指针变动时，没有考虑旧的指针引用。例如 `*field` 原来的对象 oldobj 已经扫描成黑色了，那么 `*field = newobj` 变动后，可能 oldobj 变为垃圾对象，只有等到下一轮标记时才会被回收。
* TODO

#### 1.3.2 删除写屏障
Yuasa 提出的 **`删除写屏障`**，让老对象的引用被删除时，将白色的老对象涂成灰色，这样删除写屏障就可以保证弱三色不变性。
{{< find_img "img3.png" >}}

#### 1.3.3 Go 中的屏障
Go 中使用 **`混合写屏障`**，即插入写屏障与删除写屏障都开启，并且在标记阶段开始后，将创建的所有新对象都标记为黑色，防止新分配的对象被错误的回收。

具体操作为：
1. GC 开始将栈上的对象全部扫描并标记为黑色（之后不再进行第二次重复扫描，无需 STW)，
1. GC 期间，任何在栈上创建的新对象，均为黑色。
1. 被删除的对象标记为灰色。
1. 被添加的对象标记为灰色。

## 2 三色标记算法实现
我们先不看整个的流程实现，而是从核心的标记算法入手。

### 2.1 标记

#### 2.1.1 并发标记框架
runtime 使用并发的标记方式，多个 groutine 同时执行一部分内存的标记操作。其整个就是基于一个**工作池框架**。
首先，有着一个全局的工作队列，其放在 `work` 全局变量中：
```go
var work struct {
	full  lfstack          // lock-free list of full blocks workbuf
	empty lfstack          // lock-free list of empty blocks workbuf
}
```
* `full`：全局的非空的队列组成的链表，单个 groutine 没有任务时就从这里取一个队列使用；
* `empty`：全局的空的队列组成的链表，单个 groutine 的工作队列满时就会这里去一个队列切换；

每个 goroutine 有着独立对应的 `gcWork` 对象，其类似一个队列，从队列中取出需要扫描的内存的地址。
```go
type workbuf struct {
        workbufhdr
        obj [(_WorkbufSize - unsafe.Sizeof(workbufhdr{})) / sys.PtrSize]uintptr
}

type gcWork struct {
        // wbuf1 and wbuf2 are the primary and secondary work buffers.
        //
        // This can be thought of as a stack of both work buffers'
        // pointers concatenated. When we pop the last pointer, we
        // shift the stack up by one work buffer by bringing in a new
        // full buffer and discarding an empty one. When we fill both
        // buffers, we shift the stack down by one work buffer by
        // bringing in a new empty buffer and discarding a full one.
        // This way we have one buffer's worth of hysteresis, which
        // amortizes the cost of getting or putting a work buffer over
        // at least one buffer of work and reduces contention on the
        // global work lists.
        //
        // wbuf1 is always the buffer we're currently pushing to and
        // popping from and wbuf2 is the buffer that will be discarded
        // next.
        //
        // Invariant: Both wbuf1 and wbuf2 are nil or neither are.
        wbuf1, wbuf2 *workbuf
        …
}
```
* `wbuf1` 与 `wbuf2`: 就是主队列与备队列。所有操作都会先操作 wbuf1，如果 wbuf1 空间不足或者没有对象，就会触发 wbuf1 与 wbuf2 切换。当两个缓冲区都空间不足或者满时，就会从 work.free 或者 work.list 得到一个空闲的，并赋值 wbuf1。

gcWork 有几个重要的方法：
* `gcWork.tryGetFast()` ：从 wbuf1 快速得到一个 obj 的地址；
* `gcWork.tryGet()` ：从 wbuf1 -> wbuf2 -> work.full 获取一个 obj；
* `gcWork.putFast()` ：将一个待扫描的 obj 放入 wbuf1；
* `gcWork.put()` ：将一个待扫描的 obj 放入 wbuf1 -> wbuf2 -> work.empty；
* `gcWork.balance()` ：将 wbuf2/wbuf1 放入 work.full 中；
* `gcWork.empty()` ：判断是否 gcWork 为空，为空表明没有会灰色 obj；

可以看到，整个过程就是围绕了各个工作队列的生产-消费过程。**先从根对象开始，goroutine 从 gcWork 取出待扫描的 obj，将其标记，然后将 obj 指向的子 obj 再次放入 gcWork。不断循环，直到 gcWork 为空**。

#### 2.1.2 groutine 标记流程
下面看下核心的标记流程，这里我们仅仅关注一个 groutine 的工作。大致的标记步骤如下：
1. 将根对象放入 gcWork；
1. groutine 从 gcWork 取出一个 object 地址，将其标记；
1. 将 object 包含的指针指向的 object 再次放入 gcWork；
1. 重复 2-3 步，直到 gcWork 为空；

而对应于三色标记，我们可以确认不同颜色的对象在 runtime 中的对应：
* 灰色对象 -> gcWork 中的 object；
* 黑色对象 -> 不在 gcWork 中，但是被 mark 的 object；
* 白色对象 -> 不在 gcWork 中，没有被 mark 的 object；

每个 P 会对应一个标记使用的 groutine，执行 **`gcDrain()`** 函数（runtime/mgcmark.go）：
```go
// gcDrain scans roots and objects in work buffers, blackening grey
// objects until it is unable to get more work. It may return before
// GC is done; it's the caller's responsibility to balance work from
// other Ps.
//
// If flags&gcDrainUntilPreempt != 0, gcDrain returns when g.preempt
// is set.
//
// If flags&gcDrainIdle != 0, gcDrain returns when there is other work
// to do.
//
// If flags&gcDrainFractional != 0, gcDrain self-preempts when
// pollFractionalWorkerExit() returns true. This implies
// gcDrainNoBlock.
//
// If flags&gcDrainFlushBgCredit != 0, gcDrain flushes scan work
// credit to gcController.bgScanCredit every gcCreditSlack units of
// scan work.
//
// gcDrain will always return if there is a pending STW.
//
//go:nowritebarrier
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
        gp := getg().m.curg
        preemptible := flags&gcDrainUntilPreempt != 0
        flushBgCredit := flags&gcDrainFlushBgCredit != 0
        idle := flags&gcDrainIdle != 0

        initScanWork := gcw.scanWork

        // 配置退出标记的 check 函数，根据不同策略退出标记
        checkWork := int64(1<<63 - 1)
        var check func() bool
        if flags&(gcDrainIdle|gcDrainFractional) != 0 {
                checkWork = initScanWork + drainCheckThreshold
                if idle {
                        check = pollWork
                } else if flags&gcDrainFractional != 0 {
                        check = pollFractionalWorkerExit
                }
        }

        // 扫描根对象放入 gcWork
        if work.markrootNext < work.markrootJobs {
                // Stop if we're preemptible or if someone wants to STW.
                for !(gp.preempt && (preemptible || atomic.Load(&sched.gcwaiting) != 0)) {
                        job := atomic.Xadd(&work.markrootNext, +1) - 1
                        if job >= work.markrootJobs {
                                break
                        }
                        markroot(gcw, job)
                        if check != nil && check() {
                                goto done
                        }
                }
        }

        // 标记循环
        for !(gp.preempt && (preemptible || atomic.Load(&sched.gcwaiting) != 0)) {
                if work.full == 0 {
                        gcw.balance()
                }

        // 从 gcWork 得到一个 object addr（灰色对象）
        b := gcw.tryGetFast()
        if b == 0 {
                b = gcw.tryGet()
                if b == 0 {
                        // Flush the write barrier
                        // buffer; this may create
                        // more work.
                        wbBufFlush(nil, 0)
                        b = gcw.tryGet()
                }
        }
        if b == 0 {
                // 没有任何灰色对象，退出
                break
        }
        
        // 标记 object（灰色对象变为黑色），并扫描子 object，加入 gcWork
        scanobject(b, gcw)

        // 检查是否退出
        if gcw.scanWork >= gcCreditSlack {
                atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
                if flushBgCredit {
                        gcFlushBgCredit(gcw.scanWork - initScanWork)
                        initScanWork = 0
                }
                checkWork -= gcw.scanWork
                gcw.scanWork = 0

        if checkWork <= 0 {
                checkWork += drainCheckThreshold
                if check != nil && check() {
                        break
                }
        }

done:
        if gcw.scanWork > 0 {
                atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
                if flushBgCredit {
                        gcFlushBgCredit(gcw.scanWork - initScanWork)
                }
                gcw.scanWork = 0
        }
}
```
##### (1) 根对象构建
上述流程看到，所有 gourtine 会不断调用 `makeroot()` 函数，用于多个 goroutine 并行扫描根对象，并放入 gcWork 中。
```go
// markroot scans the i'th root.
//go:nowritebarrier
func markroot(gcw *gcWork, i uint32) {
        baseFlushCache := uint32(fixedRootCount)
        baseData := baseFlushCache + uint32(work.nFlushCacheRoots)
        baseBSS := baseData + uint32(work.nDataRoots)
        baseSpans := baseBSS + uint32(work.nBSSRoots)
        baseStacks := baseSpans + uint32(work.nSpanRoots)
        end := baseStacks + uint32(work.nStackRoots)

        // Note: if you add a case here, please also update heapdump.go:dumproots.
        switch {
        case baseFlushCache <= i && i < baseData:
                flushmcache(int(i - baseFlushCache))
        case baseData <= i && i < baseBSS:
                for _, datap := range activeModules() {
                        markrootBlock(datap.data, datap.edata-datap.data, datap.gcdatamask.bytedata, gcw, int(i-baseData))
                }
        case baseBSS <= i && i < baseSpans:
                for _, datap := range activeModules() {
                        markrootBlock(datap.bss, datap.ebss-datap.bss, datap.gcbssmask.bytedata, gcw, int(i-baseBSS))
                }
        case i == fixedRootFinalizers:
                for fb := allfin; fb != nil; fb = fb.alllink {
                        cnt := uintptr(atomic.Load(&fb.cnt))
                        scanblock(uintptr(unsafe.Pointer(&fb.fin[0])), cnt*unsafe.Sizeof(fb.fin[0]), &finptrmask[0], gcw, nil)
                }
        case i == fixedRootFreeGStacks:
                // Switch to the system stack so we can call
                // stackfree.
                systemstack(markrootFreeGStacks)
        case baseSpans <= i && i < baseStacks:
                // mark mspan.specials
                markrootSpans(gcw, int(i-baseSpans))
        default:
                // 从 G 总量链表取出需要扫描的 G
                // 因为调用 makeroot() 会不断递增 job，所以不会重复扫描
                var gp *g
                if baseStacks <= i && i < end {
                        gp = allgs[i-baseStacks]
                } else {
                        throw("markroot: bad index")
                }
                
                …

        // scanstack must be done on the system stack in case
        // we're trying to scan our own stack.
        systemstack(func() {
                // 一些检查
                …
                
                // 扫描 goroutine stack
                scanstack(gp, gcw)
                gp.gcscandone = true
                resumeG(stopped)
        })
        }
}
```
由此我们可以知道，主要有哪些根对象：**数据段**、**缓存**、**BSS 段（全局变量与静态变量）**、**mspan.special（runtime 内部使用的特殊对象）**、**Groutine 的栈上的对象**。

其中，大部分扫描都会是栈上的对象，**通过得到对应 object 的地址，然后判断其地址是否存在与 heap 管理的 mspan 中决定其是否是根对象。**

{{< admonition note Note>}}
在 [**Go 内存管理总结**](../memory-manager/#42-mspan) 中说过，**任意的内存地址，都可以通过公式得到其对应的 mspan 的地址**，这也可以用于判断一个地址是否是存在与 mheap 上的。
{{< /admonition >}}

所有扫描到的 object 地址通过 `greyobject()` 函数进行 mark 并放入 gcWork。

##### (2) 标记流程
`gcDrain()` 中的核心 for 循环就是不断标记的流程，每个 object 会调用 `scanobject()` 函数：
```go
// scanobject scans the object starting at b, adding pointers to gcw.
// b must point to the beginning of a heap object or an oblet.
// scanobject consults the GC bitmap for the pointer mask and the
// spans for the size of the object.
//
//go:nowritebarrier
func scanobject(b uintptr, gcw *gcWork) {
        // 得到 b 对应 object 的 heapArena.bitmap 与 mspan
        hbits := heapBitsForAddr(b)
        s := spanOfUnchecked(b)
        n := s.elemsize
        
        // large object 对象的子 object 扫描
        if n > maxObletBytes {
                if b == s.base() {
                        if s.spanclass.noscan() {
                                // Bypass the whole scan.
                                gcw.bytesMarked += uint64(n)
                                return
                        }

        for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
                if !gcw.putFast(oblet) {
                        gcw.put(oblet)
                }
        }

        n = s.base() + s.elemsize - b
        if n > maxObletBytes {
                n = maxObletBytes
        }

        // 普通 object 的子对象扫描
        // 通过指针大小的地址偏移不断扫描
        var i uintptr
        for i = 0; i < n; i += sys.PtrSize {
                // Find bits for this word.
                if i != 0 {
                        // Avoid needless hbits.next() on last iteration.
                        hbits = hbits.next()
                }
                
                通过 bitmap 判断其是否是一个指针
                bits := hbits.bits()
                if i != 1*sys.PtrSize && bits&bitScan == 0 {
                        break // no more pointers in this object
                }
                if bits&bitPointer == 0 {
                        continue // not a pointer
                }

        // 指针的值，也是就 object 地址
        obj := *(*uintptr)(unsafe.Pointer(b + i))

        // 依旧通过 heapArena 的公式找到对应的 obj、mspan，将其放入 gcWork 
        if obj != 0 && obj-b >= n {
                if obj, span, objIndex := findObject(obj, b, i); obj != 0 {
                        greyobject(obj, b, i, span, gcw, objIndex)
                }
        }
        gcw.bytesMarked += uint64(n)
        gcw.scanWork += int64(i)
}
```
可以看到，通过当前 object 地址的**不断偏移 8 字节**，然后**通过 `heapArena.bitmap` 判断当前 8 字节是否是指针**，如果是指针就将其放入 gcWork 对象。
{{< admonition note Note>}}
在 [**Go 内存管理总结**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/#451-%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80) 中看到，**对于 heap 中每 8 个字节，通过存在对应 bit 位标识是否指针，以及是否被 mark**；
{{< /admonition >}}

`greyobject()` 用于将该 object 标记，并放入 gcWork 中：
```go
// obj is the start of an object with mark mbits.
// If it isn't already marked, mark it and enqueue into gcw.
// base and off are for debugging only and could be removed.
//
// See also wbBufFlush1, which partially duplicates this logic.
//
//go:nowritebarrierrec
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {
        mbits := span.markBitsForIndex(objIndex)

        // mark object
        if useCheckmark {
                hbits := heapBitsForAddr(obj)
                if hbits.isCheckmarked(span.elemsize) {
                        return
                }
                hbits.setCheckmarked(span.elemsize)
                if !hbits.isCheckmarked(span.elemsize) {
                        throw("setCheckmarked and isCheckmarked disagree")
                }
        } else {
                // If marked we have nothing to do.
                if mbits.isMarked() {
                        return
                }
                mbits.setMarked()

        // Mark span.
        arena, pageIdx, pageMask := pageIndexOf(span.base())
        if arena.pageMarks[pageIdx]&pageMask == 0 {
                atomic.Or8(&arena.pageMarks[pageIdx], pageMask)
        }

        // If this is a noscan object, fast-track it to black
        // instead of greying it.
        if span.spanclass.noscan() {
                gcw.bytesMarked += uint64(span.elemsize)
                return
        }

        // 放入 gcWork 中
        if !gcw.putFast(obj) {
                gcw.put(obj)
        }
}
```

### 2.2 写屏障
在各个代码的注释中，可以看到 "//go:nowritebarrier"。显然，这样让编译器编译时不加入写屏障的意思，因此可以想到，默认的函数执行都会加入写屏障的逻辑。

在 SSA 中间代码生成阶段，编译器会在 Store、Move、Zero 操作中加入写屏障，写屏障函数为 `writebarrier()` 函数（cmd/compile/internal/ssa/writebarrier.go）。

`writebarrier()` 函数很复杂，这里不展开。再次看一下混合写屏障操作：
1. GC 开始将栈上的对象全部扫描并标记为黑色（之后不再进行第二次重复扫描，无需 STW)。
1. GC 期间，任何在栈上创建的新对象，均为黑色。
1. 被删除的对象标记为灰色。
1. 被添加的对象标记为灰色。

当开始 GC 时，全局变量 `runtime.writeBarrier.enabled` 变为 true，所有的写操作都会经过 `writebarrier()` 的操作。

## 3 内存清理
内存清理与标记就是完全分隔的逻辑了，通过判断对象是否被标记就可决定是否将其内存回收。

**`gcSweep()`** 函数用于在 GC 标记结束后执行清理（src/runtime/mgc.go）：
```go
// gcSweep must be called on the system stack because it acquires the heap
// lock. See mheap for details.
//
// The world must be stopped.
//
//go:systemstack
func gcSweep(mode gcMode) {
        lock(&mheap_.lock)
        // 关键的 sweepgen 变量
        mheap_.sweepgen += 2
        mheap_.sweepdone = 0
        if !go115NewMCentralImpl && mheap_.sweepSpans[mheap_.sweepgen/2%2].index != 0 {
                // We should have drained this list during the last
                // sweep phase. We certainly need to start this phase
                // with an empty swept list.
                throw("non-empty swept list")
        }
        mheap_.pagesSwept = 0
        mheap_.sweepArenas = mheap_.allArenas
        mheap_.reclaimIndex = 0
        mheap_.reclaimCredit = 0
        unlock(&mheap_.lock)

        if go115NewMCentralImpl {
                sweep.centralIndex.clear()
        }

	    // 阻塞清理
        if !_ConcurrentSweep || mode == gcForceBlockMode {
                // Special case synchronous sweep.
                // Record that no proportional sweeping has to happen.
                lock(&mheap_.lock)
                mheap_.sweepPagesPerByte = 0
                unlock(&mheap_.lock)
                
                // 清理 span !
                // Sweep all spans eagerly.
                for sweepone() != ^uintptr(0) {
                        sweep.npausesweep++
                }
                
                // Free workbufs eagerly.
                prepareFreeWorkbufs()
                for freeSomeWbufs(false) {
                }
                // All "free" events for this mark/sweep cycle have
                // now happened, so we can make this profile cycle
                // available immediately.
                mProf_NextCycle()
                mProf_Flush()
                return
        }
 
	      // 后台清理（并发清理）
        // Background sweep.
        lock(&sweep.lock)
        if sweep.parked {
                sweep.parked = false
                ready(sweep.g, 0, true)
        }
        unlock(&sweep.lock)
}
```
1. 不断执行 `sweepone()` 来清理 mspan；
1. 启动后台并发清理；

### 3.1 阻塞清理
通过不断执行 `sweepone()` 来进行 mspan 的清理，`sweepone()` 从 heap 得到一个 mspan 并清理（src/runtime/mgcsweep.go）：
```go
// sweepone sweeps some unswept heap span and returns the number of pages returned
// to the heap, or ^uintptr(0) if there was nothing to sweep.
func sweepone() uintptr {
  …
  
  // 得到一个被清理的 mspan
  var s *mspan
  sg := mheap_.sweepgen
  for {
          if go115NewMCentralImpl {
                  s = mheap_.nextSpanForSweep()
          } else {
                  s = mheap_.sweepSpans[1-sg/2%2].pop()
          }
          if s == nil {
                  atomic.Store(&mheap_.sweepdone, 1)
                  break
          }
          // 设置标记
          if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
                  break
          }
  } 
  // 清理 mspan
  npages := ^uintptr(0)
  if s != nil {
          npages = s.npages
          if s.sweep(false) {
                  // Whole span was freed. Count it toward the
                  // page reclaimer credit since these pages can
                  // now be used for span allocation.
                  atomic.Xadduintptr(&mheap_.reclaimCredit, npages)
          } else {
                  // Span is still in-use, so this returned no
                  // pages to the heap and the span needs to
                  // move to the swept in-use list.
                  npages = 0
          }
  } 
  …
  
  return npages
}
```
最终的回收工作靠 `mspan.sweep()` 完成，这给在 [**Go 内存管理总结**](https://kanshiori.github.io/posts/language/golang/go-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%80%BB%E7%BB%93/#451-%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80) 中可以看到具体实现，大致就是将 GC 后没有被 mark 的 object 记录为可用，以后续申请使用覆盖。如果整个 mspan 变回空，就由 mheap 回收其对应的 page。

### 3.2 并发清理
并发清理就是一个死循环，被唤醒后开始执行清理任务。
```go
func bgsweep(c chan int) {
	sweep.g = getg()

	lockInit(&sweep.lock, lockRankSweep)
	lock(&sweep.lock)
	sweep.parked = true
	c <- 1
	goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceEvGoBlock, 1)

	for {
		// 依旧通过 sweepone() 清理
		for sweepone() != ^uintptr(0) {
			sweep.nbgsweep++
			Gosched()
		}
		for freeSomeWbufs(true) {
			Gosched()
		}
		lock(&sweep.lock)
		if !isSweepDone() {
			// This can happen if a GC runs between
			// gosweepone returning ^0 above
			// and the lock being acquired.
			unlock(&sweep.lock)
			continue
		}
		
		// 等待唤醒
		sweep.parked = true
		goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceEvGoBlock, 1)
	}
}
```
依旧是通过不断执行 `sweepone()` 进行清理。

## 4 标记流程

下面来看如何触发的 GC 以及 GC 的大致流程。

### 4.1 GC 触发

GC 有三个点会被触发：

1. runtime 启动后会启动一个后台 gourtine，被唤醒后就会执行 GC，而**唤醒操作由 sysmon 负责执行**。<br>
sysmon 会根据系统情况决定是否触发。

1. **分配新 object 时**（[**mallocgc() 函数**](../memory-manager/#52-object-分配)），如果 mcache 需要重新刷新，或者是分配的是 large object，那么也会触发一次 GC。

2. 通过接口 **runtime.GC() 主动触发**。

所有的触发都会使用 `gcTrigger.test()` 进行条件检测（runtime/mgc.go）：

```go
// test reports whether the trigger condition is satisfied, meaning
// that the exit condition for the _GCoff phase has been met. The exit
// condition should be tested when allocating.
func (t gcTrigger) test() bool {
        if !memstats.enablegc || panicking != 0 || gcphase != _GCoff {
                return false
        }
        switch t.kind {
        case gcTriggerHeap:
                // 由 heap 触发，也就是分配 object 时触发
                // Non-atomic access to heap_live for performance. If
                // we are going to trigger on this, this thread just
                // atomically wrote heap_live anyway and we'll see our
                // own write.
                return memstats.heap_live >= memstats.gc_trigger
        case gcTriggerTime:
                // 由 sysmon 周期性触发
                if gcpercent < 0 {
                        return false
                }
                lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
                return lastgc != 0 && t.now-lastgc > forcegcperiod
        case gcTriggerCycle:
                // 通过 runtime.GC() 主动触发
                // t.n > work.cycles, but accounting for wraparound.
                return int32(t.n-work.cycles) > 0
        }
        return true
}
```
对应的条件为：
1. sysmon 周期性触发：**触发间隔大于 2min**；
2. 分配 object 触发：**堆内存的分配达到控制计算的触发堆大小**；
3. runtime.GC() **主动触发**：当前没有正在 GC，则触发；

### 4.2 GC 开始
所有触发 GC 后调用的都是 **`gcStart()`** 函数（runtime/mgc.go）：
```go
// gcStart starts the GC. It transitions from _GCoff to _GCmark (if
// debug.gcstoptheworld == 0) or performs all of GC (if
// debug.gcstoptheworld != 0).
//
// This may return without performing this transition in some cases,
// such as when called on a system stack or with locks held.
func gcStart(trigger gcTrigger) {
  // Since this is called from malloc and malloc is called in
  // the guts of a number of libraries that might be holding
  // locks, don't attempt to start GC in non-preemptible or
  // potentially unstable situations.
  mp := acquirem()
  if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
    releasem(mp)
    return
  }
  releasem(mp)
  mp = nil

  // 再次验证是否条件，并不断调用 sweepone() 来完成上一次垃圾收集的收尾工作
  // Pick up the remaining unswept/not being swept spans concurrently
  //
  // This shouldn't happen if we're being invoked in background
  // mode since proportional sweep should have just finished
  // sweeping everything, but rounding errors, etc, may leave a
  // few spans unswept. In forced mode, this is necessary since
  // GC can be forced at any point in the sweeping cycle.
  //
  // We check the transition condition continuously here in case
  // this G gets delayed in to the next GC cycle.
  for trigger.test() && sweepone() != ^uintptr(0) {
    sweep.nbgsweep++
  }

  // Perform GC initialization and the sweep termination
  // transition.
  semacquire(&work.startSema)
  // Re-check transition condition under transition lock.
  if !trigger.test() {
    semrelease(&work.startSema)
    return
  }

  // 获取 STW 的锁
  // Ok, we're doing it! Stop everybody else
  semacquire(&gcsema)
  semacquire(&worldsema)

  // 每个 P 分配一个 G，准备开始执行后台的标记工作
  gcBgMarkStartWorkers()

  // 执行 STW !
  systemstack(stopTheWorldWithSema)
  // Finish sweep before we start concurrent scan.
  systemstack(func() {
    finishsweep_m()
  })

  // 修改 GC 状态，进入标记
  setGCPhase(_GCmark)

  // 初始化标记所需状态
  gcBgMarkPrepare() 
  // 计算 Data、BSS、Stack 等需要扫描的数量
  gcMarkRootPrepare()

  // 直接标记 tiny object
  gcMarkTinyAllocs()

  // 可以开始运行标记
  atomic.Store(&gcBlackenEnabled, 1)
          
  // STW 结束，G 开始进行并行标记
  // Concurrent mark.
  systemstack(func() {
    now = startTheWorldWithSema(trace.enabled)
    work.pauseNS += now - work.pauseStart
    work.tMark = now
  })

  // 释放锁
  semrelease(&worldsema)
  releasem(mp)
     
	if mode != gcBackgroundMode {
		Gosched()
	}
	semrelease(&work.startSema)
}
```
该函数比较复杂，大致分为下面几个步骤：
1. 主动进入休眠状态，并**等待唤醒**；
1. 根据 P.gcMarkWorkerMode **决定标记的策略**；
1. 调用 [**gcDrain()**](#212-groutine-标记流程) 进行标记
1. 所有标记任务完成后，调用 `gcMarkDone()` 完成标记阶段；

因为标记阶段是与用户进程并发的，所以会涉及到执行垃圾收集还是普通程序的问题。为此，每个垃圾收集的 G 有着不同的标记策略，其依赖于 P.gcMarkWorkerMode（由一个独立的 G 计算出不同模式的 P 的数量并设置）。

其包含三种标记策略：
* **gcMarkWorkerDedicatedMode**：P 专门用于标记对象，不会被抢占；
* **gcMarkWorkerFractionalMode**：当垃圾收集后台 CPU 使用率达不到 25%，会启动该类型工作协程帮助垃圾收集达到利用率目标，因为只占用一个 CPU 部分资源，可以被抢占；
* **gcMarkWorkerIdleMode**：当 P 没有可以执行的 G 时，会运行垃圾收集标记任务直到被抢占；

### 4.3 标记结束
当每个标记 Groutine 结束后，都会调用 `gcMarkDone()`，但是等待所有标记结束后，**只有一个 groutine 会真正执行结束逻辑**（runtime/mgc.go）。
```go
func gcMarkDone() {
	// 第一个获取到锁的才会执行 gcMarkDone()
	// Ensure only one thread is running the ragged barrier at a
	// time.
	semacquire(&work.markDoneSema)

top:
	// 后续的 gouroute 会串行的在这里退出
	if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
		semrelease(&work.markDoneSema)
		return
	}
	
	// 循环等待所有标记结束
	gcMarkDoneFlushed = 0
	systemstack(func() {
		gp := getg().m.curg
		casgstatus(gp, _Grunning, _Gwaiting)
		forEachP(func(_p_ *p) {
			wbBufFlush1(_p_)
			_p_.gcw.dispose()
			if _p_.gcw.flushedWork {
				atomic.Xadd(&gcMarkDoneFlushed, 1)
				_p_.gcw.flushedWork = false
			}
		})
		casgstatus(gp, _Gwaiting, _Grunning)
	})

	if gcMarkDoneFlushed != 0 {
		goto top
	}
	…
	
	// Perform mark termination. This will restart the world.
	gcMarkTermination(nextTriggerRatio)
}
```

在一大堆判断标记结束的逻辑后，调用 **`gcMarkTermination()`** 进入标记终止阶段。

在 `gcMarkTermination()` 会关闭混合写屏障，决定触发垃圾收集的 heap 阈值，并进行相关信息的统计，然后调用 [**gcSweep()**](#3-内存清理) 进行阻塞式清理。

## 总结
垃圾回收真的很复杂，上面省略了大量的细节，也有可能理解错误的情况。但是忽略掉繁琐的细节，需要完全明白的有几个点：
* [****三色标记算法的步骤****](#12-三色标记算法)
* [****强三色不变性概念，以及为什么需要****](#三色不变性)
* [****写屏障的作用，以及 Go 使用的混合写屏障****](#13-屏障技术)
* [****GC 触发的时机与条件****](#41-gc-触发)
* [****GC 并发标记的实现****](#211-并发标记框架)
* [****GC 标记的实现，与内存管理的协同****](#212-groutine-标记流程)
* [****内存清理的时机****](#3-内存清理)

## 参考
* [《Golang 学习笔记》](https://github.com/qyuhen/book)
* [《Golang 设计与实现》：内存分配器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/#72-%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8)
