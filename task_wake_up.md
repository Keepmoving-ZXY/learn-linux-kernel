## 简介

task唤醒涉及到的函数是`try_to_wake_up`，函数的声明为`statc int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)`，函数的作用是当指定的task `p`的状态包含`state`指定状态时尝试将task唤醒，注意这里的task可能是一个进程也可能是一个线程（即轻量级进程）。`wake_up_process`函数中会调用这个函数，`wake_up_process`函数在内核中许多地方会使用，在GPU驱动中就可以看到这个函数的身影。

## 函数流程

这个函数首先禁止任务抢占：

```c
    unsigned long flags;
    int cpu, success = 0;

    preempt_disable();
```

### 情况1：被唤醒的task与当前正在执行的task为同一个task

若需要被唤醒的task `p`与当前正在执行的task是同一个task，则判断任务状态是否匹配，当任务状态匹配时将task `p`的状态配置为正在运行（`TASK_RUNNING`），随后代码跳转到`out`标识符，代码如下：

```c
    if (p == current) {
        /*
         * We're waking current, this means 'p->on_rq' and 'task_cpu(p)
         * == smp_processor_id()'. Together this means we can special
         * case the whole 'p->on_rq && ttwu_runnable()' case below
         * without taking any locks.
         *
         * In particular:
         *  - we rely on Program-Order guarantees for all the ordering,
         *  - we're serialized against set_special_state() by virtue of
         *    it disabling IRQs (this allows not taking ->pi_lock).
         */
        if (!ttwu_state_match(p, state, &success))
            goto out;

        trace_sched_waking(p);
        WRITE_ONCE(p->__state, TASK_RUNNING);
        trace_sched_wakeup(p);
        goto out;
    }
```

上边代码中`trace_sched_xxx`函数实现暂时忽略，为了保证缓存一致性使用`WRITE_ONCE`修改task的状态，重点关注`ttwu_state_match`这个函数。

#### `ttwu_state_match`函数

任务状态的判断实现在这个函数中，实现如下：

```c
static __always_inline
bool ttwu_state_match(struct task_struct *p, unsigned int state, int *success)
{
    if (IS_ENABLED(CONFIG_DEBUG_PREEMPT)) {
        WARN_ON_ONCE((state & TASK_RTLOCK_WAIT) &&
                 state != TASK_RTLOCK_WAIT);
    }

    if (READ_ONCE(p->__state) & state) {
        *success = 1;
        return true;
    }

#ifdef CONFIG_PREEMPT_RT
    /*
     * Saved state preserves the task state across blocking on
     * an RT lock.  If the state matches, set p::saved_state to
     * TASK_RUNNING, but do not wake the task because it waits
     * for a lock wakeup. Also indicate success because from
     * the regular waker's point of view this has succeeded.
     *
     * After acquiring the lock the task will restore p::__state
     * from p::saved_state which ensures that the regular
     * wakeup is not lost. The restore will also set
     * p::saved_state to TASK_RUNNING so any further tests will
     * not result in false positives vs. @success
     */
    if (p->saved_state & state) {
        p->saved_state = TASK_RUNNING;
        *success = 1;
    }
#endif
    return false;
```

 仅仅关注这个函数中第2个`if`代码中的内容，`READ_ONCE`保证总是拿到`p->__state`字段的最新的值（相对于在CPU缓存中的值），读取到状态之后与指定的状态进行对比，若状态匹配则返回成功，`ttwu_state_match`函数的逻辑说明到此结束。

