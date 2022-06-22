---
title: timer 优化
comments: true
---



## 前言

我们已经知道了，老版本 timer 的性能瓶颈主要是在那把全局锁以及频繁的上下文切换上，今天我们看看 go 大佬们通过哪种方式进行优化的。
​

在这里解释一下为什么选择这几个版本，据我所知啊，从 1.10 版本以前都是像上一篇文中所描述的那样，在 1.10 版本开始就做了这个优化，但从 1.14 开始又对 timer 进行了优化，所以我选择了 1.8， 1.13， 1.14 这几个邻近的作为参考。

<!--more-->

## go 1.13 中的优化
### 环境信息

- go version: go1.13
- compute: linux, centos7

### timer 结构的变化
##### go1.8
```go
type timer struct {
	i int // heap index
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
}
```
##### go1.13

```go
type timer struct {
	tb *timersBucket // the bucket the timer lives in
	i  int           // heap index
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr
}
```
在go1.13版本中添加了一个 tb 字段，表示当前这个 timer 是在哪个 bucket 中的，其余字段含义还是和老版本中的一致。
​

还记得老版本把新建的 `timer` 对象都放在哪里了吗？`一个全局的 timers 中` go1.13版本中将的 timers 拆分成了 64 个大小的 timers 数组，每一个里边包含了一个 bucket ，bucket 中再存放 timer 对象，至于为什么是 64 官方的解释如下：
```go
// timersLen is the length of timers array.
//
// Ideally, this would be set to GOMAXPROCS, but that would require
// dynamic reallocation
//
// The current value is a compromise between memory usage and performance
// that should cover the majority of GOMAXPROCS values used in the wild.
const timersLen = 64

// timersLen is the length of timers array.
//
// Ideally, this would be set to GOMAXPROCS, but that would require
// dynamic reallocation
//
// The current value is a compromise between memory usage and performance
// that should cover the majority of GOMAXPROCS values used in the wild.
var timers [timersLen]struct {
	timersBucket

	// The padding should eliminate false sharing
	// between timersBucket values.
	pad [cpu.CacheLinePadSize - unsafe.Sizeof(timersBucket{})%cpu.CacheLinePadSize]byte
}

//go:notinheap
type timersBucket struct {
	lock         mutex
	gp           *g
	created      bool
	sleeping     bool
	rescheduling bool
	sleepUntil   int64
	waitnote     note
	t            []*timer
}
```
在添加 timer 对象时逻辑变成了，根据当前 p 的 id 对 timersLen 取模，得到了 p 对应的 timersBucket `id := uint8(getg().m.p.ptr().id) % _timersLen_`
从这个优化的方法来看，以前是每个p去抢同一把锁，现在变成，每个p只会操作对应的 timersBucket（大多数情况下）。

- [ ] **在超过 64 个 p 的时候，就会出现取模到同一个 bucket 中，这种情况在多核 cpu > 64 上是没办法避免的**
- [ ] **可能还有 p 从别的 p 上偷 timer 的情况**

接下里我们看下执行 timer 的 `timerproc`
1.8 版本中，全局只有一个执行 timer 的 timerproc，可以理解为只有一个消费者。1.13 中修改为每个不同的 bucket 都会有一个对应的 bucket。举个例子，比如我们有 4 个 P，就说明我们会有 4 个 bucket 和 4 个 timerproc，每当通过 addtimer 添加时，都会往 p 对应的 bucket 中添加任务，timerproc 作为消费者从中找可执行的timer，如下图：
![image.png](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/1636033788855-ece14ebf-e72d-4851-9a75-de7cd60d59ba.png)

