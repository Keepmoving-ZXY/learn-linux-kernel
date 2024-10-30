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

1.若rq中正在执行的任务设置了被抢占的标志，则退出函数；

2.若被唤醒任务所在的rq所属的CPU与当前CPU是同一个CPU，则设置两个标志然后退出；

3.若不是同一个CPU，为rq中正在执行的任务设置被抢占标志；

4.若rq所属CPU不会主动轮询重新调度信息，则通过`smp_send_reschedule`函数通知；

#### 2.3. 任务放入wake list

```c
    smp_acquire__after_ctrl_dep();
    WRITE_ONCE(p->__state, TASK_WAKING);

    if (smp_load_acquire(&p->on_cpu) &&
        ttwu_queue_wakelist(p, task_cpu(p), wake_flags))
        goto unlock;
```

`smp_acquire__after_ctrl_dep`保证先加载`p->on_rq`随后加载`p->on_cpu`，这两个字段的加载的顺序不会与代码中的顺序相反。随后将任务的状态设置为`TASK_WAKING`，这里使用了`WRITE_ONCE`来保证缓存一致性。

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

#### 2.4.寻找新的CPU运行任务

```c
    /*
     * If the owning (remote) CPU is still in the middle of schedule() with
     * this task as prev, wait until it's done referencing the task.
     *
     * Pairs with the smp_store_release() in finish_task().
     *
     * This ensures that tasks getting woken will be fully ordered against
     * their previous state and preserve Program Order.
     */
    smp_cond_load_acquire(&p->on_cpu, !VAL);

    cpu = select_task_rq(p, p->wake_cpu, wake_flags | WF_TTWU);
    if (task_cpu(p) != cpu) {
        if (p->in_iowait) {
            delayacct_blkio_end(p);
            atomic_dec(&task_rq(p)->nr_iowait);
        }

        wake_flags |= WF_MIGRATED;
        psi_ttwu_dequeue(p);
        set_task_cpu(p, cpu);
    }
```

`smp_cond_load_acquire`等待task的`on_cpu`为0，并且保证在这行代码之前的内存操作都不会在这行代码之后执行，`2.3`中解释了在什么情况下`on_cpu`字段的值不为0。

`select_task_rq`为被唤醒的任务选择一个CPU执行，当选中的CPU与任务指定的CPU不是同一个时为`wake_flags`添加`WF_MIGRATED`标识、调用`set_task_cpu`执行更新逻辑，忽略`p->in_iowait`中的逻辑以及`psi_ttwu_queue`函数。

接下来重点关注`select_task_rq`以及`set_task_cpu`中的逻辑。

##### `select_task_rq`函数

```c
static inline
int select_task_rq(struct task_struct *p, int cpu, int wake_flags)
{
    lockdep_assert_held(&p->pi_lock);

    if (p->nr_cpus_allowed > 1 && !is_migration_disabled(p))
        cpu = p->sched_class->select_task_rq(p, cpu, wake_flags);
    else
        cpu = cpumask_any(p->cpus_ptr);

    /*
     * In order not to call set_task_cpu() on a blocking task we need
     * to rely on ttwu() to place the task on a valid ->cpus_ptr
     * CPU.
     *
     * Since this is common to all placement strategies, this lives here.
     *
     * [ this allows ->select_task() to simply return task_cpu(p) and
     *   not worry about this generic constraint ]
     */
    if (unlikely(!is_cpu_allowed(p, cpu)))
        cpu = select_fallback_rq(task_cpu(p), p);

    return cpu;
}
```

若任务允许运行的CPU数量大于1、允许任务在CPU之间转移，调用调度类定义的`select_task_rq`返回一个CPU用于运行任务；反之则从任务允许执行的CPU中选择一个CPU运行任务。在选择一个CPU之后需要判断是否允许使用被选中的CPU，如果不允许则调用`select_fallback_rq`函数重新选择可以运行任务的CPU。

接下来重点关注`is_cpu_allowed`以及`select_fallback_rq`函数。

##### `is_cpu_allowed`函数

