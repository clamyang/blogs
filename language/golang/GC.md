---
title: GC 源码梳理
comment: true
---

Golang GC 垃圾回收知识点总结，有些内容还没有完成，只是暂时写上了问题，还没来得及梳理相应的内容。

<!--more-->

## GC 连连问

### GC 的流程阶段

- sweep termination
- mark
- mark termination
- sweep

注：在下一次 GC mark 阶段开始之前，一定要完成上一次的清扫工作。所以我们看到了一个 sweep termination  阶段。根本原因是 Golang 中采用的是惰性清扫的。

### GC 的触发时机

1.runtime.GC 手动触发

2.mallocgc 根据内存分配大小，比如当前使用 4M，当内存分配到达 8M 时，会触发 GC，这个百分比是可以调整的，通过 设置 triggerRatio 指定触发 GC 的阈值 go1.16.5 中，是 7/8.0 = 87.5% 

3.forcegchelper 定时触发 GC

mallocgc 是主要的触发函数。

### GC 的起点

gcStart，控制 worker 的数量，占 CPU 的 1/4。

### 为什么老版本需要重新扫描栈？

### GC 标记的根都有什么？

~~~go
// gcMarkRootPrepare queues root scanning jobs (stacks, globals, and
// some miscellany) and initializes scanning-related state.
//
// The world must be stopped.
/*
	写在最前边，我们关注的重点只是 GC，
	不要过多的被其他知识点所蒙蔽，
	比如这里，我们只需要知道，根从哪里来，根都包含什么
	其它的都可以忽略．．
*/
func gcMarkRootPrepare() {
    assertWorldStopped()

    // Compute how many data and BSS root blocks there are.
    nBlocks := func(bytes uintptr) int {
        return int(divRoundUp(bytes, rootBlockBytes))
    }

    // 初始化需要被扫描的 data、bss 段个数
    work.nDataRoots = 0
    work.nBSSRoots = 0
    
    /*
    	暂时记录下目前的理解：
    	这里的 bss data 段，有可能会变得，比如说进行动态链接的时候，
    	就会把那个被链接的文件加入到 activeModules 中，所以都是通过
    	函数调用的方式来获取对应的数据
    	注：他这个对总数的赋值操作有点迷惑，不知道为啥要这样写..
    */

    // Scan globals.
    for _, datap := range activeModules() {
        nDataRoots := nBlocks(datap.edata - datap.data)
        if nDataRoots > work.nDataRoots {
            work.nDataRoots = nDataRoots
        }
    }

    for _, datap := range activeModules() {
        nBSSRoots := nBlocks(datap.ebss - datap.bss)
        if nBSSRoots > work.nBSSRoots {
            work.nBSSRoots = nBSSRoots
        }
    }

    // Scan span roots for finalizer specials.
    //
    // We depend on addfinalizer to mark objects that get
    // finalizers after root marking.
    //
    // We're going to scan the whole heap (that was available at the time the
    // mark phase started, i.e. markArenas) for in-use spans which have specials.
    //
    // Break up the work into arenas, and further into chunks.
    //
    // Snapshot allArenas as markArenas. This snapshot is safe because allArenas
    // is append-only.
    // 扫描整个 heap，对 allArenas 做快照
    mheap_.markArenas = mheap_.allArenas[:len(mheap_.allArenas):len(mheap_.allArenas)]
    // 计算需要扫描的 span 数量，arena * （单个 arena 中 span 的数量）
    work.nSpanRoots = len(mheap_.markArenas) * (pagesPerArena / pagesPerSpanRoot)

    // Scan stacks.
    //
    // Gs may be created after this point, but it's okay that we
    // ignore them because they begin life without any roots, so
    // there's nothing to scan, and any roots they create during
    // the concurrent phase will be caught by the write barrier.
    // 如注释所说，尽管这时候有 goroutine 被创建，也不需要担心
    // 因为他们没有 root，即不需要扫描。如果在并发阶段创建出来的 goroutine，
    // 这个 G 使用的 root 会被 write barrier 捕获到。
    work.nStackRoots = int(atomic.Loaduintptr(&allglen))

    // 初始化标记的开始位置
    work.markrootNext = 0
    //　计算所有根的数量
    //　包括了：Data　段，BSS　段,span,以及 goroutine 栈
    work.markrootJobs = uint32(fixedRootCount + work.nDataRoots + work.nBSSRoots + work.nSpanRoots + work.nStackRoots)

    // Calculate base indexes of each root type
    // markroot 标记的时候会根据不同的 i 找到不同的根
    work.baseData = uint32(fixedRootCount)
    work.baseBSS = work.baseData + uint32(work.nDataRoots)
    work.baseSpans = work.baseBSS + uint32(work.nBSSRoots)
    work.baseStacks = work.baseSpans + uint32(work.nSpanRoots)
    work.baseEnd = work.baseStacks + uint32(work.nStackRoots)
}
~~~

- 结合上边代码来看，GC 标记过程中涉及到的根有：全局变量，stack，heap（span）

### 单独使用 D 屏障有何问题？

### 单独使用 Y 屏障有何问题？

### 混合写屏障是怎么一回事？

### 协助标记流程

这个涉及到 credit 的机制，一般都是 malloc 中 credit 不够了，才进行的。

### 对象的交叉引用是如何剪枝的？

通过原子操作 `atomic.Or8` 避免了重复标记

#### 与运算

两个都是一才是一

#### 或运算

有一个是一就是一

#### 异或运算

同零异一

### GC 的 CPU 使用率

- GC cpu 使用率主要用来控制，启动 mark worker 的数量

mark worker count = gomaxprocs * 25%

如果 gomaxprocs = 4，那么只需要启动一个 markworker

对于计算结果不为整数的情况，比如 gomaxprocs = 6，那么 work count  = 1.5，会对结果 + 0.5 进行 rounding， 然后通过计算误差是否 > 0.3 判断开几个全职 worker

2/1.5 -1 = 1/3 > 0.3

### gcTriggerKind

gc 触发类型：

- gcTriggerHeap 内存到达阈值
- gcTriggerTime 到达触发时间
- gcTriggerCycle 可以理解为用户手动触发类型

### STW 时间怎么算出来的？

### GC 过程中是什么时候将对象标记为黑色的

换句话说，是通过修改了什么变量，就代表这个指针被标记为黑色。

解释如下：

​	通过源码分析，我们可以知道，从白色对象到灰色对象是通过 greyobject 来实现的，同事也能够知道在 gcw 队列中的对象都是灰色的。

​	标记为黑色的过程是从 gcw 队列中取出灰色对象，再遍历其子对象并将其标灰，也即是说当灰色对象出队的时候就自动变成黑色了，就完成了将一个灰色对象标记位黑色的过程，在 Go 的源码中，其实并不存在的某个方法或者某个标志位，来表示一个对象是黑色的。

## gcStart 源码剖析

~~~go
// gcStart starts the GC. It transitions from _GCoff to _GCmark (if
// debug.gcstoptheworld == 0) or performs all of GC (if
// debug.gcstoptheworld != 0).
//
// This may return without performing this transition in some cases,
// such as when called on a system stack or with locks held.