### timer 的“生产者”
```go
// 如前所述，每个p对应不同的 timersBucket，那么在创建之前我们是不是应该先找到在哪个 p 上执行
func (t *timer) assignBucket() *timersBucket {
	id := uint8(getg().m.p.ptr().id) % timersLen
	t.tb = &timers[id].timersBucket
	return t.tb
}

// 将 timer 添加到对应的 timersBucket 中
func addtimer(t *timer) {
	tb := t.assignBucket()
	lock(&tb.lock)
	ok := tb.addtimerLocked(t)
	unlock(&tb.lock)
	if !ok {
		badTimer()
	}
}

// 咱们只把重点放在与以前不同的地方上
func (tb *timersBucket) addtimerLocked(t *timer) bool {
	if t.when < 0 {
		t.when = 1<<63 - 1
	}
	t.i = len(tb.t)
	tb.t = append(tb.t, t) 		// 添加到p对应的timersBucket中，而不是全局的 timers 中了
	if !siftupTimer(tb.t, t.i) {
		return false
	}
	if t.i == 0 {
		// siftup moved to top: new earliest deadline.
		if tb.sleeping && tb.sleepUntil > t.when {
			tb.sleeping = false
			notewakeup(&tb.waitnote)
		}
		if tb.rescheduling {
			tb.rescheduling = false
			goready(tb.gp, 0)
		}
		if !tb.created {
			// 判断属于这个 tb 的 timerproc 是否启动了，
			// 区别于1.8版本是一个全局变量控制的，只有一个消费者，这里是每一个 tb 都有一个消费者
			tb.created = true
			go timerproc(tb)
		}
	}
	return true
}
```
### timer 的“消费者”
```go
// 主要的逻辑还是同 1.8 版本中一致的，不同的地方就是针对每个tb进行的操作，不是全局的 timers
func timerproc(tb *timersBucket) {
	tb.gp = getg()
	for {
		lock(&tb.lock)
		tb.sleeping = false
		now := nanotime()
		delta := int64(-1)
		for {
			if len(tb.t) == 0 {
				delta = -1
				break
			}
			t := tb.t[0]
			delta = t.when - now
			if delta > 0 {
				break
			}
			ok := true
			if t.period > 0 {
				// leave in heap but adjust next time to fire
				t.when += t.period * (1 + -delta/t.period)
				if !siftdownTimer(tb.t, 0) {
					ok = false
				}
			} else {
				// remove from heap
				last := len(tb.t) - 1
				if last > 0 {
					tb.t[0] = tb.t[last]
					tb.t[0].i = 0
				}
				tb.t[last] = nil
				tb.t = tb.t[:last]
				if last > 0 {
					if !siftdownTimer(tb.t, 0) {
						ok = false
					}
				}
				t.i = -1 // mark as removed
			}
			f := t.f
			arg := t.arg
			seq := t.seq
			unlock(&tb.lock)
			if !ok {
				badTimer()
			}
			if raceenabled {
				raceacquire(unsafe.Pointer(t))
			}
			f(arg, seq)
			lock(&tb.lock)
		}
		if delta < 0 || faketime > 0 {
			// No timers left - put goroutine to sleep.
			tb.rescheduling = true
			goparkunlock(&tb.lock, waitReasonTimerGoroutineIdle, traceEvGoBlock, 1)
			continue
		}
		// At least one timer pending. Sleep until then.
		tb.sleeping = true
		tb.sleepUntil = now + delta
		noteclear(&tb.waitnote)
		unlock(&tb.lock)
		notetsleepg(&tb.waitnote, delta)
	}
}
```
相比于 1.8，1.13版本中还添加了一个 `modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr)`
modtimer 函数主要做了，将 t 从 tb 中删除，然后有 重新给它 加入进去

```go
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) {
	tb := t.tb

	lock(&tb.lock)
	_, ok := tb.deltimerLocked(t)
	if ok {
		t.when = when
		t.period = period
		t.f = f
		t.arg = arg
		t.seq = seq
		ok = tb.addtimerLocked(t)
	}
	unlock(&tb.lock)
	if !ok {
		badTimer()
	}
}
```
在 netpoll 中，有两处地方调用了这个函数，主要就是给 fd 调整超时处理使用的。

总的来说这个版本中的优化只是做了全局锁粒度的拆分，上下文切换带来额外的性能开销仍然没有得到优化，不过不要着急，1.14 版本中针对这个问题已经做了妥善的处理，我们马上就来看一下。
​

## go 1.14 中的优化

### 环境信息
go version: go1.14.1
compute: linux, centos7

