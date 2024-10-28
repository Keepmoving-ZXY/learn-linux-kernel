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

这种情况下首先调用`ttwu_state_match`函数进行状态检查，如果状态不匹配则跳转到`unlock`标签，在调用这个函数之前需要持有`pi_lock`锁、禁用当前CPU的中断、保存中断状态（仅在这个这个函数中很难看到`pi_lock`的作用，需要结合更多进程调度源码进行分析），代码如下：

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

#### 2.2.唤醒任务

```c
    smp_rmb();
    if (READ_ONCE(p->on_rq) && ttwu_runnable(p, wake_flags))
        goto unlock;
```

使用`smp_rmb`防止读操作乱序，以此来保证`p->on_rq`对应的读取操作不会被提前，真正执行任务唤醒的流程在`ttwu_runnable`函数中。若`ttwu_runnable`返回为1则task唤醒成功，跳转到`unlock`执行。

##### `ttwu_runnable`函数

该函数的内部函数调用拓扑如下所示：

```
ttwu_runnable:
    __task_rq_lock
    update_rq_clock
        update_rq_clock_task
            update_rq_clock_pelt
    ttwu_do_wakeup
        check_preempt_curr
            resched_curr
```

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

接下来重点关注`__task_rq_lock`、`update_rq_clock`、`ttwu_do_wakeup`这三个函数。

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

##### `update_rq_clock_task`函数

```c
static void update_rq_clock_task(struct rq *rq, s64 delta)
{
/*
 * In theory, the compile should just see 0 here, and optimize out the call
 * to sched_rt_avg_update. But I don't trust it...
 */
    s64 __maybe_unused steal = 0, irq_delta = 0;

#ifdef CONFIG_IRQ_TIME_ACCOUNTING
    irq_delta = irq_time_read(cpu_of(rq)) - rq->prev_irq_time;

    /*
     * Since irq_time is only updated on {soft,}irq_exit, we might run into
     * this case when a previous update_rq_clock() happened inside a
     * {soft,}irq region.
     *
     * When this happens, we stop ->clock_task and only update the
     * prev_irq_time stamp to account for the part that fit, so that a next
     * update will consume the rest. This ensures ->clock_task is
     * monotonic.
     *
     * It does however cause some slight miss-attribution of {soft,}irq
     * time, a more accurate solution would be to update the irq_time using
     * the current rq->clock timestamp, except that would require using
     * atomic ops.
     */
    if (irq_delta > delta)
        irq_delta = delta;

    rq->prev_irq_time += irq_delta;
    delta -= irq_delta;
#endif
#ifdef CONFIG_PARAVIRT_TIME_ACCOUNTING
    if (static_key_false((&paravirt_steal_rq_enabled))) {
        steal = paravirt_steal_clock(cpu_of(rq));
        steal -= rq->prev_steal_time_rq;

        if (unlikely(steal > delta))
            steal = delta;

        rq->prev_steal_time_rq += steal;
        delta -= steal;
    }
#endif

    rq->clock_task += delta;

#ifdef CONFIG_HAVE_SCHED_AVG_IRQ
    if ((irq_delta + steal) && sched_feat(NONTASK_CAPACITY))
        update_irq_load_avg(rq, irq_delta + steal);
#endif
    update_rq_clock_pelt(rq, delta);
}
```

