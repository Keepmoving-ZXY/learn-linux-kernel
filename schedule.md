## 简介

在linux任务调度过程中，当为某个CPU中rq上正在运行的任务设置了`TIF_NEED_RESCHED`标志，意味着这个任务即将被其他任务抢占，即另外一个任务替代这个任务占用CPU资源，完成这个动作的函数是`schedule`函数。在`try_to_wake_up`中会将rq中正在执行的任务加上`TIF_NEED_RESCHED`标志，另外一个流程是定时器由驱动的rq的时间更新流程，在这个流程中可能会为rq中正在运行的任务添加`TIF_NEED_RESCHED`函数。在记录`schedule`函数之前，先记录高精度定时器如何影响任务调度过程的。

## 代码流程

### 1.高精度定时器

本节关注高精度定时器对调度流程的影响，大致的流程是使用`hrtick_rq_init`函数初始化rq中高精度定时器，然后调用`__hrtick_restart`启用rq中的高精度定时器，当定时器触发时调用`hrtick`函数更新rq的时间相关字段。

#### `hrtick_rq_init`函数

```c
static void hrtick_rq_init(struct rq *rq)
{
#ifdef CONFIG_SMP
	INIT_CSD(&rq->hrtick_csd, __hrtick_start, rq);
#endif
	hrtimer_init(&rq->hrtick_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_HARD);
	rq->hrtick_timer.function = hrtick;
}
```

这个函数初始化rq中高精度定时器，这个定时器使用单调递增时钟、使用相对时间表示定时器触发时刻、在硬件中断上下文中运行，定时器触发是执行的函数是`hrtick`。同时还会初始化`call single data`相关的字段，`call single data`的用途在接下来的函数中说明。

#### `hrtick_start`函数

```c
/*
 * Called to set the hrtick timer state.
 *
 * called with rq->lock held and irqs disabled
 */
void hrtick_start(struct rq *rq, u64 delay)
{
	struct hrtimer *timer = &rq->hrtick_timer;
	s64 delta;

	/*
	 * Don't schedule slices shorter than 10000ns, that just
	 * doesn't make sense and can cause timer DoS.
	 */
	delta = max_t(s64, delay, 10000LL);
	rq->hrtick_time = ktime_add_ns(timer->base->get_time(), delta);

	if (rq == this_rq())
		__hrtick_restart(rq);
	else
		smp_call_function_single_async(cpu_of(rq), &rq->hrtick_csd);
}
```

这个函数设置触发时间为当前时间加上至少10us的延迟时间，如果定时器所属的rq与当前运行CPU的rq相同，则调用`__krtick_restart`函数启动定时器，反之则通知rq所属cpu执行`__hrtick_start`函数启动定时器（`smp_call_function_single_async`）。

#### `__hrtick_restart`函数

```c
static void __hrtick_restart(struct rq *rq)
{
	struct hrtimer *timer = &rq->hrtick_timer;
	ktime_t time = rq->hrtick_time;

	hrtimer_start(timer, time, HRTIMER_MODE_ABS_PINNED_HARD);
}
```

这个函数调用`hrtimer_start`启动一个高精度定时器，`HRTIMER_MODE_ABS_PINNED_HARD`表示这个高精度定时器触发时间不是相对时间、这个定时器在指定CPU中运行、这个定时器在硬中断的上下文中执行。

#### `__hrtick_start`函数

```c
/*
 * called from hardirq (IPI) context
 */
static void __hrtick_start(void *arg)
{
	struct rq *rq = arg;
	struct rq_flags rf;

	rq_lock(rq, &rf);
	__hrtick_restart(rq);
	rq_unlock(rq, &rf);
}
```

这个函数在硬件中断上下文中执行（`IPI`中断）中执行，在持有rq锁的前提下在rq所在的CPU中调用`__hrtick_restart`函数启动高精度定时器。

#### `hrtick`函数

```c
/*
 * High-resolution timer tick.
 * Runs from hardirq context with interrupts disabled.
 */
static enum hrtimer_restart hrtick(struct hrtimer *timer)
{
	struct rq *rq = container_of(timer, struct rq, hrtick_timer);
	struct rq_flags rf;

	WARN_ON_ONCE(cpu_of(rq) != smp_processor_id());

	rq_lock(rq, &rf);
	update_rq_clock(rq);
	rq->curr->sched_class->task_tick(rq, rq->curr, 1);
	rq_unlock(rq, &rf);

	return HRTIMER_NORESTART;
}
```

这个函数在持有rq锁的前提下，更新rq的时间相关字段、调用rq所在CPU中正在运行的任务使用的调度类的`task_tick`方法。注意到这个函数返回的是`HRTIMER_NORESTART`，意味着这个定时器不会自动启用，在调度类中才会继续启动这个定时器，例如CFS的`enqueue_task_fair`函数。