// gcStart 是 GC 的起点，并将 GC 的状态由 _GCoff 切换到 _GCmark，
// 如果 gcStart 在系统栈上被调用或者持有锁的时候，就不会执行状态的改变直接返回了。
func gcStart(trigger gcTrigger) {
    // Since this is called from malloc and malloc is called in
    // the guts of a number of libraries that might be holding
    // locks, don't attempt to start GC in non-preemptible or
    // potentially unstable situations.

    // 如果 gcStart 是从 malloc 调用的，并且 malloc 又是被其他的库调用的，
    // 这种情况下可能会持有锁（mp.locks > 1）
    // 不要再非抢占或者不稳定的情况下调用 gcStart
    /*
       我的理解：如果不是非抢占模式，可能会导致后面 stw 时，一些 g 没有办法停止
       会影响 GC 的结果。
    */
    mp := acquirem()
    // mp.locks > 1 说明在 gcStart 之前就持有锁
    // mp.preemptoff != "" 说明在 non-preempt 模式下
    if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
        releasem(mp)
        return
    }
    releasem(mp)
    mp = nil	

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
    /*
        trigger.test() 检测是否满足 GC 的触发条件
        sweepone() 我的理解是：清扫上次 GC 遗留下来的 unswept 的 span
        ？？ 是否可以理解成 sweep termination 的阶段 
    */
    for trigger.test() && sweepone() != ^uintptr(0) {
        sweep.nbgsweep++
    }

    // Perform GC initialization and the sweep termination
    // transition.
    /*
    	semaacquire 的操作，是否可以理解为，通过 atomic 去掉了锁。
		换句话说，只有获取了某个 sema，才能对 gc 状态进行修改。
    */
    semacquire(&work.startSema)
    // Re-check transition condition under transition lock.
    if !trigger.test() {
        semrelease(&work.startSema)
        return
    }

    // For stats, check if this GC was forced by the user.
    work.userForced = trigger.kind == gcTriggerCycle

    // In gcstoptheworld debug mode, upgrade the mode accordingly.
    // We do this after re-checking the transition condition so
    // that multiple goroutines that detect the heap trigger don't
    // start multiple STW GCs.
    // 如果没开启 GODEBUG 都是 gcBackgroundMode 模式
    mode := gcBackgroundMode
    if debug.gcstoptheworld == 1 {
        mode = gcForceMode
    } else if debug.gcstoptheworld == 2 {
        mode = gcForceBlockMode
    }

    // Ok, we're doing it! Stop everybody else
    // 获取 STW 需要的 semaphore
    semacquire(&gcsema)
    semacquire(&worldsema)

    if trace.enabled {
        traceGCStart()
    }

    // Check that all Ps have finished deferred mcache flushes.
    // TODO 检查 P 的 mcache
    for _, p := range allp {
        if fg := atomic.Load(&p.mcache.flushGen); fg != mheap_.sweepgen {
            println("runtime: p", p.id, "flushGen", fg, "!= sweepgen", mheap_.sweepgen)
            throw("p mcache not flushed")
        }
    }

    // 开启 nproc 个 gcMarkWorker，加入到 workerPool 中
    // 创建完一个 worker，休眠一个 worker
    // 由 schedule.findRunnableGCWorker 唤醒
    gcBgMarkStartWorkers()

    // 重置标志位，allg 的标志位，heapArena 的标志位
    // 清空了每一个 g 的 AssistBytes
    systemstack(gcResetMarkState)

    work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
    if work.stwprocs > ncpu {
        // This is used to compute CPU time of the STW phases,
        // so it can't be more than ncpu, even if GOMAXPROCS is.
        work.stwprocs = ncpu
    }
    work.heap0 = atomic.Load64(&memstats.heap_live)
    work.pauseNS = 0
    work.mode = mode

    now := nanotime()
    work.tSweepTerm = now
    work.pauseStart = now
    if trace.enabled {
        traceGCSTWStart(1)
    }

    // STW 调用者必须为 stopTheWorldWithSema 获取 worldsema 并且 关闭抢占
    systemstack(stopTheWorldWithSema)

    // Finish sweep before we start concurrent scan.
    // 确保本次 GC 开始时，已完成上一次 GC 的 sweep 工作
    systemstack(func() {
        finishsweep_m()
    })

    // clearpools before we start the GC. If we wait they memory will not be
    // reclaimed until the next GC cycle.
    // 1.处理 sync.Pool
    // 2.清空 sudog cache
    // 3.清空 defer pools
    clearpools()

    // 增加 gc 周期数
    work.cycles++

    // 开始本次 gc 周期
    gcController.startCycle()
    work.heapGoal = memstats.next_gc

    // In STW mode, disable scheduling of user Gs. This may also
    // disable scheduling of this goroutine, so it may block as
    // soon as we start the world again.
    if mode != gcBackgroundMode {
        schedEnableUser(false)
    }

    // Enter concurrent mark phase and enable
    // write barriers.
    //
    // Because the world is stopped, all Ps will
    // observe that write barriers are enabled by
    // the time we start the world and begin
    // scanning.
    //
    // Write barriers must be enabled before assists are
    // enabled because they must be enabled before
    // any non-leaf heap objects are marked. Since
    // allocations are blocked until assists can
    // happen, we want enable assists as early as
    // possible.
    setGCPhase(_GCmark)

    gcBgMarkPrepare() // Must happen before assist enable.
    gcMarkRootPrepare()

    // Mark all active tinyalloc blocks. Since we're
    // allocating from these, they need to be black like
    // other allocations. The alternative is to blacken
    // the tiny block on every allocation from it, which
    // would slow down the tiny allocator.
    gcMarkTinyAllocs()

    // At this point all Ps have enabled the write
    // barrier, thus maintaining the no white to
    // black invariant. Enable mutator assists to
    // put back-pressure on fast allocating
    // mutators.
    atomic.Store(&gcBlackenEnabled, 1)

    // Assists and workers can start the moment we start
    // the world.
    gcController.markStartTime = now

    // In STW mode, we could block the instant systemstack
    // returns, so make sure we're not preemptible.
    mp = acquirem()

    // Concurrent mark.
    systemstack(func() {
        now = startTheWorldWithSema(trace.enabled)
        work.pauseNS += now - work.pauseStart
        work.tMark = now
        memstats.gcPauseDist.record(now - work.pauseStart)
    })

    // Release the world sema before Gosched() in STW mode
    // because we will need to reacquire it later but before
    // this goroutine becomes runnable again, and we could
    // self-deadlock otherwise.
    semrelease(&worldsema)
    releasem(mp)

    // Make sure we block instead of returning to user code
    // in STW mode.
    if mode != gcBackgroundMode {
        Gosched()
    }

    semrelease(&work.startSema)
}

~~~

## findRunnableGCWorker 源码剖析

~~~go
// findRunnableGCWorker returns a background mark worker for _p_ if it
// should be run. This must only be called when gcBlackenEnabled != 0.
// 只有在 gc 开启的时候才会执行，gcStart 中设置为 1
func (c *gcControllerState) findRunnableGCWorker(_p_ *p) *g {
    if gcBlackenEnabled == 0 {
        throw("gcControllerState.findRunnable: blackening not enabled")
    }

    if !gcMarkWorkAvailable(_p_) {
        // No work to be done right now. This can happen at
        // the end of the mark phase when there are still
        // assists tapering off. Don't bother running a worker
        // now because it'll just return immediately.
        return nil
    }

    // Grab a worker before we commit to running below.
    // 从 workpool 中弹出一个 gcMarkWorker
    node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
    if node == nil {
        // There is at least one worker per P, so normally there are
        // enough workers to run on all Ps, if necessary. However, once
        // a worker enters gcMarkDone it may park without rejoining the
        // pool, thus freeing a P with no corresponding worker.
        // gcMarkDone never depends on another worker doing work, so it
        // is safe to simply do nothing here.
        //
        // If gcMarkDone bails out without completing the mark phase,
        // it will always do so with queued global work. Thus, that P
        // will be immediately eligible to re-run the worker G it was
        // just using, ensuring work can complete.
        return nil
    }

    // 用于计算该 markNode 的工作模式
    // >  0 dedicatedMode
    // <= 0 fractionalMode
    decIfPositive := func(ptr *int64) bool {
        for {
            v := atomic.Loadint64(ptr)
            if v <= 0 {
                return false
            }

            if atomic.Casint64(ptr, v, v-1) {
                return true
            }
        }
    }

    if decIfPositive(&c.dedicatedMarkWorkersNeeded) {
        // This P is now dedicated to marking until the end of
        // the concurrent mark phase.
        _p_.gcMarkWorkerMode = gcMarkWorkerDedicatedMode
    } else if c.fractionalUtilizationGoal == 0 {
        // No need for fractional workers.
        gcBgMarkWorkerPool.push(&node.node)
        return nil
    } else {
        // Is this P behind on the fractional utilization
        // goal?
        //
        // This should be kept in sync with pollFractionalWorkerExit.
        delta := nanotime() - c.markStartTime
        if delta > 0 && float64(_p_.gcFractionalMarkTime)/float64(delta) > c.fractionalUtilizationGoal {
            // Nope. No need to run a fractional worker.
            gcBgMarkWorkerPool.push(&node.node)
            return nil
        }
        // Run a fractional worker.
        _p_.gcMarkWorkerMode = gcMarkWorkerFractionalMode
    }

    // Run the background mark worker.
    // 修改 workerg 的状态并返回
    gp := node.gp.ptr()
    casgstatus(gp, _Gwaiting, _Grunnable)
    if trace.enabled {
        traceGoUnpark(gp, 0)
    }
    return gp
}