忽略`CONFIG_HAVE_SCHED_AVG_IRQ`、`CONFIG_IRQ_TIME_ACCOUNTING`这两个宏启用的代码逻辑，这个函数增加了rq中任务执行占用的时间，这个时间值在某些情况下与`rq->clock`的值不一致，因为要考虑到虚拟化部分的逻辑（见`CONFIG_PARAVIRT_TIME_ACCOUNTING`宏启用的代码逻辑），随后调用`update_rq_clock_pelt`更新rq中pelt（[per-entity load tracking](https://docs.kernel.org/scheduler/schedutil.html)）相关的计数器。

##### `update_rq_clock_pelt`函数

```c
static inline void update_rq_clock_pelt(struct rq *rq, s64 delta)
{
    if (unlikely(is_idle_task(rq->curr))) {
        _update_idle_rq_clock_pelt(rq);
        return;
    }

    /*
     * When a rq runs at a lower compute capacity, it will need
     * more time to do the same amount of work than at max
     * capacity. In order to be invariant, we scale the delta to
     * reflect how much work has been really done.
     * Running longer results in stealing idle time that will
     * disturb the load signal compared to max capacity. This
     * stolen idle time will be automatically reflected when the
     * rq will be idle and the clock will be synced with
     * rq_clock_task.
     */

    /*
     * Scale the elapsed time to reflect the real amount of
     * computation
     */
    delta = cap_scale(delta, arch_scale_cpu_capacity(cpu_of(rq)));
    delta = cap_scale(delta, arch_scale_freq_capacity(cpu_of(rq)));

    rq->clock_pelt += delta;
}
```

忽略函数中对空闲任务的处理，这个函数根据CPU的capacity以及frequency对时间差（`delta`）进行缩放，将缩放结果更新到`clock_pelt`字段中。这个函数更新了pelt功能使用的字段，CPU的capacity的含义见[Capacity Aware Scheduling &#8212; The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-capacity.html)。

##### `ttwu_do_wakeup`函数

```c
static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags,
               struct rq_flags *rf)
{
    check_preempt_curr(rq, p, wake_flags);
    WRITE_ONCE(p->__state, TASK_RUNNING);
    trace_sched_wakeup(p);

#ifdef CONFIG_SMP
    if (p->sched_class->task_woken) {
        /*
         * Our task @p is fully woken up and running; so it's safe to
         * drop the rq->lock, hereafter rq is only used for statistics.
         */
        rq_unpin_lock(rq, rf);
        p->sched_class->task_woken(rq, p);
        rq_repin_lock(rq, rf);
    }

    if (rq->idle_stamp) {
        u64 delta = rq_clock(rq) - rq->idle_stamp;
        u64 max = 2*rq->max_idle_balance_cost;

        update_avg(&rq->avg_idle, delta);

        if (rq->avg_idle > max)
            rq->avg_idle = max;

        rq->wake_stamp = jiffies;
        rq->wake_avg_idle = rq->avg_idle / 2;

        rq->idle_stamp = 0;
    }
#endif
}
```

这个函数的流程如下：

1.检查当前rq中执行的任务是否可以被抢占，由`check_preempt_curr`函数实现；

2.设置任务状态为`TASK_RUNNING`；

3.调用调度类定义的`task_woken`实现；

4.更新CPU空闲时间统计。

检查rq中正在执行任务是否可以被抢占的逻辑在`check_preempt_curr`函数中详细说明，这里说明CPU空闲时间统计相关的逻辑。当rq处于空闲状态时会更新`idle_stamp`为进入空闲状态的时间戳，得到rq空闲时间之后更新rq的平均空闲时间，这个空闲时间是计算方式如下代码所示：

```c
static inline void update_avg(u64 *avg, u64 sample)
{
    s64 diff = sample - *avg;
    *avg += diff / 8;
}
```

为了防止平均空闲时间过长导致调度器一直向这个CPU中调度新的任务，需要限制rq的平均空闲时间最大值为`max_idle_balance_cost`的2倍。更新rq的平均空闲时间的同时也更新了`wake_stamp`、`wake_avg_idle`字段的值，这些字段都在调度器进行负载均衡过程中使用，例如某些CPU处于比较繁忙的状态而另外一些CPU处于空闲状态，就可以将忙碌CPU的rq中任务调度到空闲CPU中执行。

##### `check_preempt_curr`函数

```c
void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags)
{
    if (p->sched_class == rq->curr->sched_class)
        rq->curr->sched_class->check_preempt_curr(rq, p, flags);
    else if (sched_class_above(p->sched_class, rq->curr->sched_class))
        resched_curr(rq);

    /*
     * A queue event has occurred, and we're going to schedule.  In
     * this case, we can save a useless back to back clock update.
     */
    if (task_on_rq_queued(rq->curr) && test_tsk_need_resched(rq->curr))
        rq_clock_skip_update(rq);
}
```

若被唤醒的任务与rq中正在执行的任务使用同一个调度类（比如都是用CFS），则调用调度类自己定义的`check_preempt_curr`函数进行抢占检查；若使用不同的调度类，若被唤醒进程的调度类优先级比rq中正在运行的task使用的调度类高，则添加抢占正在运行的任务相关标识（`resched_curr`函数，下边详细介绍它的流程）。当rq中正在运行的任务在rq中并且它将会被抢占，这说明马上要进行任务调度，使用`rq_clock_skip_update`跳过在执行抢占过程中调度器时间相关字段更新。

##### `resched_curr`函数

```c
void resched_curr(struct rq *rq)
{
    struct task_struct *curr = rq->curr;
    int cpu;

    lockdep_assert_rq_held(rq);

    if (test_tsk_need_resched(curr))
        return;

    cpu = cpu_of(rq);

    if (cpu == smp_processor_id()) {
        set_tsk_need_resched(curr);
        set_preempt_need_resched();
        return;
    }

    if (set_nr_and_not_polling(curr))
        smp_send_reschedule(cpu);
    else
        trace_sched_wake_idle_without_ipi(cpu);
}
```

忽略`lockdep_assert_rq_held`、`trace_sched_wake_idle_without_ipi`函数，这个函数的整体流程如下：

1.若rq中正在执行的任务设置了被抢占的标志，则推出函数；

2.若被唤醒任务所在的rq所属的CPU与当前CPU是同一个CPU，则设置两个标志然后退出；

3.若不是同一个CPU，为rq中正在执行的任务设置被抢占标志；

4.若rq所属CPU不会主动轮询重新调度信息，则通过`smp_send_reschedule`函数通知；

#### 2.3. 任务放入wake list

```c
	if (smp_load_acquire(&p->on_cpu) &&
	    ttwu_queue_wakelist(p, task_cpu(p), wake_flags))
		goto unlock;

```

当被唤醒的任务还在处于另外一个CPU把它调出的过程中时，它的`on_cpu`字段不为0，此时想把它放到其他的CPU中执行，这个时候需要把这个任务放到即将执行任务的CPU的wake list，然后通过IPI中断通知执行这个任务的CPU继续进行处理。这个逻辑主要由`ttwu_queue_wakelist` 函数实现。

##### `ttwu_queue_wakelist`函数

```c
static bool ttwu_queue_wakelist(struct task_struct *p, int cpu, int wake_flags)
{
	if (sched_feat(TTWU_QUEUE) && ttwu_queue_cond(p, cpu)) {
		sched_clock_cpu(cpu); /* Sync clocks across CPUs */
		__ttwu_queue_wakelist(p, cpu, wake_flags);
		return true;
	}

	return false;
}
```

若定义了`CONFIG_HAVE_UNSTABLE_SCHED_CLOCK`宏，`sched_clock_cpu`才会进行时间同步操作，否则这个这个函数仅仅是返回了一下当前的时间并且函数参数没有任何意义。这里仅关注`CONFIG_HAVE_UNSTABLE_SCHED_LOCK`宏未定义的情况，此时`sched_clock_cpu`调用的是`native_sched_clock`函数，这个函数仅仅返回了当前的时间（以纳秒为单位）。接下来关注`__ttwu_queue_wakelist`函数。

##### `__ttwu_queue_wakelist`函数

```c
static void __ttwu_queue_wakelist(struct task_struct *p, int cpu, int wake_flags)
{
	struct rq *rq = cpu_rq(cpu);

	p->sched_remote_wakeup = !!(wake_flags & WF_MIGRATED);

	WRITE_ONCE(rq->ttwu_pending, 1);
	__smp_call_single_queue(cpu, &p->wake_entry.llist);
}
```

函数流程如下：

1.更新`sched_remote_wakeup`字段，这个字段为1表示任务需要在CPU之间进行转移；

2.更新`ttwu_pending`字段的值为1，使用`WRITE_ONCE`保证缓存一致性；

3.调用`__smp_call_single_queue`函数向`cpu`参数指定的CPU触发IPI中断，通知这个CPU处理任务唤醒请求；

`__smp_call_single_queue`函数将被唤醒的任务加入到`cpu`参数执行的CPU中的队列中（`call_single_queue`，为per-cpu变量），若这个队列在新的任务添加之前为空则向它发送IPI中断。这里注意到当这个队列在添加之前不为空时不需要发送IPI中断，这是因为这种情况下CPU正在处理队列中的内容，无需主动通知。



## 待办

- [ ] 整理任务`pi_lock`的作用；
