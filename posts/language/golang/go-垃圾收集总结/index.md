# Go 垃圾收集总结


## 1 背景知识

### 1.1 术语
垃圾回收算法有一些基本的术语，首先需要知道对应的含义：
* **`Mutator`**：具有“改变对象”的意思，GC 中就是改动对象间引用关系的意思，也就是程序；
* **`堆`**：对象使用的内存空间，GC 就是将垃圾对象空间放回到堆中；
* **`根对象`**：对象的指针的“起点”部分，一般就是全局对象和栈对象；

### 1.2 三色标记算法
三色标记算法是 GC [标记清除算法]^(Mark-Sweep) 的一种，也是 Golang 中使用的算法。
{{< admonition note Note>}}
推荐阅读：
{{< /admonition >}}

首先，三色标记算法的最基本逻辑为：
1. 标记的最开始，**所有对象默认为`白色`**；
1. 将 **`根对象`标记为灰色**，放入灰色集合；
1. 从`灰色集合`中取出灰色对象，将其**子对象标记为`灰色`**，加入灰色集合，该灰色对象标记为黑色；
1. 重复第 3 步，直到灰色集合为空；
1. **清理所有的白色对象**；

整个逻辑很简单，就是一个树的层次遍历，将所有可达的结点标记，然后清理未标记的不可达结点。
{{< find_img "img1.png" >}}

但是，仅仅是普通的三色标记算法要求执行时，Mutator 不能同时运行。因为如果 Mutator 并行时，某个扫描过的结点的引用关系变化，就可能导致[悬挂指针]^(dangling pointer)问题。

例如，上图中第 3 步将 A 指向 D，那么 D 还是无法被标记，被错误回收。

而想要让 Mutator 同时运行时，标记的结果还保持正确，那么每个时刻标记的结果要满足[三色不变性]^(Tri-color invariant)
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

### 1.3.1 插入写屏障
Dijkstra 提出的 **`插入写屏障`**，在更新对象指针时，将其被指向的对象重新加入扫描集合（三色标记中也就是变为灰色），这样接下来还是能够被扫描。
{{< find_img "img2.png" >}}

可以看到，这样黑色对象始终指向的是灰色对象，永远都会满足强三色不变性。

但是，Dijkstra 也有一些缺点：
* 对象指针变动时，没有考虑旧的指针引用。例如 `*field` 原来的对象 oldobj 已经扫描成黑色了，那么 `*field = newobj` 变动后，可能 oldobj 变为垃圾对象，只有等到下一轮标记时才会被回收。
* TODO

### 1.3.2 删除写屏障
Yuasa 提出的 **`删除写屏障`**，让老对象的引用被删除时，将白色的老对象涂成灰色，这样删除写屏障就可以保证弱三色不变性。
{{< find_img "img3.png" >}}

### 1.3.3 Go 中的屏障
Go 中使用 **`混合写屏障`**，即插入写屏障与删除写屏障都开启，并且在标记阶段开始后，将创建的所有新对象都标记为黑色，防止新分配的对象被错误的回收。

具体操作为：
1. GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)，
1. GC期间，任何在栈上创建的新对象，均为黑色。
1. 被删除的对象标记为灰色。
1. 被添加的对象标记为灰色。



## Go 中的三色标记算法
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

每个 P 会对应一个标记使用的 groutine，执行 `gcDrain()` 函数（runtime/mgcmark.go）：
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
在 [**Go 内存管理总结**]() 中说过，**任意的内存地址，都可以通过公式得到其对应的 mspan 的地址**，这也可以用于判断一个地址是否是存在与 mheap 上的。
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
        }

        n = s.base() + s.elemsize - b
        if n > maxObletBytes {
                n = maxObletBytes
        }
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
        }
        gcw.bytesMarked += uint64(n)
        gcw.scanWork += int64(i)
}
```
可以看到，通过当前 object 地址的**不断偏移 8 字节**，然后**通过 `heapArena.bitmap` 判断当前 8 字节是否是指针**，如果是指针就将其放入 gcWork 对象。
{{< admonition note Note>}}
在 [**Go 内存管理总结**]() 中看到，**对于 heap 中每 8 个字节，通过存在对应 bit 位标识是否指针，以及是否被 mark**；
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
1. GC 开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)。
1. GC 期间，任何在栈上创建的新对象，均为黑色。
1. 被删除的对象标记为灰色。
1. 被添加的对象标记为灰色。

当开始 GC 时，全局变量 `runtime.writeBarrier.enabled` 变为 true，所有的写操作都会经过 `writebarrier()` 的操作。