这里可以看到在读、写任务状态的时候使用了`READ_ONCE`以及`WRITE_ONCE`宏，关于这两个宏的详细介绍建议直接阅读内核文档中的内容（[https://www.kernel.org/doc/Documentation/memory-barriers.txt]()）。

### 情况2：被唤醒的task与当前正在运行的task不是同一个task

#### 2.1.任务状态检查

这种情况下首先调用`ttwu_state_match`函数进行状态检查，如果状态不匹配则跳转到`out`标签，在调用这个函数之前需要持有`pi_lock`锁、禁用当前CPU的中断、保存中断状态（仅在这个这个函数中很难看到`pi_lock`的作用，需要结合更多进程调度源码进行分析），代码如下：

```c
	/*
	 * If we are going to wake up a thread waiting for CONDITION we
	 * need to ensure that CONDITION=1 done by the caller can not be
	 * reordered with p->state check below. This pairs with smp_store_mb()
	 * in set_current_state() that the waiting thread does.
	 */
	raw_spin_lock_irqsave(&p->pi_lock, flags);
	smp_mb__after_spinlock();
	if (!ttwu_state_match(p, state, &success))
		goto unlock;

	trace_sched_waking(p);


```

#### 2.2.尝试唤醒任务

```c
	smp_rmb();
	if (READ_ONCE(p->on_rq) && ttwu_runnable(p, wake_flags))
		goto unlock;


```

使用`smp_rmb`防止读操作乱序，以此来保证`p->on_rq`对应的读取操作不会被提前，真正执行任务唤醒的流程在`ttwu_runnable`函数中。

##### `ttwu_runnable`函数

```c
static int ttwu_runnable(struct task_struct *p, int wake_flags)
{
	struct rq_flags rf;
	struct rq *rq;
	int ret = 0;

	rq = __task_rq_lock(p, &rf);
	if (task_on_rq_queued(p)) {
		/* check_preempt_curr() may use rq clock */
		update_rq_clock(rq);
		ttwu_do_wakeup(rq, p, wake_flags, &rf);
		ret = 1;
	}
	__task_rq_unlock(rq, &rf);

	return ret;
}
```

这个函数的流程如下：

1.获取task所在rq的锁，这个锁是一个自旋锁；

2.如果task已经被放入到某个rq之中则更新rq中涉及到时间的各种字段；

3.尝试唤醒任务

接下来重点介绍`__task_rq_lock`、`update_rq_clock`、`ttwu_do_wakeup`这三个函数。

##### `__task_rq_lock`函数

```c
struct rq *__task_rq_lock(struct task_struct *p, struct rq_flags *rf)
	__acquires(rq->lock)
{
	struct rq *rq;

	lockdep_assert_held(&p->pi_lock);

	for (;;) {
		rq = task_rq(p);
		raw_spin_rq_lock(rq);
		if (likely(rq == task_rq(p) && !task_on_rq_migrating(p))) {
			rq_pin_lock(rq, rf);
			return rq;
		}
		raw_spin_rq_unlock(rq);

		while (unlikely(task_on_rq_migrating(p)))
			cpu_relax();
	}
}

```

考虑到task可能正在CPU之间移动，有可能会出现此时获取到的锁并不是task在移动到指定CPU的锁这种情况，所以代码使用一个`for`循环来处理来规避这个问题。通过`raw_spin_rq_lock`获取rq的锁，然后在保证task已经完成迁移并且获取到了迁移之后cpu的rq的锁才会返回，否则继续进行循环。

##### `update_rq_clock`函数

```c
void update_rq_clock(struct rq *rq)
{
	s64 delta;

	lockdep_assert_rq_held(rq);

	if (rq->clock_update_flags & RQCF_ACT_SKIP)
		return;

#ifdef CONFIG_SCHED_DEBUG
	if (sched_feat(WARN_DOUBLE_CLOCK))
		SCHED_WARN_ON(rq->clock_update_flags & RQCF_UPDATED);
	rq->clock_update_flags |= RQCF_UPDATED;
#endif

	delta = sched_clock_cpu(cpu_of(rq)) - rq->clock;
	if (delta < 0)
		return;
	rq->clock += delta;
	update_rq_clock_task(rq, delta);
}
```

这个函数用于更新rq中时间相关的字段，参数中的rq为任务所在CPU的rq，忽略函数中死锁检测（`lockdep_assert_rq_held`）、SCHED_DEBUG（`CONFIG_SCHED_DEBUG`宏定义包含的代码）相关的内容，函数的流程如下：

1.更新rq的时间，这个时间是系统启动之后与现在的时间间隔，单位是纳秒；

2.更新rq中其他的字段，主要在`update_rq_clock_task`函数中完成；



## 待办

- [ ] 整理任务`pi_lock`的作用；