```c
static inline bool is_cpu_allowed(struct task_struct *p, int cpu)
{
    /* When not in the task's cpumask, no point in looking further. */
    if (!cpumask_test_cpu(cpu, p->cpus_ptr))
        return false;

    /* migrate_disabled() must be allowed to finish. */
    if (is_migration_disabled(p))
        return cpu_online(cpu);

    /* Non kernel threads are not allowed during either online or offline. */
    if (!(p->flags & PF_KTHREAD))
        return cpu_active(cpu) && task_cpu_possible(cpu, p);

    /* KTHREAD_IS_PER_CPU is always allowed. */
    if (kthread_is_per_cpu(p))
        return cpu_online(cpu);

    /* Regular kernel threads don't get to stay during offline. */
    if (cpu_dying(cpu))
        return false;

    /* But are allowed during online. */
    return cpu_online(cpu);
}
```

`cpu_online`的含义为CPU是否处于可用状态，`cpu_active`的含义为CPU是否可以接受任务调度，`cpu_dying`的含义为CPU是否正在下线。注意CPU在可用状态下不一定意味着CPU可以用接受任务调度，CPU状态含义以及转换转换逻辑参加内核相关源码。

这个函数主要用于检查CPU是否可用，从如下几个方面进行检查：

1.CPU是否在允许的CPU之中；

2.若任务不允许转移，则返回CPU是否处于可用状态；

3.若任务是一个内核线程，返回CPU是否可以接受任务调度并且任务可以运行在这个CPU之中；

4.若任务是在CPU中运行的内核线程（每个CPU上都运行一个内核线程），则返回CPU是否可用状态；

5.若CPU正在下线过程中则返回False；

6.返回CPU是否处于可用状态；

##### `select_fallback_rq`函数

```c
static int select_fallback_rq(int cpu, struct task_struct *p)
{
    int nid = cpu_to_node(cpu);
    const struct cpumask *nodemask = NULL;
    enum { cpuset, possible, fail } state = cpuset;
    int dest_cpu;

    /*
     * If the node that the CPU is on has been offlined, cpu_to_node()
     * will return -1. There is no CPU on the node, and we should
     * select the CPU on the other node.
     */
    if (nid != -1) {
        nodemask = cpumask_of_node(nid);

        /* Look for allowed, online CPU in same node. */
        for_each_cpu(dest_cpu, nodemask) {
            if (is_cpu_allowed(p, dest_cpu))
                return dest_cpu;
        }
    }

    for (;;) {
        /* Any allowed, online CPU? */
        for_each_cpu(dest_cpu, p->cpus_ptr) {
            if (!is_cpu_allowed(p, dest_cpu))
                continue;

            goto out;
        }

        /* No more Mr. Nice Guy. */
        switch (state) {
        case cpuset:
            if (cpuset_cpus_allowed_fallback(p)) {
                state = possible;
                break;
            }
            fallthrough;
        case possible:
            /*
             * XXX When called from select_task_rq() we only
             * hold p->pi_lock and again violate locking order.
             *
             * More yuck to audit.
             */
            do_set_cpus_allowed(p, task_cpu_possible_mask(p));
            state = fail;
            break;
        case fail:
            BUG();
            break;
        }
    }

out:
    if (state != cpuset) {
        /*
         * Don't tell them about moving exiting tasks or
         * kernel threads (both mm NULL), since they never
         * leave kernel.
         */
        if (p->mm && printk_ratelimit()) {
            printk_deferred("process %d (%s) no longer affine to cpu%d\n",
                    task_pid_nr(p), p->comm, cpu);
        }
    }

    return dest_cpu;
}
```

当CPU所在的numa节点在线时，尝试从这个numa节点中选出另外一个可用的CPU，若无法选出一个可用的CPU进入`for`循环之中。在这个循环之中首先尝试从任务允许的CPU之中选出一个可用的CPU，若无法找到这样的CPU则考虑从cgroup中的cpuset中寻找一个可用的CPU，对应的逻辑在`cpuset_cpus_allowed_fallback`之中，无论有没有从cpuset中找到合适的CPU，都要执行`__do_set_cpus_allowed`函数。当任务作为一个进程并且这个任务无法调度到指定的CPU中运行时，打印日志随后返回。

接下来关注`cpuset_cpus_allowed_fallback`、`__do_set_cpus_allowed`这两个函数中的内容。

##### `cpuset_cpus_allowed_fallback`函数