// gcMarkWorkAvailable reports whether executing a mark worker
// on p is potentially useful. p may be nil, in which case it only
// checks the global sources of work.
/*
	该函数主要作用：返回是否有标记工作可以干
	gcw p的队列
	work 全局队列
*/
func gcMarkWorkAvailable(p *p) bool {
    if p != nil && !p.gcw.empty() {
        return true
    }
    if !work.full.empty() {
        return true // global work available
    }
    if work.markrootNext < work.markrootJobs {
        return true // root scan work available
    }
    return false
}
~~~

## 剖析 gcBgMarkWorker 创建过程

~~~go
// gcBgMarkStartWorkers prepares background mark worker goroutines. These
// goroutines will not run until the mark phase, but they must be started while
// the work is not stopped and from a regular G stack. The caller must hold
// worldsema.
func gcBgMarkStartWorkers() {
    // Background marking is performed by per-P G's. Ensure that each P has
    // a background GC G.
    //
    // Worker Gs don't exit if gomaxprocs is reduced. If it is raised
    // again, we can reuse the old workers; no need to create new workers.
    for gcBgMarkWorkerCount < gomaxprocs {
        go gcBgMarkWorker()

        notetsleepg(&work.bgMarkReady, -1)
        noteclear(&work.bgMarkReady)
        // The worker is now guaranteed to be added to the pool before
        // its P's next findRunnableGCWorker.

        gcBgMarkWorkerCount++
    }
}
~~~

- 由上可知，所有 gcMarkWorker 都是通过该函数创建的，我们需要关注的是 `notesleepg()` 和 `go gcBgMarkWorker` 这两行代码

### `notesleepg`

~~~go
// same as runtime·notetsleep, but called on user g (not g0)
// calls only nosplit functions between entersyscallblock/exitsyscall
func notetsleepg(n *note, ns int64) bool {
    gp := getg()
    if gp == gp.m.g0 {
        throw("notetsleepg on g0")
    }
    // 这个我们在学习 timer 的时候已经看过了
    // 主要就是执行 handoff
    entersyscallblock()
    // 这里才是真正进行系统调用的地方，ns = -1，会无休止的休眠
    // 直到通过 wakeup 唤醒
    ok := notetsleep_internal(n, ns)
    // 上述，通过 wakeup 唤醒后会继续执行这块代码
    // 该函数主要是给刚刚剥离的 GM 找一个 P
    // 找到了，执行
    // 没找到，把 g 放到全局队列
    exitsyscall()
    return ok
}
~~~

### `go gcBgMarkWorker`

- 这里应用到 MPG 调度的知识点，我们通过 go 关键字新建一个 goroutine，放入 runnext 等待被调度。

### 结合着看

我们假设现在只有一个 P，即 `runtime.GOMAXPROCS(1)` 。这时我们来看上述代码，其执行过程为：

- 1.通过 `gcBgMarkStartWorkers` 创建 worker
- 2.worker 被放入 runnext 等待被调度
- 3.执行 `notesleepg`
	- handoffp
	- futex && timeout = -1
	- 第 3 步执行完成后，GM 已经从 P 上剥离
- 4.handoffp 中会启动一个 M 继续执行调度循环
- 5.newM 从 P 上找 G 执行
- 6.拿到我们刚刚创建的 newg
- 7.进入到 `gcBgMarkWorker()` 中执行
	- 新建 node && **wakeup** 休眠的 GM
	- 然后 `gopark` 挂起 worker 等待唤醒
- 8.休眠的 GM 醒来后
	- 尝试获取 oldp
	- 如果获取不到则，尝试获取 idlep
	- 如果获取不到则，将 g 放入全局队列
- 9.第 8 步中的 g 被调度后，会继续执行 create worker 的工作，回到第 1 步 继续执行

## gcMarkWorker 执行过程

### gcBgMarkWorker 源码剖析