### timer 结构的变化
以前的结构体都是全局变量，在 1.14 版本开始，timer 结构体就内嵌到了 P 中。
```go
// Package time knows the layout of this structure.
// If this struct changes, adjust ../time/sleep.go:/runtimeTimer.
type timer struct {
	// If this timer is on a heap, which P's heap it is on.
	// puintptr rather than *p to match uintptr in the versions
	// of this struct defined in other packages.
	pp puintptr

	// Timer wakes up at when, and then at when+period, ... (period > 0 only)
	// each time calling f(arg, now) in the timer goroutine, so f must be
	// a well-behaved function and not block.
	when   int64
	period int64
	f      func(interface{}, uintptr)
	arg    interface{}
	seq    uintptr

	// What to set the when field to in timerModifiedXX status.
	nextwhen int64

	// The status field holds one of the values below.
	status uint32
}
```
```go
type p struct {
	id          int32
    
    (...)
    
	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	// 讲道理，你p处理本地的 timer 用锁干什么？
	// 1.14 是可以偷 timer 的，这时候就变成了共享资源，访问的时候是一定要加锁的。
	// 上边注释（英文）说的也很清楚，这个是官方的解释
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer

	// Number of timers in P's heap.
	// Modified using atomic instructions.
	// 记录当前 p 中 timer的总数量
	numTimers uint32

	// Number of timerModifiedEarlier timers on P's heap.
	// This should only be modified while holding timersLock,
	// or while the timer status is in a transient state
	// such as timerModifying.
	// P 中 调整 when 的时间提前了的 timer 数量
	adjustTimers uint32

	// Number of timerDeleted timers in P's heap.
	// Modified using atomic instructions.
	// 记录 p 中被删除的 timer 数量
	deletedTimers uint32

	// Race context used while executing timer functions.
	timerRaceCtx uintptr
    
    (...)
}

```
### timer 的“生产者”
`... 基本上大同小异，只不过是加了一些状态`
不过需要注意的一点是，1.14 中有了 timer 和 netpoll 的结合。我的理解是：
findrunnable 最后没有找到可执行的 g 的时候会再检查 netpoll。这个调用过程是阻塞的，阻塞 delta 这段时间，然后这时候比如说我通过 addtimer 加入新timer，就是假设哈，1s 后要执行，然后你那个阻塞过程要阻塞 3s，但这是在阻塞没有办法执行我们的 timer，然后这时候 addtimer 中的 wakenetpoller 就派上用场，通过 `netpollbreak` 中断那个阻塞调用，然后就回到 `findrunnable` 继续执行，及时响应那个近期的 timer 对象。

```go
// addtimer:
//   timerNoStatus   -> timerWaiting
//   anything else   -> panic: invalid value
// deltimer:
//   timerWaiting         -> timerModifying -> timerDeleted
//   timerModifiedEarlier -> timerModifying -> timerDeleted
//   timerModifiedLater   -> timerModifying -> timerDeleted
//   timerNoStatus        -> do nothing
//   timerDeleted         -> do nothing
//   timerRemoving        -> do nothing
//   timerRemoved         -> do nothing
//   timerRunning         -> wait until status changes
//   timerMoving          -> wait until status changes
//   timerModifying       -> wait until status changes
// modtimer:
//   timerWaiting    -> timerModifying -> timerModifiedXX
//   timerModifiedXX -> timerModifying -> timerModifiedYY
//   timerNoStatus   -> timerModifying -> timerWaiting
//   timerRemoved    -> timerModifying -> timerWaiting
//   timerDeleted    -> timerModifying -> timerModifiedXX
//   timerRunning    -> wait until status changes
//   timerMoving     -> wait until status changes
//   timerRemoving   -> wait until status changes
//   timerModifying  -> wait until status changes
// cleantimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// adjusttimers (looks in P's timer heap):
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerModifiedXX -> timerMoving -> timerWaiting
// runtimer (looks in P's timer heap):
//   timerNoStatus   -> panic: uninitialized timer
//   timerWaiting    -> timerWaiting or
//   timerWaiting    -> timerRunning -> timerNoStatus or
//   timerWaiting    -> timerRunning -> timerWaiting
//   timerModifying  -> wait until status changes
//   timerModifiedXX -> timerMoving -> timerWaiting
//   timerDeleted    -> timerRemoving -> timerRemoved
//   timerRunning    -> panic: concurrent runtimer calls
//   timerRemoved    -> panic: inconsistent timer heap
//   timerRemoving   -> panic: inconsistent timer heap
//   timerMoving     -> panic: inconsistent timer heap
```
### timer 的“消费者”
新版本中的“消费者”有着非常重要的改变，`timerproc` 没了，首先我们要明确：
timerproc 不仅仅是一个函数，它是 runtime 创建的一个 goroutine，因此可知，以前的“消费者”就是一个 goroutine， 它并没有什么不同，同样被 `scheduler`调度。

1.14 中，直接给“消费者”升到“头等舱”，看你小子干活勤勤恳恳，scheduler说，你来我这上班吧，结果人家就去了。

1.14 中，timer 的消费者就是在调度循环的 `schedule` 中，其次就是 `sysmon` （sysmon作为兜底），我们看下源码，看看新版本的消费者是怎么“晋升”的。