```c
bool cpuset_cpus_allowed_fallback(struct task_struct *tsk)
{
    const struct cpumask *possible_mask = task_cpu_possible_mask(tsk);
    const struct cpumask *cs_mask;
    bool changed = false;

    rcu_read_lock();
    cs_mask = task_cs(tsk)->cpus_allowed;
    if (is_in_v2_mode() && cpumask_subset(cs_mask, possible_mask)) {
        do_set_cpus_allowed(tsk, cs_mask);
        changed = true;
    }
    rcu_read_unlock();

    /*
     * We own tsk->cpus_allowed, nobody can change it under us.
     *
     * But we used cs && cs->cpus_allowed lockless and thus can
     * race with cgroup_attach_task() or update_cpumask() and get
     * the wrong tsk->cpus_allowed. However, both cases imply the
     * subsequent cpuset_change_cpumask()->set_cpus_allowed_ptr()
     * which takes task_rq_lock().
     *
     * If we are called after it dropped the lock we must see all
     * changes in tsk_cs()->cpus_allowed. Otherwise we can temporary
     * set any mask even if it is not right from task_cs() pov,
     * the pending set_cpus_allowed_ptr() will fix things.
     *
     * select_fallback_rq() will fix things ups and set cpu_possible_mask
     * if required.
     */
    return changed;
}
```

`task_cpu_possible_mask`返回系统中所有可能可用的CPU，`cs_mask`为cgroup cpuset中的所有可用的CPU，若cpuset中的CPU为系统可能可用的CPU中的一个子集，那么调用`do_set_cpus_allowed`函数修改任务允许使用的CPU。`do_set_cpus_allowed`函数中调用了任务使用的调度类定义的诸多方法，这个函数调用了`__do_set_cpus_allowed`函数并将第三个参数设置为0，下面关注`__do_set_cpus_allowed`这个函数。

##### `__do_set_cpus_allowed`函数

```c
static void
__do_set_cpus_allowed(struct task_struct *p, const struct cpumask *new_mask, u32 flags)
{
    struct rq *rq = task_rq(p);
    bool queued, running;

    /*
     * This here violates the locking rules for affinity, since we're only
     * supposed to change these variables while holding both rq->lock and
     * p->pi_lock.
     *
     * HOWEVER, it magically works, because ttwu() is the only code that
     * accesses these variables under p->pi_lock and only does so after
     * smp_cond_load_acquire(&p->on_cpu, !VAL), and we're in __schedule()
     * before finish_task().
     *
     * XXX do further audits, this smells like something putrid.
     */
    if (flags & SCA_MIGRATE_DISABLE)
        SCHED_WARN_ON(!p->on_cpu);
    else
        lockdep_assert_held(&p->pi_lock);

    queued = task_on_rq_queued(p);
    running = task_current(rq, p);

    if (queued) {
        /*
         * Because __kthread_bind() calls this on blocked tasks without
         * holding rq->lock.
         */
        lockdep_assert_rq_held(rq);
        dequeue_task(rq, p, DEQUEUE_SAVE | DEQUEUE_NOCLOCK);
    }
    if (running)
        put_prev_task(rq, p);

    p->sched_class->set_cpus_allowed(p, new_mask, flags);

    if (queued)
        enqueue_task(rq, p, ENQUEUE_RESTORE | ENQUEUE_NOCLOCK);
    if (running)
        set_next_task(rq, p);
}
```

忽略代码中`lockdep_assert_held`、`lockdep_assert_rq_held`函数，若禁止了任务在CPU之间转移并且当任务不在任何CPU中时会产生一条警告日志。若任务在某个rq之中，则先后执行`dequeue_task`、`enqueue_task`函数，若任务为当前rq中正在运行的任务则先后执行`put_prev_task`以及`set_next_task`函数。当这个任务正在rq队列之中时，这个修改会影响调度过程中的许多细节，因此需要将任务从rq中出队然后入；当这个任务为正在运行的任务时，修改任务可以使用的CPU会影响调度类中的许多细节，因此需要先执行调度器特有的`put_prev_task`以及`set_next_task`将任务先后放到调度类的就绪队列、执行队列之中。`put_prev_task`与`set_next_task`调用调度类的`put_prev_task`方法以及`set_next_task`方法，接下来关注`dequeue_task`、`enqueue_task`函数。

##### `dequeue_task`函数