~~~go
func gcBgMarkWorker() {
    gp := getg()

    // We pass node to a gopark unlock function, so it can't be on
    // the stack (see gopark). Prevent deadlock from recursively
    // starting GC by disabling preemption.
    gp.m.preemptoff = "GC worker init"
    node := new(gcBgMarkWorkerNode)
    gp.m.preemptoff = ""

    node.gp.set(gp)

    node.m.set(acquirem())
    notewakeup(&work.bgMarkReady)
    // After this point, the background mark worker is generally scheduled
    // cooperatively by gcController.findRunnableGCWorker. While performing
    // work on the P, preemption is disabled because we are working on
    // P-local work buffers. When the preempt flag is set, this puts itself
    // into _Gwaiting to be woken up by gcController.findRunnableGCWorker
    // at the appropriate time.
    //
    // When preemption is enabled (e.g., while in gcMarkDone), this worker
    // may be preempted and schedule as a _Grunnable G from a runq. That is
    // fine; it will eventually gopark again for further scheduling via
    // findRunnableGCWorker.
    //
    // Since we disable preemption before notifying bgMarkReady, we
    // guarantee that this G will be in the worker pool for the next
    // findRunnableGCWorker. This isn't strictly necessary, but it reduces
    // latency between _GCmark starting and the workers starting.

    for {
        // Go to sleep until woken by
        // gcController.findRunnableGCWorker.
        gopark(func(g *g, nodep unsafe.Pointer) bool {
            node := (*gcBgMarkWorkerNode)(nodep)

            if mp := node.m.ptr(); mp != nil {
                // The worker G is no longer running; release
                // the M.
                //
                // N.B. it is _safe_ to release the M as soon
                // as we are no longer performing P-local mark
                // work.
                //
                // However, since we cooperatively stop work
                // when gp.preempt is set, if we releasem in
                // the loop then the following call to gopark
                // would immediately preempt the G. This is
                // also safe, but inefficient: the G must
                // schedule again only to enter gopark and park
                // again. Thus, we defer the release until
                // after parking the G.
                releasem(mp)
            }

            // Release this G to the pool.
            gcBgMarkWorkerPool.push(&node.node)
            // Note that at this point, the G may immediately be
            // rescheduled and may be running.
            return true
        }, unsafe.Pointer(node), waitReasonGCWorkerIdle, traceEvGoBlock, 0)

        /*
        	以上代码都已经解释过了
        */

        /*
        	TODO 关于在 GC 期间什么时候可以抢占，什么时候禁止抢占
        	还需要进一步研究，目前可以先把整个流程梳理下来
        */

        // Preemption must not occur here, or another G might see
        // p.gcMarkWorkerMode.

        // Disable preemption so we can use the gcw. If the
        // scheduler wants to preempt us, we'll stop draining,
        // dispose the gcw, and then preempt.
        node.m.set(acquirem())
        pp := gp.m.p.ptr() // P can't change with preemption disabled.

        // 是否已开启标记
        if gcBlackenEnabled == 0 {
            println("worker mode", pp.gcMarkWorkerMode)
            throw("gcBgMarkWorker: blackening not enabled")
        }

        // 检查 markworker 的 Mode
        if pp.gcMarkWorkerMode == gcMarkWorkerNotWorker {
            throw("gcBgMarkWorker: mode not set")
        }

        // 记录 worker 标记的开始时间
        startTime := nanotime()
        pp.gcMarkWorkerStartTime = startTime

        // 将等待执行的 worker 数量减一
        decnwait := atomic.Xadd(&work.nwait, -1)
        if decnwait == work.nproc {
            println("runtime: work.nwait=", decnwait, "work.nproc=", work.nproc)
            throw("work.nwait was > work.nproc")
        }

        systemstack(func() {
            // Mark our goroutine preemptible so its stack
            // can be scanned. This lets two mark workers
            // scan each other (otherwise, they would
            // deadlock). We must not modify anything on
            // the G stack. However, stack shrinking is
            // disabled for mark workers, so it is safe to
            // read from the G stack.
            // 关于这里为什么把 G 的 running 状态修改为 waiting 状态
            // TODO 需要进一步的研究，曹大说要结合 suspendG 来看
            casgstatus(gp, _Grunning, _Gwaiting)
            switch pp.gcMarkWorkerMode {
                default:
                	throw("gcBgMarkWorker: unexpected gcMarkWorkerMode")
                case gcMarkWorkerDedicatedMode:
                	gcDrain(&pp.gcw, gcDrainUntilPreempt|gcDrainFlushBgCredit)
                    if gp.preempt {
                        // We were preempted. This is
                        // a useful signal to kick
                        // everything out of the run
                        // queue so it can run
                        // somewhere else.
                        if drainQ, n := runqdrain(pp); n > 0 {
                            lock(&sched.lock)
                            globrunqputbatch(&drainQ, int32(n))
                            unlock(&sched.lock)
                        }
                    }
                    // Go back to draining, this time
                    // without preemption.
                    gcDrain(&pp.gcw, gcDrainFlushBgCredit)
                case gcMarkWorkerFractionalMode:
                	gcDrain(&pp.gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
                case gcMarkWorkerIdleMode:
                	gcDrain(&pp.gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
            }
            casgstatus(gp, _Gwaiting, _Grunning)
        })

        // Account for time.
        duration := nanotime() - startTime
        switch pp.gcMarkWorkerMode {
            case gcMarkWorkerDedicatedMode:
                atomic.Xaddint64(&gcController.dedicatedMarkTime, duration)
                atomic.Xaddint64(&gcController.dedicatedMarkWorkersNeeded, 1)
            case gcMarkWorkerFractionalMode:
                atomic.Xaddint64(&gcController.fractionalMarkTime, duration)
                atomic.Xaddint64(&pp.gcFractionalMarkTime, duration)
            case gcMarkWorkerIdleMode:
                atomic.Xaddint64(&gcController.idleMarkTime, duration)
        }

        // Was this the last worker and did we run out
        // of work?
        incnwait := atomic.Xadd(&work.nwait, +1)
        if incnwait > work.nproc {
            println("runtime: p.gcMarkWorkerMode=", pp.gcMarkWorkerMode,
                    "work.nwait=", incnwait, "work.nproc=", work.nproc)
            throw("work.nwait > work.nproc")
        }

        // We'll releasem after this point and thus this P may run
        // something else. We must clear the worker mode to avoid
        // attributing the mode to a different (non-worker) G in
        // traceGoStart.
        pp.gcMarkWorkerMode = gcMarkWorkerNotWorker

        // If this worker reached a background mark completion
        // point, signal the main GC goroutine.
        if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
            // We don't need the P-local buffers here, allow
            // preemption becuse we may schedule like a regular
            // goroutine in gcMarkDone (block on locks, etc).
            releasem(node.m.ptr())
            node.m.set(nil)

            gcMarkDone()
        }
    }
}
~~~

- gcMarkWorker 的状态（dedicated, fraction）这个状态是绑定在 P 上，并不是 G 的状态。

### GC 标记的几个工作模式

1. gcDrainFlushBgCredit: 把 bgMarkWorker 积累的 credit 刷新到全局的 gcController 中。
2. gcDrainFractional: self-preempt 表示在达到 fractional 后自动退出。
3. gcDrainUntilPreempt: 一直执行，直到被抢占。
4. gcDrainIdle: 一直执行，直到其他任务要做。

### bitmap 与 ha 的映射关系（标记过程用到的位图）

#### bitmap 的使用

两个比特表示一个字

ha 64 M，bitmap 2 M，bitmap 中的一个字节可以表示ha连续4个指针的内存大小。

0-4，1-5， 2-6， 3-7 

低位 bit 用于表示是否为指针，0 为非指针，1 为指针。

高位 bit 用于表示是否要继续扫描该对象中后续的内容，0 为不需要，1 为需要。

![img](https://blogstatic.haohtml.com/uploads/2021/04/d2b5ca33bd970f64a6301fa75ae2eb22-6.png?x-oss-process=image%2Fformat,webp)

#### heapBitsForAddr

```go
// heapBitsForAddr returns the heapBits for the address addr.
// The caller must ensure addr is in an allocated span.
// In particular, be careful not to point past the end of an object.
//
// nosplit because it is used during write barriers and must not be preempted.
//go:nosplit
func heapBitsForAddr(addr uintptr) (h heapBits) {
    // 2 bits per word, 4 pairs per byte, and a mask is hard coded.
    // 如注释所说，两个比特可以用来表示一个字的大小
    arena := arenaIndex(addr)
    ha := mheap_.arenas[arena.l1()][arena.l2()]
    // The compiler uses a load for nil checking ha, but in this
    // case we'll almost never hit that cache line again, so it
    // makes more sense to do a value check.
    if ha == nil {
        // addr is not in the heap. Return nil heapBits, which
        // we expect to crash in the caller.
        return
    }
    // 笔者理解：之所以要 *4，就是因为 bitmap 与 ha 的映射关系
    // 一字节可以表示堆上 4 个连续的指针内存
    h.bitp = &ha.bitmap[(addr/(sys.PtrSize*4))%heapArenaBitmapBytes]
    // 笔者理解：shift 就像掩码一样，用来计算低位与高位比特的位置
    // &3 的运算也能说明这一点，&3 的结果只能是 0 1 2 3，
    // 这里就对应上了 8 比特的低四位
    h.shift = uint32((addr / sys.PtrSize) & 3)
    // 记录当前 arena 的位置
    h.arena = uint32(arena)
    // 记录当前 arena 对应的 bitmap 中的最后一字节
    h.last = &ha.bitmap[len(ha.bitmap)-1]
    return
}
```

```go
// next returns the heapBits describing the next pointer-sized word in memory.
// That is, if h describes address p, h.next() describes p+ptrSize.
// Note that next does not modify h. The caller must record the result.
//
// 如注释所描述的，next 函数，返回描述了在内存中下一个指针类型的word的heapBits
// 如果 h 表示的是指针 p 的地址，那么 h.next() 表示的就是 p+ptrSize 的heapBits
// nosplit because it is used during write barriers and must not be preempted.
//go:nosplit
func (h heapBits) next() heapBits {
    if h.shift < 3*heapBitsShift { // 在同一字节上扫描，四个连续的ptr还没扫描完
        h.shift += heapBitsShift
    } else if h.bitp != h.last {	// 如果没扫描到该 bitmap 的最后一字节，那么扫描下一个byte
        h.bitp, h.shift = add1(h.bitp), 0
    } else {	// 此时，当前已经扫描完了当前 ha 的所有内容，移动到下一个 ha 进行扫描
        // Move to the next arena.
        return h.nextArena()
    }
    return h
}
```

### gcDrain 标记过程

```go
// gcDrain 扫描 work buffer 中的根节点和对象，
// gcDrain 会将队列中所有的对象标记为灰色。
// gcDrain 可能在 GC 结束前返回。
// gcDrain 的调用者负责平衡当前 P 与 其他 P 的标记工作。
//go:nowritebarrier
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
    if !writeBarrier.needed {
        throw("gcDrain phase incorrect")
    }

    gp := getg().m.curg
    preemptible := flags&gcDrainUntilPreempt != 0
    flushBgCredit := flags&gcDrainFlushBgCredit != 0
    idle := flags&gcDrainIdle != 0

    initScanWork := gcw.scanWork

    // checkWork is the scan work before performing the next
    // self-preempt check.
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

    // Drain root marking jobs.
    // 判断当前标记的位置是否超出了总的标记数量
    if work.markrootNext < work.markrootJobs {
        // Stop if we're preemptible or if someone wants to STW.
        for !(gp.preempt && (preemptible || atomic.Load(&sched.gcwaiting) != 0)) {
            job := atomic.Xadd(&work.markrootNext, +1) - 1
            if job >= work.markrootJobs {
                break
            }
            // 对 job 索引位置的 root 进行标记工作
            markroot(gcw, job)
            // 完成一个根的标记工作，就去检查
            // TODO 是否达到了 fractional 或者 其他
            if check != nil && check() {
                goto done
            }
        }
    }

    // Drain heap marking jobs.
    // Stop if we're preemptible or if someone wants to STW.
    for !(gp.preempt && (preemptible || atomic.Load(&sched.gcwaiting) != 0)) {
        // Try to keep work available on the global queue. We used to
        // check if there were waiting workers, but it's better to
        // just keep work available than to make workers wait. In the
        // worst case, we'll do O(log(_WorkbufSize)) unnecessary
        // balances.
        // 平衡标记工作，让它有活可干比让它等着要好
        if work.full == 0 {
            gcw.balance()
        }
		
        // 从 gcw 队列中取值
        b := gcw.tryGetFast()
        if b == 0 {
            b = gcw.tryGet()
            if b == 0 {
                // Flush the write barrier
                // buffer; this may create
                // more work.
                如果说 wbuf 中都没有，将
                wbBufFlush(nil, 0)
                b = gcw.tryGet()
            }
        }
        if b == 0 {
            // Unable to get work.
            break
        }
        scanobject(b, gcw)

        // Flush background scan work credit to the global
        // account if we've accumulated enough locally so
        // mutator assists can draw on it.
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
    // Flush remaining scan work credit.
    if gcw.scanWork > 0 {
        atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
        if flushBgCredit {
            gcFlushBgCredit(gcw.scanWork - initScanWork)
        }
        gcw.scanWork = 0
    }
}
```

#### 标记工作的平衡

##### balance

```go
// balance moves some work that's cached in this gcWork back on the
// global queue.
// 将缓存在 P gcw 上的标记任务，适当的移动到全局queue中
//go:nowritebarrierrec
func (w *gcWork) balance() {
    // 如果 P 的本地 wbf 没有标记任务，直接返回
    if w.wbuf1 == nil {
        return
    }
    /*
   	1.先处理 wbuf2 中的数据，如果有，将数据放入到
   	全局 work full 队列中
   	2.查看 wbuf1 中对象数量，超过四个，执行 handoff
   */
    if wbuf := w.wbuf2; wbuf.nobj != 0 {
        putfull(wbuf)
        w.flushedWork = true
        w.wbuf2 = getempty()
    } else if wbuf := w.wbuf1; wbuf.nobj > 4 {
        w.wbuf1 = handoff(wbuf)
        w.flushedWork = true // handoff did putfull
    } else {
        return
    }
    // We flushed a buffer to the full list, so wake a worker.
    if gcphase == _GCmark {
        gcController.enlistWorker()
    }
}
```

#####  enlistWorker

```go
// enlistWorker encourages another dedicated mark worker to start on
// another P if there are spare worker slots. It is used by putfull
// when more work is made available.
//
//go:nowritebarrier
func (c *gcControllerState) enlistWorker() {
    // If there are idle Ps, wake one so it will run an idle worker.
    // NOTE: This is suspected of causing deadlocks. See golang.org/issue/19112.
    //
    // if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
    //    wakep()
    //    return
    // }

    // There are no idle Ps. If we need more dedicated workers,
    // try to preempt a running P so it will switch to a worker.
    if c.dedicatedMarkWorkersNeeded <= 0 {
        return
    }
    // Pick a random other P to preempt.
    if gomaxprocs <= 1 {
        return
    }
    gp := getg()
    if gp == nil || gp.m == nil || gp.m.p == 0 {
        return
    }
    myID := gp.m.p.ptr().id
    for tries := 0; tries < 5; tries++ {
        id := int32(fastrandn(uint32(gomaxprocs - 1)))
        if id >= myID {
            id++
        }
        p := allp[id]
        if p.status != _Prunning {
            continue
        }
        // 执行抢占
        if preemptone(p) {
            return
        }
    }
}
```

​	不知道你们想没想过这样一个问题，通过 preemptone 函数进行抢占，是怎么做到去启动一个 gcMarkWorker 的？

​	如果你没想到的话，给你一点提示，和调度循环相关。

##### handoff

```go
// 在 wbuf 中对象数量大于 4 时，将前一半放入到全局 work full 队列中
//go:nowritebarrier
func handoff(b *workbuf) *workbuf {
    // Make new buffer with half of b's pointers.
    b1 := getempty()
    n := b.nobj / 2
    b.nobj -= n
    b1.nobj = n
    // 通过获取一个新的 empty workbuf
    // 将前一半放进去
    memmove(unsafe.Pointer(&b1.obj[0]), unsafe.Pointer(&b.obj[b.nobj]), uintptr(n)*unsafe.Sizeof(b1.obj[0]))

    // Put b on full list - let first half of b get stolen.
    putfull(b)
    return b1
}
```

#### GC 过程中 P 的 gcw 队列

```go
type P struct {
    (...)
    // gcw is this P's GC work buffer cache. The work buffer is
    // filled by write barriers, drained by mutator assists, and
    // disposed on certain GC state transitions.
    gcw gcWork
    (...)
}
```

以下是 Go 官方对 gcw 的描述：

为灰色指针对象实现了一个生产者/消费者模型。对象被标记为灰色后，放入到队列中。对象被标记为黑色后，会在队列中移除。

写屏障，根节点扫描，栈扫描和对象扫描过程会产生灰色对象。scanning 过程会消费灰色对象的指针并且有可能产生新的灰色对象指针。

gcWork 为垃圾回收器提供了一个生产和消费的接口。

可以在 stack 上这样使用 gcWork：

- 调用 gcw.Put() 进行生产，调用 gcw.tryGet() 进行消费。

重要的是，在 mark phase 使用 gcWork 可以阻止垃圾回收器从 transitioning 转变为 mark termination 因为 gcWork 可能在本地保存 GC work buffers。这可以通过禁用抢占来完成。

##### wbuf1 和 wbuf2

可以将 wbuf1 和 wbuf2 想象成一块栈空间，然后这俩轮流使用。

当我们弹出队列中最后一个指针时，我们通过引入一个新的缓冲区并丢弃一个空缓冲区（交换这两个缓冲区），然后将 stack 向上移动一个新 buffer。 当我们将buffers都填满了，我们通过引入一个新的空缓冲区并丢弃那个满的缓冲区，然后将stack向下移动一个新 buffer。

这样我们就有了一个缓冲区的滞后值，它可以将获取或放置工作缓冲区的成本分摊到至少一个工作缓冲区上，并减少全局工作列表上的争用。

wbuf1 永远都是我们正在操作的那个 buffer，wbuf2 是接下来要丢弃的缓冲区。

总结：

wbuf1 和 wbuf2 都是用来存储灰色指针的，wbuf1 是我们真正操作的那个队列，入队出队操作的都是它，wbuf2 起到的是替换作用，当 wbuf1 空了，替换 wbuf2，当 wbuf1 满了，替换 wbuf2。

##### work.full 和 work.empty

全局 work 中有两个队列，full 和 empty，灰色对象入队过程：

fastPath: 检查 wbuf1 是否为空，是否满了，是就返回；都不是，将灰色对象入队。
slowPath：检查 wbuf1 是否为 nil，为 nil，执行初始化。否则，判断是否满了，满了，交换 wbuf1 和 wbuf2，再次判断 wbuf1 是否满了，满了加入到全局 full 队列中，然后从 empty 队列获取一个空的并赋值给 wbuf1，最后把这个灰色对象入队。

#### GC 过程中负责标记的函数

##### markroot

markroot 主要是根据不同的 i 来定位要扫描哪些区域，当 i == 0 ，i == 1 时，对应到以下两种情况，具体的含义暂时还不清晰。

```go
// TODO 了解这两种分别是什么
case i == fixedRootFinalizers:
   for fb := allfin; fb != nil; fb = fb.alllink {
      cnt := uintptr(atomic.Load(&fb.cnt))
      scanblock(uintptr(unsafe.Pointer(&fb.fin[0])), cnt*unsafe.Sizeof(fb.fin[0]), &finptrmask[0], gcw, nil)
   }

case i == fixedRootFreeGStacks:
   // Switch to the system stack so we can call
   // stackfree.
   systemstack(markrootFreeGStacks)
```

##### scanobject

```go
// scanobject scans the object starting at b, adding pointers to gcw.
// b must point to the beginning of a heap object or an oblet.
// scanobject consults the GC bitmap for the pointer mask and the
// spans for the size of the object.
//
// 如注释所写，scanobject 扫描从 b 为起始地址的对象，并添加指针到 gcw 队列中。
// b 必须指向堆对象的起始地址，或者一个 oblet。
//go:nowritebarrier
func scanobject(b uintptr, gcw *gcWork) {
    // Find the bits for b and the size of the object at b.
    //
    // b is either the beginning of an object, in which case this
    // is the size of the object to scan, or it points to an
    // oblet, in which case we compute the size to scan below.
    // 这个函数前边已经详细的分析过了，没啥好说的
    hbits := heapBitsForAddr(b)
    // 找到指针 b 对应的 mspan
    s := spanOfUnchecked(b)
    // s.elemsize 表示的是一个 object 的大小
    n := s.elemsize
    if n == 0 {
        throw("scanobject n == 0")
    }
    /*
		这里对大对象（这里的大对象与内存分配的有些许差异
		内存分配大对象指的是大于32KB的，这里指的是大于 128KB的）
		进行了进一步的区分
		maxobletBytes = 128 << 10 = 128 KB
		意味着，当前 obj 的大小超过了 128 KB
		要进行一些优化操作
	*/
    if n > maxObletBytes {
        // Large object. Break into oblets for better
        // parallelism and lower latency.
        if b == s.base() {
            // It's possible this is a noscan object (not
            // from greyobject, but from other code
            // paths), in which case we must *not* enqueue
            // oblets since their bitmaps will be
            // uninitialized.
            if s.spanclass.noscan() {
                // Bypass the whole scan.
                gcw.bytesMarked += uint64(n)
                return
            }

            // Enqueue the other oblets to scan later.
            // Some oblets may be in b's scalar tail, but
            // these will be marked as "no more pointers",
            // so we'll drop out immediately when we go to
            // scan those.
            for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
                if !gcw.putFast(oblet) {
                    gcw.put(oblet)
                }
            }
        }

        // Compute the size of the oblet. Since this object
        // must be a large object, s.base() is the beginning
        // of the object.
        n = s.base() + s.elemsize - b
        if n > maxObletBytes {
            n = maxObletBytes
        }
    }

    var i uintptr
    // i < n 说明 mspan 中，只对当前这个对象进行扫描
    // 执行完一次 for 循环，并不意味着扫描完当前这个对象了
    // 只是检查了当前两个比特对应的一个字
    // 除非当前这个对象是不需要扫描的即，bits&bitScan == 0
    for i = 0; i < n; i, hbits = i+sys.PtrSize, hbits.next() {
        // Load bits once. See CL 22712 and issue 16973 for discussion.
        bits := hbits.bits()
        // 这里的两个 if 判断就好像两个卡尺
        // 分别对高位和低位进行检查
        if bits&bitScan == 0 {
            break // no more pointers in this object
        }
        if bits&bitPointer == 0 {
            continue // not a pointer
        }

        // Work here is duplicated in scanblock and above.
        // If you make changes here, make changes there too.
        /*
			设 fakeObj = (*uintptr)(unsafe.Pointer(b + i))
			对 fakeObj 解引用 --> *fakeObj
            	1.解出来结果 != 0，说明是一个指针
            	2.解出来结果 == 0，说明结果为 nil
        */
        obj := *(*uintptr)(unsafe.Pointer(b + i))


        // At this point we have extracted the next potential pointer.
        // Quickly filter out nil and pointers back to the current object.
        // 如果 obj 是一个指针，并且指向的不是当前 object
        // 笔者理解：之所以要过滤掉当前的 object，
        // 是因为当前这个 object 就是从 gcw 队列中取出来的
        // 所以也就不需要再对该 obj 进行标记（入队）
        if obj != 0 && obj-b >= n {
            // Test if obj points into the Go heap and, if so,
            // mark the object.
            // 检查 obj 是否指向了 Go 的堆，如果是，对这个 obj 进行标记
            //
            // Note that it's possible for findObject to
            // fail if obj points to a just-allocated heap
            // object because of a race with growing the
            // heap. In this case, we know the object was
            // just allocated and hence will be marked by
            // allocation itself.
            if obj, span, objIndex := findObject(obj, b, i); obj != 0 {
                greyobject(obj, b, i, span, gcw, objIndex)
            }
        }
    }
    // 记录刚刚被扫描的 obj 大小
    gcw.bytesMarked += uint64(n)
    // 积累 credit，到了 2000 就更新到全局 gcCrontroller 中
    gcw.scanWork += int64(i)
}
```

##### findObject

```go
/*
	参数解释：
		refBase, refOff 主要 panic 用
	返回值解释：
		base: 表示 p 指针所在对象，在堆中的起始地址
		s: 表示 p 指针所在的 mspan
		objIndex: 表示包含 p 指针的对象在 mspan 中的位置
*/
//go:nosplit
func findObject(p, refBase, refOff uintptr) (base uintptr, s *mspan, objIndex uintptr) {
    s = spanOf(p)

    (...) // 省略检查代码
    
    objIndex = s.objIndex(p)
    base = s.base() + objIndex*s.elemsize
    return
}
```

##### markrootBlock

##### markrootFreeGStacks

##### markrootSpans

##### scanstack

扫描 goroutine 栈

##### scanblock

##### gcmarknewobject

##### gcMarkTinyAllocs



## gcMarkDone

gcMarkDone 是一个过度的函数，是由 _GCMark 到 _GCMarkTermination 状态转换前要做的一些处理。此外，gcMarkDone 中会处理 writebarrier buffer，如果这时候将数据写入到了全局的标记队列中，那还需要继续执行标记工作。

```go
// 如果所有的可达对象都已经被标记了（意味着没有任何灰色对象并且将来也不会有）
// gcMarkDone 将 GC 的状态由 mark 转为 mark termination。
// 否则（还有没被标记的灰色对象），gcMarkDone 会将所有本地队列中的对象推到全局工作队列中，
// 这样其他的 worker 就可以知道还有活要干。
// 
// This should be called when all local mark work has been drained and
// there are no remaining workers. Specifically, when
//
//   work.nwait == work.nproc && !gcMarkWorkAvailable(p)
//
// 调用 gcMarkDone 的上下文一定是可抢占的。
//
// Flushing local work is important because idle Ps may have local
// work queued. This is the only way to make that work visible and
// drive GC to completion.
// 
// 刷新 P 的本地队列到全局队列中是十分重要的，因为 idle 状态的 P 可能在本地
// 队列中有缓存。gcMarkDone 中是唯一的方式让这些本地队列中的对象可以被看到，
// 促使 GC 完成。
//
// It is explicitly okay to have write barriers in this function. If
// it does transition to mark termination, then all reachable objects
// have been marked, so the write barrier cannot shade any more
// objects.
func gcMarkDone() {
    // Ensure only one thread is running the ragged barrier at a
    // time.
    semacquire(&work.markDoneSema)

    top:
    // Re-check transition condition under transition lock.
    //
    // It's critical that this checks the global work queues are
    // empty before performing the ragged barrier. Otherwise,
    // there could be global work that a P could take after the P
    // has passed the ragged barrier.
    if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
        semrelease(&work.markDoneSema)
        return
    }

    // forEachP needs worldsema to execute, and we'll need it to
    // stop the world later, so acquire worldsema now.
    semacquire(&worldsema)

    // Flush all local buffers and collect flushedWork flags.
    gcMarkDoneFlushed = 0
    systemstack(func() {
        gp := getg().m.curg
        // Mark the user stack as preemptible so that it may be scanned.
        // Otherwise, our attempt to force all P's to a safepoint could
        // result in a deadlock as we attempt to preempt a worker that's
        // trying to preempt us (e.g. for a stack scan).
        /*
        	标记当前 gp 是可抢占的，目的是为了能够被扫描。当我们强制所有的 P 到达
        	一个 safepoint 可能会导致死锁。比如，我们想抢占一个 worker，同时
        	那个worker也在对我们执行抢占（对我们进行栈扫描）。
        */
        casgstatus(gp, _Grunning, _Gwaiting)
        forEachP(func(_p_ *p) {
            // Flush the write barrier buffer, since this may add
            // work to the gcWork.
            /*
	            1.获取当前 P 的 writebarrier buffer 中的内容
	            2.遍历buffer中的对象，标灰，标记 span
	            3.将灰色对象刷到全局的 work queue 中
	            4.清空 P 的 write barrier buffer
            */
            wbBufFlush1(_p_)

            // Flush the gcWork, since this may create global work
            // and set the flushedWork flag.
            //
            // TODO(austin): Break up these workbufs to
            // better distribute work.
            // 这里处理的是 P 的本地队列
            // 将本地队列中的内容刷到全局队列中
            _p_.gcw.dispose()
            // Collect the flushedWork flag.
            // flushedWork 记录当前 P 是否又将对象放入到全局workqueue
            if _p_.gcw.flushedWork {
                atomic.Xadd(&gcMarkDoneFlushed, 1)
                _p_.gcw.flushedWork = false
            }
        })
        casgstatus(gp, _Gwaiting, _Grunning)
    })

    // 如果有 P 往全局队列放入对象，那还得将那些内容进行标记
    if gcMarkDoneFlushed != 0 {
        // More grey objects were discovered since the
        // previous termination check, so there may be more
        // work to do. Keep going. It's possible the
        // transition condition became true again during the
        // ragged barrier, so re-check it.
        semrelease(&worldsema)
        goto top
    }

    // There was no global work, no local work, and no Ps
    // communicated work since we took markDoneSema. Therefore
    // there are no grey objects and no more objects can be
    // shaded. Transition to mark termination.
    // 走到这一步意味着可以将状态转变为 markTermination
    now := nanotime()
    work.tMarkTerm = now
    work.pauseStart = now
    getg().m.preemptoff = "gcing"
    if trace.enabled {
        traceGCSTWStart(0)
    }
    systemstack(stopTheWorldWithSema)
    // The gcphase is _GCmark, it will transition to _GCmarktermination
    // below. The important thing is that the wb remains active until
    // all marking is complete. This includes writes made by the GC.

    // There is sometimes work left over when we enter mark termination due
    // to write barriers performed after the completion barrier above.
    // Detect this and resume concurrent mark. This is obviously
    // unfortunate.
    //
    // See issue #27993 for details.
    //
    // Switch to the system stack to call wbBufFlush1, though in this case
    // it doesn't matter because we're non-preemptible anyway.
    restart := false
    systemstack(func() {
        for _, p := range allp {
            wbBufFlush1(p)
            if !p.gcw.empty() {
                restart = true
                break
            }
        }
    })
    if restart {
        getg().m.preemptoff = ""
        systemstack(func() {
            now := startTheWorldWithSema(true)
            work.pauseNS += now - work.pauseStart
            memstats.gcPauseDist.record(now - work.pauseStart)
        })
        semrelease(&worldsema)
        goto top
    }

    // Disable assists and background workers. We must do
    // this before waking blocked assists.
    atomic.Store(&gcBlackenEnabled, 0)

    // Wake all blocked assists. These will run when we
    // start the world again.
    // 唤醒因协助标记而阻塞的 goroutine
    gcWakeAllAssists()

    // Likewise, release the transition lock. Blocked
    // workers and assists will run when we start the
    // world again.
    semrelease(&work.markDoneSema)

    // In STW mode, re-enable user goroutines. These will be
    // queued to run after we start the world.
    schedEnableUser(true)

    // endCycle depends on all gcWork cache stats being flushed.
    // The termination algorithm above ensured that up to
    // allocations since the ragged barrier.
    nextTriggerRatio := gcController.endCycle(work.userForced)

    // Perform mark termination. This will restart the world.
    gcMarkTermination(nextTriggerRatio)
}
```

## 番外篇

### suspendG

- 从全局 g 队列中获取一个，这个 g 有可能是一下几种情况
  - 正在别的 P 上运行
  - 正在进行系统调用
  - 正在某个地方等待
  - 已经是可执行状态
  - 正在发生抢占
  - 也可能是当前这个 gcMarkWorker

```go
/*
    suspendG 在一个安全时刻暂停 goroutine 并且返回被挂起 goroutine 的状态。
    suspendG 的调用者拥有该 goroutine 的读权限直到调用的 resumeG。

    多个调用者在同一时刻想挂起同一个 goroutine 是安全的。
    The goroutine may execute between subsequent successful suspend operations.
    当前的实现为互斥访问 goroutine，因此多个调用者的访问会被串行化。
    然而，这样的目的是为了读共享，所以请不要依赖互斥访问。

    suspendG 一定要在 system stack 上调用并且当前 M 上的 goroutine 一定是可以被抢占的状态。这阻止了两个 goroutine 尝试相互挂起但是他们都处在非抢占状态下发生死锁的情况。虽然还有其他的方式可以解决死锁，但是这种是最简单的方式。
*/
//go:systemstack
func suspendG(gp *g) suspendGState {
    if mp := getg().m; mp.curg != nil && readgstatus(mp.curg) == _Grunning {
        // Since we're on the system stack of this M, the user
        // G is stuck at an unsafe point. If another goroutine
        // were to try to preempt m.curg, it could deadlock.
        throw("suspendG from non-preemptible goroutine")
    }

    // See https://golang.org/cl/21503 for justification of the yield delay.
    const yieldDelay = 10 * 1000
    var nextYield int64

    // Drive the goroutine to a preemption point.
    stopped := false
    var asyncM *m
    var asyncGen uint32
    var nextPreemptM int64
    for i := 0; ; i++ {
        switch s := readgstatus(gp); s {
            default:
            if s&_Gscan != 0 {
                // Someone else is suspending it. Wait
                // for them to finish.
                //
                // TODO: It would be nicer if we could
                // coalesce suspends.
                break
            }

            dumpgstatus(gp)
            throw("invalid g status")

            case _Gdead:
            // Nothing to suspend.
            //
            // preemptStop may need to be cleared, but
            // doing that here could race with goroutine
            // reuse. Instead, goexit0 clears it.
            return suspendGState{dead: true}

            case _Gcopystack:
            // The stack is being copied. We need to wait
            // until this is done.

            case _Gpreempted:
            // We (or someone else) suspended the G. Claim
            // ownership of it by transitioning it to
            // _Gwaiting.
            if !casGFromPreempted(gp, _Gpreempted, _Gwaiting) {
                break
            }

            // We stopped the G, so we have to ready it later.
            stopped = true

            s = _Gwaiting
            fallthrough

            case _Grunnable, _Gsyscall, _Gwaiting:
            // Claim goroutine by setting scan bit.
            // This may race with execution or readying of gp.
            // The scan bit keeps it from transition state.
            if !castogscanstatus(gp, s, s|_Gscan) {
                break
            }

            // Clear the preemption request. It's safe to
            // reset the stack guard because we hold the
            // _Gscan bit and thus own the stack.
            gp.preemptStop = false
            gp.preempt = false
            gp.stackguard0 = gp.stack.lo + _StackGuard

            // The goroutine was already at a safe-point
            // and we've now locked that in.
            //
            // TODO: It would be much better if we didn't
            // leave it in _Gscan, but instead gently
            // prevented its scheduling until resumption.
            // Maybe we only use this to bump a suspended
            // count and the scheduler skips suspended
            // goroutines? That wouldn't be enough for
            // {_Gsyscall,_Gwaiting} -> _Grunning. Maybe
            // for all those transitions we need to check
            // suspended and deschedule?
            return suspendGState{g: gp, stopped: stopped}

            case _Grunning:
            // Optimization: if there is already a pending preemption request
            // (from the previous loop iteration), don't bother with the atomics.
            if gp.preemptStop && gp.preempt && gp.stackguard0 == stackPreempt && asyncM == gp.m && atomic.Load(&asyncM.preemptGen) == asyncGen {
                break
            }

            // Temporarily block state transitions.
            if !castogscanstatus(gp, _Grunning, _Gscanrunning) {
                break
            }

            // Request synchronous preemption.
            gp.preemptStop = true
            gp.preempt = true
            gp.stackguard0 = stackPreempt

            // Prepare for asynchronous preemption.
            asyncM2 := gp.m
            asyncGen2 := atomic.Load(&asyncM2.preemptGen)
            needAsync := asyncM != asyncM2 || asyncGen != asyncGen2
            asyncM = asyncM2
            asyncGen = asyncGen2

            casfrom_Gscanstatus(gp, _Gscanrunning, _Grunning)

            // Send asynchronous preemption. We do this
            // after CASing the G back to _Grunning
            // because preemptM may be synchronous and we
            // don't want to catch the G just spinning on
            // its status.
            if preemptMSupported && debug.asyncpreemptoff == 0 && needAsync {
                // Rate limit preemptM calls. This is
                // particularly important on Windows
                // where preemptM is actually
                // synchronous and the spin loop here
                // can lead to live-lock.
                now := nanotime()
                if now >= nextPreemptM {
                    nextPreemptM = now + yieldDelay/2
                    preemptM(asyncM)
                }
            }
        }

        // TODO: Don't busy wait. This loop should really only
        // be a simple read/decide/CAS loop that only fails if
        // there's an active race. Once the CAS succeeds, we
        // should queue up the preemption (which will require
        // it to be reliable in the _Grunning case, not
        // best-effort) and then sleep until we're notified
        // that the goroutine is suspended.
        if i == 0 {
            nextYield = nanotime() + yieldDelay
        }
        if nanotime() < nextYield {
            procyield(10)
        } else {
            osyield()
            nextYield = nanotime() + yieldDelay/2
        }
    }
}
```

### resumeG

### 税收与开支

- 基本理念

​		每个赋值器线程都应当参与一定的回收工作（即纳税）。同时也应当与回收器交替执行，以确保最小赋值器使用率的要求。	

​		回收器应可在赋值器执行间隙尽量多的执行回收工作（即尽量多的积累 credit，以供赋值器支出。）

```go
// gcFlushBgCredit flushes scanWork units of background scan work
// credit. This first satisfies blocked assists on the
// work.assistQueue and then flushes any remaining credit to
// gcController.bgScanCredit.
//
// Write barriers are disallowed because this is used by gcDrain after
// it has ensured that all work is drained and this must preserve that
// condition.
//
//go:nowritebarrierrec
func gcFlushBgCredit(scanWork int64) {
    if work.assistQueue.q.empty() {
        // Fast path; there are no blocked assists. There's a
        // small window here where an assist may add itself to
        // the blocked queue and park. If that happens, we'll
        // just get it on the next flush.
        atomic.Xaddint64(&gcController.bgScanCredit, scanWork)
        return
    }

    assistBytesPerWork := float64frombits(atomic.Load64(&gcController.assistBytesPerWork))
    scanBytes := int64(float64(scanWork) * assistBytesPerWork)

    lock(&work.assistQueue.lock)
    for !work.assistQueue.q.empty() && scanBytes > 0 {
        gp := work.assistQueue.q.pop()
        // Note that gp.gcAssistBytes is negative because gp
        // is in debt. Think carefully about the signs below.
        // scanBytes+gp.gcAssistBytes 意味着足够支付债务
        if scanBytes+gp.gcAssistBytes >= 0 {
            // Satisfy this entire assist debt.
            scanBytes += gp.gcAssistBytes
            gp.gcAssistBytes = 0
            // It's important that we *not* put gp in
            // runnext. Otherwise, it's possible for user
            // code to exploit the GC worker's high
            // scheduler priority to get itself always run
            // before other goroutines and always in the
            // fresh quantum started by GC.
            // 还债后唤醒这个 g
            ready(gp, 0, false)
        } else {
            // 没法完全支付债务，但是可以偿还一部分
            // 偿还完成后，又放回了协助标记的队列中
            // Partially satisfy this assist.
            gp.gcAssistBytes += scanBytes
            scanBytes = 0
            // As a heuristic, we move this assist to the
            // back of the queue so that large assists
            // can't clog up the assist queue and
            // substantially delay small assists.
            work.assistQueue.q.pushBack(gp)
            break
        }
    }
	
    // 如果仍然有剩余的 credit 就刷新到全局的 credit 中
    if scanBytes > 0 {
        // Convert from scan bytes back to work.
        assistWorkPerByte := float64frombits(atomic.Load64(&gcController.assistWorkPerByte))
        scanWork = int64(float64(scanBytes) * assistWorkPerByte)
        atomic.Xaddint64(&gcController.bgScanCredit, scanWork)
    }
    unlock(&work.assistQueue.lock)
}
```

## 题外话

GC 真的是太难了... 感觉这些都还只是冰山一角，曹大讲过，GC 这部分量力而行，对于大部分程序员来说可以完整的把三色抽象、GC 流程给面试官描述出来，已经是很厉害了，如果他还了解一些理论基础，那算是相当不错了...



我们追求的不是得到别人的认可，而是对知识的渴望。严于律己，提高自己的职业素养。

