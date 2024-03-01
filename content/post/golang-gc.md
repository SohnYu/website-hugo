---
title:       "Golang GC 源码阅读"
subtitle:    ""
description: ""
date:        2018-06-04
author:      ""
image:       ""
tags:        ["GC", "Golang", "SourceCode"]
categories:  ["Golang" ]
---
### Golang GC 源码阅读

goland的内存回收 无分代(使用分代是为了倾向回收新生对象 go创建在栈上) 无碎片整理(go使用的tcmalloc内存分配器) 与用户协程并发执行

| 阶段       | 说明                                                     | 赋值器状态 |
|------------|----------------------------------------------------------|------------|
| 清扫终止   | 为下一个阶段的并发标记做准备工作，启动写屏障           | STW        |
| 标记       | 与赋值器并发执行，写屏障处于开启状态                   | 并发       |
| 标记终止   | 保证一个周期内标记任务完成，停止写屏障                 | STW        |
| 内存清扫   | 将需要回收的内存归还到堆中，写屏障处于关闭状态         | 并发       |
| 内存归还   | 将过多的内存归还给操作系统，写屏障处于关闭状态         | 并发       |

> 自动内存管理的另一个重要的组成部分便是自动回收。在自动内存回收中， 垃圾回收器扮演一个十分重要的角色。通常， 垃圾回收器的执行过程可根据代码的行为被划分为两个半独立的组件： 赋值器（Mutator）和回收器（Collector）。
赋值器一词最早由 Dijkstra 引入 [Dijkstra et al., 1978]，意指用户态代码。 因为对垃圾回收器而言，需要回收的内存是由用户态的代码产生的， 用户态代码仅仅只是在修改对象之间的引用关系（对象之间引用关系的一个有向图，即对象图） 进行操作。回收器即为程序运行时负责执行垃圾回收的代码。

``` go
func GC() {
	n := work.cycles.Load()
	gcWaitOnMark(n)

	gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})

	gcWaitOnMark(n + 1)

	for work.cycles.Load() == n+1 && sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
		Gosched()
	}

	for work.cycles.Load() == n+1 && !isSweepDone() {
		Gosched()
	}

	mp := acquirem()
	cycle := work.cycles.Load()
	if cycle == n+1 || (gcphase == _GCmark && cycle == n+2) {
		mProf_PostSweep()
	}
	releasem(mp)
}
```

#### gcWaitOnMark

```
这个函数是 gcWaitOnMark，它的作用是等待垃圾回收完成第 N 次标记阶段。如果垃圾回收已经完成了这个标记阶段，那么函数会立即返回，否则会一直等待直到完成为止。现在让我逐步解释这个函数的主要部分：

lock(&work.sweepWaiters.lock)：这一行代码用于获取一个锁，确保在访问 work.sweepWaiters 结构体时的线程安全性。

nMarks := work.cycles.Load()：这里通过 work.cycles.Load() 函数获取当前的循环次数，表示当前是第几个垃圾回收周期。

if gcphase != _GCmark {...}：这个条件判断当前的垃圾回收阶段是否是标记阶段。如果不是标记阶段，意味着当前阶段已经完成了。因此，nMarks++ 会使 nMarks 表示下一个循环的次数。

if nMarks > n {...}：这个条件检查当前的循环次数是否已经超过了目标循环次数 n。如果超过了，说明垃圾回收已经完成了第 N 次标记阶段，函数就会释放锁并返回。

work.sweepWaiters.list.push(getg())：如果还没有完成目标循环次数的标记阶段，那么当前的 goroutine（通过 getg() 获取）会加入到等待垃圾回收完成的队列中。

goparkunlock(&work.sweepWaiters.lock, waitReasonWaitForGCCycle, traceBlockUntilGCEnds, 1)：这一行代码会将当前 goroutine 挂起，等待垃圾回收完成。它会解锁之前获取的锁，以允许其他 goroutine 进入临界区。挂起的原因被标记为 waitReasonWaitForGCCycle，这样在追踪和调试时就知道了当前 goroutine 是为了等待垃圾回收而挂起的。最后一个参数 1 表示当前 goroutine 是主动挂起的，而不是被调度器挂起的。
```

#### gcStart

```go
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
    // 验证垃圾收集条件 ,并清理已经被标记的内存单元
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

	// In gcstoptheworld debug mode, upgrade the mode accordingly.
	// We do this after re-checking the transition condition so
	// that multiple goroutines that detect the heap trigger don't
	// start multiple STW GCs.
	mode := gcBackgroundMode
	if debug.gcstoptheworld == 1 {
		mode = gcForceMode
	} else if debug.gcstoptheworld == 2 {
		mode = gcForceBlockMode
	}

	// Ok, we're doing it! Stop everybody else
	semacquire(&gcsema)
	semacquire(&worldsema)

	// For stats, check if this GC was forced by the user.
	// Update it under gcsema to avoid gctrace getting wrong values.
	work.userForced = trigger.kind == gcTriggerCycle

	if traceEnabled() {
		traceGCStart()
	}

	// Check that all Ps have finished deferred mcache flushes.
	for _, p := range allp {
		if fg := p.mcache.flushGen.Load(); fg != mheap_.sweepgen {
			println("runtime: p", p.id, "flushGen", fg, "!= sweepgen", mheap_.sweepgen)
			throw("p mcache not flushed")
		}
	}

	gcBgMarkStartWorkers()

	systemstack(gcResetMarkState)

	work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
	if work.stwprocs > ncpu {
		// This is used to compute CPU time of the STW phases,
		// so it can't be more than ncpu, even if GOMAXPROCS is.
		work.stwprocs = ncpu
	}
	work.heap0 = gcController.heapLive.Load()
	work.pauseNS = 0
	work.mode = mode

	now := nanotime()
	work.tSweepTerm = now
	work.pauseStart = now
	systemstack(func() { stopTheWorldWithSema(stwGCSweepTerm) })
	// Finish sweep before we start concurrent scan.
	systemstack(func() {
		finishsweep_m()
	})

	// clearpools before we start the GC. If we wait they memory will not be
	// reclaimed until the next GC cycle.
	clearpools()

	work.cycles.Add(1)

	// Assists and workers can start the moment we start
	// the world.
	gcController.startCycle(now, int(gomaxprocs), trigger)

	// Notify the CPU limiter that assists may begin.
	gcCPULimiter.startGCTransition(true, now)

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

	// In STW mode, we could block the instant systemstack
	// returns, so make sure we're not preemptible.
	mp = acquirem()

	// Concurrent mark.
	systemstack(func() {
		now = startTheWorldWithSema()
		work.pauseNS += now - work.pauseStart
		work.tMark = now
		memstats.gcPauseDist.record(now - work.pauseStart)

		sweepTermCpu := int64(work.stwprocs) * (work.tMark - work.tSweepTerm)
		work.cpuStats.gcPauseTime += sweepTermCpu
		work.cpuStats.gcTotalTime += sweepTermCpu

		// Release the CPU limiter.
		gcCPULimiter.finishGCTransition(now)
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
```