```c
static inline void dequeue_task(struct rq *rq, struct task_struct *p, int flags)
{
    if (sched_core_enabled(rq))
        sched_core_dequeue(rq, p, flags);

    if (!(flags & DEQUEUE_NOCLOCK))
        update_rq_clock(rq);

    if (!(flags & DEQUEUE_SAVE)) {
        sched_info_dequeue(rq, p);
        psi_dequeue(p, flags & DEQUEUE_SLEEP);
    }

    uclamp_rq_dec(rq, p);
    p->sched_class->dequeue_task(rq, p, flags);
}
```

这个函数会调用调度类的`dequeue_task`方法，若未指定`DEQUEUE_NOCLOCK`则更新rq的时间相关字段，若启用了[core schedule]([Core Scheduling &#8212; The Linux Kernel documentation](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html))特性则调用`sched_core_dequeue`更新相关字段，忽略`sched_info_dequeue`、`psi_dequeue`、`uclamp_rq_dec`这三个函数。

`update_rq_clock`函数的流程之前已经提及，`sched_core_dequeue`函数内容如下：

```c
void sched_core_dequeue(struct rq *rq, struct task_struct *p, int flags)
{
    rq->core->core_task_seq++;

    if (sched_core_enqueued(p)) {
        rb_erase(&p->core_node, &rq->core_tree);
        RB_CLEAR_NODE(&p->core_node);
    }

    /*
     * Migrating the last task off the cpu, with the cpu in forced idle
     * state. Reschedule to create an accounting edge for forced idle,
     * and re-examine whether the core is still in forced idle state.
     */
    if (!(flags & DEQUEUE_SAVE) && rq->nr_running == 1 &&
        rq->core->core_forceidle_count && rq->curr == rq->idle)
        resched_curr(rq);
}
```

这个函数将任务从红黑树中移除，在强制rq所在CPU为空闲状态、当前运行的任务为idle任务时调用`resched_curr`将它转移到其他的CPU中去。

##### `enqueue_task`函数

```c
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
    if (!(flags & ENQUEUE_NOCLOCK))
        update_rq_clock(rq);

    if (!(flags & ENQUEUE_RESTORE)) {
        sched_info_enqueue(rq, p);
        psi_enqueue(p, flags & ENQUEUE_WAKEUP);
    }

    uclamp_rq_inc(rq, p);
    p->sched_class->enqueue_task(rq, p, flags);

    if (sched_core_enabled(rq))
        sched_core_enqueue(rq, p);
}
```

忽略`sched_info_enqueue`、`psi_enqueue`、`uclamp_rq_inc`这三个函数，这个函数调用了调度类特有的`enqueue_task`方法，若未指定`ENQUEUE_NOCLOCK`则更新rq时间相关字段，若启用了[core schedule]([Core Scheduling — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html))特性则调用`sched_core_enqueue`函数更新相关字段，`sched_core_enqueue`函数内容如下：

```c
void sched_core_enqueue(struct rq *rq, struct task_struct *p)
{
    rq->core->core_task_seq++;

    if (!p->core_cookie)
        return;

    rb_add(&p->core_node, &rq->core_tree, rb_sched_core_less);
}
```

这个函数将任务加入到红黑树中之中，红黑树中的节点中应该保存了允许在这个CPU中运行的任务，详见内核的`core_schedule`特性。

##### `set_task_cpu`函数

```c
void set_task_cpu(struct task_struct *p, unsigned int new_cpu)
{
#ifdef CONFIG_SCHED_DEBUG
    unsigned int state = READ_ONCE(p->__state);

    /*
     * We should never call set_task_cpu() on a blocked task,
     * ttwu() will sort out the placement.
     */
    WARN_ON_ONCE(state != TASK_RUNNING && state != TASK_WAKING && !p->on_rq);

    /*
     * Migrating fair class task must have p->on_rq = TASK_ON_RQ_MIGRATING,
     * because schedstat_wait_{start,end} rebase migrating task's wait_start
     * time relying on p->on_rq.
     */
    WARN_ON_ONCE(state == TASK_RUNNING &&
             p->sched_class == &fair_sched_class &&
             (p->on_rq && !task_on_rq_migrating(p)));

#ifdef CONFIG_LOCKDEP
    /*
     * The caller should hold either p->pi_lock or rq->lock, when changing
     * a task's CPU. ->pi_lock for waking tasks, rq->lock for runnable tasks.
     *
     * sched_move_task() holds both and thus holding either pins the cgroup,
     * see task_group().
     *
     * Furthermore, all task_rq users should acquire both locks, see
     * task_rq_lock().
     */
    WARN_ON_ONCE(debug_locks && !(lockdep_is_held(&p->pi_lock) ||
                      lockdep_is_held(__rq_lockp(task_rq(p)))));
#endif
    /*
     * Clearly, migrating tasks to offline CPUs is a fairly daft thing.
     */
    WARN_ON_ONCE(!cpu_online(new_cpu));

    WARN_ON_ONCE(is_migration_disabled(p));
#endif

    trace_sched_migrate_task(p, new_cpu);

    if (task_cpu(p) != new_cpu) {
        if (p->sched_class->migrate_task_rq)
            p->sched_class->migrate_task_rq(p, new_cpu);
        p->se.nr_migrations++;
        rseq_migrate(p);
        perf_event_task_migrate(p);
    }

    __set_task_cpu(p, new_cpu);
}
```

忽略`CONFIG_SCHED_DEBUG`、`CONFIG_LOCKDEP`这两个宏启用的代码以及`trace_sched_migrate_task`、`perf_event_task_migrate`这两个函数。当为任务选择的新的CPU与任务指定运行的CPU不一致时，需要调用调度类特定的`migrate_task_rq`方法，更新`Restartable Sequences`特性在面对任务在CPU之间移动时的需要记录的字段（具体内容见`resq_migrate`函数说明），调用`__set_task_cpu`更新任务使用的CPU。接下来关注`rseq_migrate`、`__set_task_cpu`函数内容。

##### `rseq_migrate`函数

```c
/* rseq_migrate() requires preemption to be disabled. */
static inline void rseq_migrate(struct task_struct *t)
{
    __set_bit(RSEQ_EVENT_MIGRATE_BIT, &t->rseq_event_mask);
    rseq_set_notify_resume(t);
}
```

其中`rseq_set_notify_resume`实现为：

```c
static inline void rseq_set_notify_resume(struct task_struct *t)
{
    if (t->rseq)
        set_tsk_thread_flag(t, TIF_NOTIFY_RESUME);
}
```

这个函数仅仅设置两个标记位，提醒`Restartable Sequence`实现中的其他函数这个任务运行的CPU会发生改变，`Restartable Sequence`这个功能允许用户态中的一小段代码访问per-cpu的数据结构而不需要引入锁开销。

##### `__set_task_cpu`函数

```c
static inline void __set_task_cpu(struct task_struct *p, unsigned int cpu)
{
    set_task_rq(p, cpu);
#ifdef CONFIG_SMP
    /*
     * After ->cpu is set up to a new value, task_rq_lock(p, ...) can be
     * successfully executed on another CPU. We must ensure that updates of
     * per-task data have been completed by this moment.
     */
    smp_wmb();
    WRITE_ONCE(task_thread_info(p)->cpu, cpu);
    p->wake_cpu = cpu;
#endif
}
```

使用`set_task_rq`更新任务所在的rq以及相关字段，接下来为任务设置新的运行CPU以及唤醒CPU，设置任务运行的新的CPU时使用`WRITE_ONCE`保证缓存一致性，在更新这两个信息之前使用写屏障来保证之前的写操作都已经完成。使用写屏障的原因在于一旦`cpu`字段的值被更新，会激活调度器中的其他函数，这些函数可能会使用在设置`cpu`字段之前更新的字段的值，所以要保证在更新`cpu`字段之前其他的字段更新已经完成。接下来关注`set_task_rq`函数。

##### `set_task_rq`函数

```c
/* Change a task's cfs_rq and parent entity if it moves across CPUs/groups */
static inline void set_task_rq(struct task_struct *p, unsigned int cpu)
{
#if defined(CONFIG_FAIR_GROUP_SCHED) || defined(CONFIG_RT_GROUP_SCHED)
    struct task_group *tg = task_group(p);
#endif

#ifdef CONFIG_FAIR_GROUP_SCHED
    set_task_rq_fair(&p->se, p->se.cfs_rq, tg->cfs_rq[cpu]);
    p->se.cfs_rq = tg->cfs_rq[cpu];
    p->se.parent = tg->se[cpu];
    p->se.depth = tg->se[cpu] ? tg->se[cpu]->depth + 1 : 0;
#endif

#ifdef CONFIG_RT_GROUP_SCHED
    p->rt.rt_rq  = tg->rt_rq[cpu];
    p->rt.parent = tg->rt_se[cpu];
#endif
}
```

仅考虑`CONFIG_FAIR_GROUP_SCHED`宏启动的代码，这个宏表示CFS调度器调度的基本单位为进程组而非单个进程。这段代码更新了任务所在的rq、父`sched_entity`（`p->se.parent`）、深度（`se.depth`）三个字段，后两个字段的含义与开启了进程组调度之后进程的组织方式相关。当开启了进程组调度特性之后，调度器处理的任务按照层级结构关联起来，每个任务的`se.parent`字段指向任务所在组的`sched_entity`实例，`se.depth`字段的值为任务所在的层级。

#### 2.5.唤醒任务

第`2.4`中的逻辑执行完成之后，直接进入这部分的代码逻辑，代码如下：

```c
	ttwu_queue(p, cpu, wake_flags);
unlock:
	raw_spin_unlock_irqrestore(&p->pi_lock, flags);
out:
	if (success)
		ttwu_stat(p, task_cpu(p), wake_flags);
	preempt_enable();

	return success;
```

忽略`ttwu_stat`函数，这段代码调用`ttwu_queue`函数将任务p放入到wake list之中或者直接唤醒一个任务，随后释放`pi_lock`锁并恢复软中断，最后允许任务抢占。接下来关注`ttwu_queue`函数实现。

##### `ttwu_queue`函数

```c
static void ttwu_queue(struct task_struct *p, int cpu, int wake_flags)
{
	struct rq *rq = cpu_rq(cpu);
	struct rq_flags rf;

	if (ttwu_queue_wakelist(p, cpu, wake_flags))
		return;

	rq_lock(rq, &rf);
	update_rq_clock(rq);
	ttwu_do_activate(rq, p, wake_flags, &rf);
	rq_unlock(rq, &rf);
}
```

这个函数首先尝试将任务放入到wake list之中，若失败则更新rq的时间相关字段随后开始激活这个任务，更新rq的时间相关字段以及激活任务是在持有rq的锁的情况下进行的。这两个函数（`ttwu_queue_wakelist`、`update_rq_clock`）在前边提到过，接下来关注`ttwu_do_active`函数。

##### `ttwu_do_active`函数

```c
static void
ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags,
		 struct rq_flags *rf)
{
	int en_flags = ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK;

	lockdep_assert_rq_held(rq);

	if (p->sched_contributes_to_load)
		rq->nr_uninterruptible--;

#ifdef CONFIG_SMP
	if (wake_flags & WF_MIGRATED)
		en_flags |= ENQUEUE_MIGRATED;
	else
#endif
	if (p->in_iowait) {
		delayacct_blkio_end(p);
		atomic_dec(&task_rq(p)->nr_iowait);
	}

	activate_task(rq, p, en_flags);
	ttwu_do_wakeup(rq, p, wake_flags, rf);
}


```

忽略`lockdep_assert_rq_held`函数以及当`p->iowait`为真时执行的代码，这个函数调用`activate_task`将任务加入到rq之中、调用`ttwu_do_wakeup`唤醒这个任务。同时若这个任务对系统负载有贡献，即这个任务会占用CPU资源，将rq中不可中断的计数器递减（`nr_uninterruptible`）。`ttwu_do_wakeup`函数的内容在前边已经提到，接下来关注`activate_task`函数的内容。

##### `activate_task`函数

```c
void activate_task(struct rq *rq, struct task_struct *p, int flags)
{
	if (task_on_rq_migrating(p))
		flags |= ENQUEUE_MIGRATED;

	enqueue_task(rq, p, flags);

	p->on_rq = TASK_ON_RQ_QUEUED;
}

```

若这个任务在CPU中移动，则此时已经完成了移动，向`flags`中添加`ENQUEUE_MIGRATED`标识标记此种情况，随后调用`enqueue_task`将任务放入到rq之中，最后将任务设置为已经在rq中（`TASK_ON_RQ_QUEUED`）。

## 待办

- [ ] 整理任务`pi_lock`的作用；