```go
func schedule() {
    (...) // 省略安全检查
    
	// 看看人家 timer 直接被安排到顶级位置
	// 调度循环上来就是先检查 timer
	checkTimers(pp, 0)

    (...)
    
    if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}

    (...)
    
	execute(gp, inheritTime)
}
```
可以得出结论，新版本的“消费者” 从 goroutine 级别 转变到 函数级别。

#### checkTimers()
```go
// checkTimers runs any timers for the P that are ready.
// If now is not 0 it is the current time.
// It returns the current time or 0 if it is not known,
// and the time when the next timer should run or 0 if there is no next timer,
// and reports whether it ran any timers.
// If the time when the next timer should run is not 0,
// it is always larger than the returned time.
// We pass now in and out to avoid extra calls of nanotime.
//go:yeswritebarrierrec
// rnow			
// pollUntil	0 表示没有下一个 timer，非 0 表示下一个timer的等待时间
// ran			表示是否执行了 timer
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
	// If there are no timers to adjust, and the first timer on
	// the heap is not yet ready to run, then there is nothing to do.
	if atomic.Load(&pp.adjustTimers) == 0 {
		next := int64(atomic.Load64(&pp.timer0When))
		if next == 0 {
			return now, 0, false
		}
		if now == 0 {
			now = nanotime()
		}
		if now < next {
			// Next timer is not ready to run.
			// But keep going if we would clear deleted timers.
			// This corresponds to the condition below where
			// we decide whether to call clearDeletedTimers.
			// 尽可能找机会清理 timer
			if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
				return now, next, false
			}
		}
	}

	lock(&pp.timersLock)

	adjusttimers(pp)

	rnow = now
	if len(pp.timers) > 0 {
		if rnow == 0 {
			rnow = nanotime()
		}
		for len(pp.timers) > 0 {
			// Note that runtimer may temporarily unlock
			// pp.timersLock.
			if tw := runtimer(pp, rnow); tw != 0 {
				if tw > 0 {
					pollUntil = tw
				}
				break
			}
			ran = true
		}
	}

	// If this is the local P, and there are a lot of deleted timers,
	// clear them out. We only do this for the local P to reduce
	// lock contention on timersLock.
	if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
		clearDeletedTimers(pp)
	}

	unlock(&pp.timersLock)

	return rnow, pollUntil, ran
}
```
在上述 `checkTimers` 中，通过 `adjusttimers` 调整当前 p 的 timers 数组，我们看一下它的实现
```go
// adjusttimers looks through the timers in the current P's heap for
// any timers that have been modified to run earlier, and puts them in
// the correct place in the heap. While looking for those timers,
// it also moves timers that have been modified to run later,
// and removes deleted timers. The caller must have locked the timers for pp.
func adjusttimers(pp *p) {
	// 判断当前 p 是否有 timer
	if len(pp.timers) == 0 {
		return
	}
	if atomic.Load(&pp.adjustTimers) == 0 {
		if verifyTimers {
			verifyTimerHeap(pp)
		}
		return
	}
	// 存放需要移动的 timer
	var moved []*timer
loop:
	// 遍历当前 p 的 timers
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		if t.pp.ptr() != pp {
			throw("adjusttimers: bad p")
		}
		// 判断当前 timer 的状态
		switch s := atomic.Load(&t.status); s {
		// 表示 timer 需要删除，但是还没有删除呢
		case timerDeleted:
			// 修改 timer 的状态为，正在删除中 timerRemoving
			if atomic.Cas(&t.status, s, timerRemoving) {
				// 执行删除操作
				dodeltimer(pp, i)
				// 修改 timer 的状态为，已删除 timerRemoved
				if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
					badTimer()
				}
				// 修改待删除 timer 的数量 pp.deletedTimers - 1
				atomic.Xadd(&pp.deletedTimers, -1)
				// Look at this heap position again.
				// 思考一下就可以知道，为什么需要再次检查当前这个位置的 timer
				// 通过 dodeltimer 将索引为 i 的 timer 删除后，我们知道的是
				// 假设总数量为 n, [0, i) 之前的元素不需要改变，删掉第 I 个后
				// 需要在 [i,n-1) 里边中选一个填补 i 的位置，所以需要重新检查一次
				i--
			}
		// 表示 timer 的等待时间被调整了
		// timerModifiedEarlier 向前调整
		// timerModifiedLater 向后调整
		case timerModifiedEarlier, timerModifiedLater:
			// 因为调整了 timer 的时间点，所以需要重新调整该 timer 在堆中的位置
			// 修改 timer 状态为，移动中 timerMoving
			if atomic.Cas(&t.status, s, timerMoving) {
				// Now we can change the when field.
				t.when = t.nextwhen
				// Take t off the heap, and hold onto it.
				// We don't add it back yet because the
				// heap manipulation could cause our
				// loop to skip some other timer.
				dodeltimer(pp, i)
				// 将这个 timer 加入到需要移动的 timer 当中
				moved = append(moved, t)
				if s == timerModifiedEarlier {
					if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
						break loop
					}
				}
				// Look at this heap position again.
				i--
			}
		case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
			badTimer()
		case timerWaiting:
			// OK, nothing to do.
		case timerModifying:
			// Check again after modification is complete.
			osyield()
			i--
		default:
			badTimer()
		}
	}

	if len(moved) > 0 {
		// 将 timer 重新加入到当前 p 的 timers 中
		// 并且按照小顶堆进行排序
		addAdjustedTimers(pp, moved)
	}

	if verifyTimers {
		verifyTimerHeap(pp)
	}
}
```
然后我们再看一下 `checkTimers` 函数末尾的位置，就是要真正执行 timer 的时候了，通过 `runtimer` 来执行 p 中的 timer
```go
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.

/*
	根据上述注释可以了解到:
	返回值 = 0;  表示执行了一个 timer
	返回值 = -1; 表示 p 中没有 timer 了
	返回值 > 0;  表示第一个 timer 要执行的时间点
	
	（这里的源码就不做过多分析了，没有什么可说的，基本上都覆盖到了）
*/

//go:systemstack
func runtimer(pp *p, now int64) int64 {
	for {
		t := pp.timers[0]
		if t.pp.ptr() != pp {
			throw("runtimer: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerWaiting:
			if t.when > now {
				// Not ready to run.
				return t.when
			}

			if !atomic.Cas(&t.status, s, timerRunning) {
				continue
			}
			// Note that runOneTimer may temporarily unlock
			// pp.timersLock.
			runOneTimer(pp, t, now)
			return 0

		case timerDeleted:
			if !atomic.Cas(&t.status, s, timerRemoving) {
				continue
			}
			dodeltimer0(pp)
			if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
				badTimer()
			}
			atomic.Xadd(&pp.deletedTimers, -1)
			if len(pp.timers) == 0 {
				return -1
			}

		case timerModifiedEarlier, timerModifiedLater:
			if !atomic.Cas(&t.status, s, timerMoving) {
				continue
			}
			t.when = t.nextwhen
			dodeltimer0(pp)
			doaddtimer(pp, t)
			if s == timerModifiedEarlier {
				atomic.Xadd(&pp.adjustTimers, -1)
			}
			if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
				badTimer()
			}

		case timerModifying:
			// Wait for modification to complete.
			osyield()

		case timerNoStatus, timerRemoved:
			// Should not see a new or inactive timer on the heap.
			badTimer()
		case timerRunning, timerRemoving, timerMoving:
			// These should only be set when timers are locked,
			// and we didn't do it.
			badTimer()
		default:
			badTimer()
		}
	}
}
```
截止到目前为止，我们已经把 `checkTimers` 给分析完了。
### 偷 timer
这里的偷 timer 不是说把 另一个 p 的 timer 偷到我本地后再执行，而是在当前这个 p ，执行其他 p timer。
```go
// 截取部分 findrunnable 代码
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            (...)
            
            if i > 2 && shouldStealTimers(p2) {
				tnow, w, ran := checkTimers(p2, now)
                (...)
            }
		}
	}
```

## 后语

给大家分享一下我在整理过程中的参考资料吧：
_1.luozhiyun Go中定时器实现原理及源码解析_
   [_https://www.cnblogs.com/luozhiyun/p/14494540.html_](https://www.cnblogs.com/luozhiyun/p/14494540.html)
_2.猪吃鱼 Netpoll 解析_
   [_https://www.pefish.club/2020/05/04/Golang/1011Netpoll%E8%A7%A3%E6%9E%90/_](https://www.pefish.club/2020/05/04/Golang/1011Netpoll%E8%A7%A3%E6%9E%90/)
_3.issues 
   [_https://github.com/golang/go/issues/6239_](https://github.com/golang/go/issues/6239)
4.峰云就她了  go1.14基于netpoll定时器实现原理_ 
   [_http://xiaorui.cc/archives/6483_](http://xiaorui.cc/archives/6483)

