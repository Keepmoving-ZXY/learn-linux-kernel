## 简介

在linux任务调度过程中，当为某个CPU中rq上正在运行的任务设置了`TIF_NEED_RESCHED`标志，意味着这个任务即将被其他任务抢占，即另外一个任务替代这个任务占用CPU资源，完成这个动作的函数是`schedule`函数。在`try_to_wake_up`中会将rq中正在执行的任务加上`TIF_NEED_RESCHED`标志，另外一个流程是定时器由驱动的rq的时间更新流程，在这个流程中可能会为rq中正在运行的任务添加`TIF_NEED_RESCHED`标记。在记录`schedule`函数之前，先记录高精度定时器是如何影响任务调度过程。

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

这个函数在持有rq锁的前提下，更新rq的时间相关字段、调用rq所在CPU中正在运行的任务使用的调度类的`task_tick`方法。注意到这个函数返回的是`HRTIMER_NORESTART`，意味着这个定时器不会自动启用，在调度类中才会继续启动这个定时器，例如CFS的`enqueue_task_fair`函数。在执行调度类的`task_tick`方法过程中会为当前正在运行的任务设置`TIF_NEED_RESCHED`标记，然后内核其他部分代码执行过程中调用`schedule`函数触发一次任务切换。

### 2.任务切换

#### `schedule`函数

```c
asmlinkage __visible void __sched schedule(void)
{
    struct task_struct *tsk = current;

    sched_submit_work(tsk);
    do {
        preempt_disable();
        __schedule(SM_NONE);
        sched_preempt_enable_no_resched();
    } while (need_resched());
    sched_update_worker(tsk);
}
```

忽略`sched_submit_work`以及`submit_update_worker`两个函数，这个函数循环调用`__schedule`函数直到当前运行的任务没有标记`TIF_NEED_RESCHED`标志，在执行`__schedule`函数过程中要暂时进程任务抢占。注意到这里使用了一个`while`循环而非调用一次，这是考虑到优先级更高的任务抢占马上要执行的任务这种情况，即新的任务马上要开始执行但优先级更高的任务到来，内核将新的任务重新设置`TIF_NEED_RESCHED`标志，这个时候如果不继续进行循环将会导致优先级更高的任务得不到执行。接下来关注`__schedule`函数。

#### `__schedule`函数

```c
    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;

    schedule_debug(prev, !!sched_mode);

    if (sched_feat(HRTICK) || sched_feat(HRTICK_DL))
        hrtick_clear(rq);

    local_irq_disable();
    rcu_note_context_switch(!!sched_mode);
```

忽略`schedule_debug`、`rcu_note_context_switch`函数中的内容，这些代码禁用中断、停止rq中高精度定时器运行，在执行任务切换过程中高精度定时器到期触发的函数可能会对任务切换过程造成干扰，所以要停止高精度定时器。

```c
    /*
     * Make sure that signal_pending_state()->signal_pending() below
     * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
     * done by the caller to avoid the race with signal_wake_up():
     *
     * __set_current_state(@state)        signal_wake_up()
     * schedule()                  set_tsk_thread_flag(p, TIF_SIGPENDING)
     *                      wake_up_state(p, state)
     *   LOCK rq->lock                LOCK p->pi_state
     *   smp_mb__after_spinlock()            smp_mb__after_spinlock()
     *     if (signal_pending_state())        if (p->state & @state)
     *
     * Also, the membarrier system call requires a full memory barrier
     * after coming from user-space, before storing to rq->curr.
     */
    rq_lock(rq, &rf);
    smp_mb__after_spinlock();

    /* Promote REQ to ACT */
    rq->clock_update_flags <<= 1;
    update_rq_clock(rq);
```

读写操作乱序可能会导致任务状态检测先于任务状态设置发生，这可能会导致`__schedule`函数与`signal_wake_up`函数产生冲突，所以在获取rq锁之后使用添加内存屏障保证读写之间的顺序（`smp_mb__after_spinlock`）。随后更新rq的时间并且标记rq中的时间更新操作已经执行。

```c
    switch_count = &prev->nivcsw;

    /*
     * We must load prev->state once (task_struct::state is volatile), such
     * that we form a control dependency vs deactivate_task() below.
     */
    prev_state = READ_ONCE(prev->__state);
    if (!(sched_mode & SM_MASK_PREEMPT) && prev_state) {
        if (signal_pending_state(prev_state, prev)) {
            WRITE_ONCE(prev->__state, TASK_RUNNING);
        } else {
            prev->=
                (prev_state & TASK_UNINTERRUPTIBLE) &&
                !(prev_state & TASK_NOLOAD) &&
                !(prev_state & TASK_FROZEN);

            if (prev->sched_contributes_to_load)
                rq->nr_uninterruptible++;

            /*
             * __schedule()            ttwu()
             *   prev_state = prev->state;    if (p->on_rq && ...)
             *   if (prev_state)            goto out;
             *     p->on_rq = 0;          smp_acquire__after_ctrl_dep();
             *                  p->state = TASK_WAKING
             *
             * Where __schedule() and ttwu() have matching control dependencies.
             *
             * After this, schedule() must not care about p->state any more.
             */
            deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

            if (prev->in_iowait) {
                atomic_inc(&rq->nr_iowait);
                delayacct_blkio_start();
            }
        }
        switch_count = &prev->nvcsw;
    }
```

在将被抢占的任务不再处于正在运行状态、没有禁止任务抢占的情况下，若任务有待处理的信号则将任务设置为正在运行状态，否则这个任务是不可中断的任务。不可被中断的任务含义是此时任务处于休眠状态并且还不能被信号唤醒，当一个任务在执行磁盘io或者DMA操作时任务不可中断，忽略`prev->in_iowait`为真时执行的代码，此时若任务对系统负载有影响则更新rq中不可中断任务计数器，此时任务虽然不可中断但是依然处于睡眠状态，调用`deactivate_task`函数将任务从rq中移除，`deactivate_task`函数的具体流程在`task_wake_up.md`文件中记录。

```c
    next = pick_next_task(rq, prev, &rf);
    clear_tsk_need_resched(prev);
    clear_preempt_need_resched();
#ifdef CONFIG_SCHED_DEBUG
    rq->last_seen_need_resched_ns = 0;
#endif
```

调用`pick_next_task`函数选择接下来运行的任务，这个函数的流程在后边详细记录，随后清空被抢占任务的`TIF_NEED_RESCHED`标记、清空运行这些代码的cpu上的`PREEMPT_NEED_RESCHED`标记。

```c
    if (likely(prev != next)) {
        rq->nr_switches++;
        /*
         * RCU users of rcu_dereference(rq->curr) may not see
         * changes to task_struct made by pick_next_task().
         */
        RCU_INIT_POINTER(rq->curr, next);
        /*
         * The membarrier system call requires each architecture
         * to have a full memory barrier after updating
         * rq->curr, before returning to user-space.
         *
         * Here are the schemes providing that barrier on the
         * various architectures:
         * - mm ? switch_mm() : mmdrop() for x86, s390, sparc, PowerPC.
         *   switch_mm() rely on membarrier_arch_switch_mm() on PowerPC.
         * - finish_lock_switch() for weakly-ordered
         *   architectures where spin_unlock is a full barrier,
         * - switch_to() for arm64 (weakly-ordered, spin_unlock
         *   is a RELEASE barrier),
         */
        ++*switch_count;

        migrate_disable_switch(rq, prev);
        psi_account_irqtime(rq, prev, next);
        psi_sched_switch(prev, next, !task_on_rq_queued(prev));

        trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next, prev_state);

        /* Also unlocks the rq: */
        rq = context_switch(rq, prev, next, &rf);
    } else {
        rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);

        rq_unpin_lock(rq, &rf);
        __balance_callbacks(rq);
        raw_spin_rq_unlock_irq(rq);
    }
```

当即将运行的任务与被抢占的任务相同时，忽略`rq_unpin_lock`函数，更新rq的`clock_update_flags`来表示rq的时间相关字段已经更新，调用任务负载均衡功能相关的回调函数、释放rq锁并且恢复中断。

当即将运行的任务与被抢占任务不同是一个任务时，首先使用`RCU`机制更新rq正在运行的任务为即将运行的任务、递增`switch_count`、`nr_switches`等统计字段，调用`migrate_disable_switch`函数禁止任务在cpu之间移动，调用`context_switch`执行任务切换。忽略`psi_account_irqtime`、`psi_sched_switch`、`trace_sched_switch`函数内容，在后边详细记录`migrate_disable_switch`、`context_switch`函数内容。

#### `pick_next_task`函数

这个函数是开启[core scheduling](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html)特性之后的`pick_next_task`函数实现，这个实现比不启用`core scheduling`特性的实现要复杂许多，其中一个主要的概念是core cookie。每个任务的task_struct结构中都有一个叫做core_cookie的字段，这个字段存储一个数值，只有具有相同的core_cookie字段值的任务才能够在同一个物理cpu之中执行，这样可以防止一部分的侧信道攻击。

```c
    if (!sched_core_enabled(rq))
        return __pick_next_task(rq, prev, rf);

    cpu = cpu_of(rq);

    /* Stopper task is switching into idle, no need core-wide selection. */
    if (cpu_is_offline(cpu)) {
        /*
         * Reset core_pick so that we don't enter the fastpath when
         * coming online. core_pick would already be migrated to
         * another cpu during offline.
         */
        rq->core_pick = NULL;
        return __pick_next_task(rq, prev, rf);
    }
```

这部分代码是函数最开始执行的代码逻辑，考虑两种比较简单的情况，若`core_schedule`特性未启用或者参数中rq所在的cpu无法使用，则调用`__pick_next_task`函数返回下一个可以执行的任务，后边会详细记录这个函数的流程。

```c
    /*
     * If there were no {en,de}queues since we picked (IOW, the task
     * pointers are all still valid), and we haven't scheduled the last
     * pick yet, do so now.
     *
     * rq->core_pick can be NULL if no selection was made for a CPU because
     * it was either offline or went offline during a sibling's core-wide
     * selection. In this case, do a core-wide selection.
     */
    if (rq->core->core_pick_seq == rq->core->core_task_seq &&
        rq->core->core_pick_seq != rq->core_sched_seq &&
        rq->core_pick) {
        WRITE_ONCE(rq->core_sched_seq, rq->core->core_pick_seq);

        next = rq->core_pick;
        if (next != prev) {
            put_prev_task(rq, prev);
            set_next_task(rq, next);
        }

        rq->core_pick = NULL;
        goto out;
    }
```

这部分代码关心到新进程的调度，这里边的逻辑涉及到`core_task_seq`、`core_pick_seq`、`core_sched_seq`三个字段的值的判断，`core_task_seq`在每次将任务放入到rq、从rq中出队时都会递增，这意味着接下来要执行的进程会发生变化；`core_pick_seq`标记某次的rq入队或出队操作是否有新的任务选择与之对应，即rq入队或出队操作之后执行了新运行任务的选择，则将`core_pick_seq`的值更新为`core_task_seq`的值，此时新选择的任务放在`rq->core_pick`字段。当新运行的任务即将抢占其他占用cpu的任务时将`core_sched_seq`更新为`core_pick_seq`的值。因为在rq上进行入队、出队操作之后可能并不会立马进行新执行任务的选择，进行了新的任务选择之后并不会立马让新选择的任务抢占其他的任务，所以需要使用这三个字段来同步rq变动、新的任务选择、新的任务调度这三个过程。若`rq->core->core_pick_seq == rq->core->core_task_seq`并且`rq->core->core_pick_seq != rq->core_sched_seq`相同时表示rq变动之后已经选择了新的任务，但是新的任务还没有执行。此时若`rq->core_pick`不为空并且不是正在运行的任务，调用调度类的`put_prev_task`（当任务被抢占时执行此方法）、`set_next_task`方法（当任务抢占其他任务的时候执行此方法），然后跳转到`out`这个标签处继续执行。

```c
    put_prev_task_balance(rq, prev, rf);

    smt_mask = cpu_smt_mask(cpu);
    need_sync = !!rq->core->core_cookie;
```

这三行代码与后边的流程紧密相关，逐一记录每行代码的含义。`put_prev_task_balance`函数按照优先级从高到低的顺序调用调度类的`balance`方法，然后对即将被抢占的任务调用调度类的`put_prev_task`函数，后边会详细记录`put_prev_task_balance`函数流程。`smt`是`Simultaneous Multi-Threading`的缩写，intel的许多处理器都支持超线程技术，开启超线程技术的情况下一颗物理内核对应两颗虚拟内核，这种情况就叫做`Simultaneous Multi-Threading`技术。`cpu_smt_mask`返回的是所有的逻辑cpu，这些cpu实际上是同一颗物理cpu通过`SMT`技术（例如intel的超线程技术）虚拟出来的。`need_sync`表明当前的rq中是否设置了`core_cookie`的值。

```c
    /* reset state */
    rq->core->core_cookie = 0UL;
    if (rq->core->core_forceidle_count) {
        if (!core_clock_updated) {
            update_rq_clock(rq->core);
            core_clock_updated = true;
        }
        sched_core_account_forceidle(rq);
        /* reset after accounting force idle */
        rq->core->core_forceidle_start = 0;
        rq->core->core_forceidle_count = 0;
        rq->core->core_forceidle_occupation = 0;
        need_sync = true;
        fi_before = true;
    }
```

在这个函数的后续流程会提到，某个逻辑cpu中无法找到与指定`core_cookie`值相匹配的任务时，只能选择一个idle task来运行，此时`core_forceidle_count`这个字段将不为0，这个字段对于所有属于同一颗物理cpu的逻辑cpu都可见。当这个字段的值不为0时调用`sched_core_account_forceidle`函数更新关于force idle的相关统计信息、重置这颗物理cpu中force idle的相关字段，将`need_sync`、`fi_before`设置为True。`need_sync`为`True`表示物理cpu的core_cookie的值需要重新设置、`fi_before`为`True`意味着之前某个逻辑cpu中选择了idle task作为即将执行的任务，这两个字段具体带来的影响在后续的流程中详细记录。<u>这里我不理解的地方是为什么不需要考虑缓存一致性，即这块代码可能在一个物理cpu中对应的多个逻辑cpu中执行，某个逻辑cpu重置了`core_forceidle_count`的值之后另外一个逻辑cpu可能无法看到，导致统计字段重复更新。</u>

```c
    /*
     * Optimize for common case where this CPU has no cookies
     * and there are no cookied tasks running on siblings.
     */
    if (!need_sync) {
        next = pick_task(rq);
        if (!next->core_cookie) {
            rq->core_pick = NULL;
            /*
             * For robustness, update the min_vruntime_fi for
             * unconstrained picks as well.
             */
            WARN_ON_ONCE(fi_before);
            task_vruntime_update(rq, next, false);
            goto out_set_next;
        }
    }

    cookie = rq->core->core_cookie = max->core_cookie;
```

当还没有一个`core_cookie`设置到某个逻辑cpu上时，此时cpu中运行的可能是idle task，调用`pick_task`函数按照调度类的优先级调用每个调度类的`pick_task`方法，直到某个调度类的`pick_next`方法返回一个任务，这个任务就是接下来在这个逻辑cpu上执行的任务。若选出来的task中没有设置`core_cookie`，调用CFS的`task_vruntime_update`函数更新cfs_rq的`min_vruntime_fi`字段，这里忽略这个字段的含义。接下来代码跳转到`out_set_next`标记处继续执行。

```c
    /*
     * For each thread: do the regular task pick and find the max prio task
     * amongst them.
     *
     * Tie-break prio towards the current CPU
     */
    for_each_cpu_wrap(i, smt_mask, cpu) {
        rq_i = cpu_rq(i);

        /*
         * Current cpu always has its clock updated on entrance to
         * pick_next_task(). If the current cpu is not the core,
         * the core may also have been updated above.
         */
        if (i != cpu && (rq_i != rq->core || !core_clock_updated))
            update_rq_clock(rq_i);

        p = rq_i->core_pick = pick_task(rq_i);
        if (!max || prio_less(max, p, fi_before))
            max = p;
    }

    cookie = rq->core->core_cookie = max->core_cookie;
```

当某个逻辑cpu的`core_cookie`在之前已经设置，会执行这部段代码。这段代码是从某个物理cpu的某个逻辑cpu开始编译这个物理cpu上的所有逻辑cpu，从中选出来优先级最高的task，将这个task的`core_cookie`设置为这个逻辑cpu的`core_cookie`。在遍历过程中还会选出来优先级最高的任务，将物理cpu的`core_cookie`设置为优先级最高任务的`core_cookie`。<u>遍历过程中还会考虑更新逻辑cpu的rq时间，可以参照注释内容简单理解，细节以后补充。</u>

```c
    /*
     * For each thread: try and find a runnable task that matches @max or
     * force idle.
     */
    for_each_cpu(i, smt_mask) {
        rq_i = cpu_rq(i);
        p = rq_i->core_pick;

        if (!cookie_equals(p, cookie)) {
            p = NULL;
            if (cookie)
                p = sched_core_find(rq_i, cookie);
            if (!p)
                p = idle_sched_class.pick_task(rq_i);
        }

        rq_i->core_pick = p;

        if (p == rq_i->idle) {
            if (rq_i->nr_running) {
                rq->core->core_forceidle_count++;
                if (!fi_before)
                    rq->core->core_forceidle_seq++;
            }
        } else {
            occ++;
        }
    }
```

现在已经确定了物理cpu的`core_cookie`的值，那个物理cpu对应的逻辑cpu中运行任务的`core_cookie`的值要与物理cpu的`core_cookie`的值相同。这个`for`循环迭代物理cpu中的每个逻辑cpu，判断逻辑cpu中即将运行任务的`core_cookie`的值是否与物理cpu的`core_cookie`的值相同，若不同则调用`sched_core_find`找到一个新的任务，若找不到这样的任务则使用idle task。当为正在遍历的cpu寻找到的新任务是idle task，则需要更新`core_forceidle_count`以及`core_forceidle_seq`两个字段的值：当逻辑cpu对应的rq中有可运行任务的计数不为0时，即此时有可以运行的任务、但是不得不运行idle task，因此`core_forceidle_count`递增；当上次执行`pick_next_task`时未出现强制让某个逻辑cpu运行idle task的情况，将`core_forceidle_seq`递增用以记录逻辑cpu不得不执行idle task出现的次数。

```c
    if (schedstat_enabled() && rq->core->core_forceidle_count) {
        rq->core->core_forceidle_start = rq_clock(rq->core);
        rq->core->core_forceidle_occupation = occ;
    }

    rq->core->core_pick_seq = rq->core->core_task_seq;
    next = rq->core_pick;
    rq->core_sched_seq = rq->core->core_pick_seq;

    /* Something should have been selected for current CPU */
    WARN_ON_ONCE(!next);
```

接下来更新统计字段，包含最近一次强制让某个逻辑cpu运行idle task的时间以及出现次数。最重要的是更新`core_pick_seq`以及`core_sched_seq`的值为`core_task_seq`，表明新的任务选择过程中已经考虑到了之前rq的变动，即考虑到了在rq中的出队、入队操作。

```c
    /*
     * Reschedule siblings
     *
     * NOTE: L1TF -- at this point we're no longer running the old task and
     * sending an IPI (below) ensures the sibling will no longer be running
     * their task. This ensures there is no inter-sibling overlap between
     * non-matching user state.
     */
    for_each_cpu(i, smt_mask) {
        rq_i = cpu_rq(i);

        /*
         * An online sibling might have gone offline before a task
         * could be picked for it, or it might be offline but later
         * happen to come online, but its too late and nothing was
         * picked for it.  That's Ok - it will pick tasks for itself,
         * so ignore it.
         */
        if (!rq_i->core_pick)
            continue;

        /*
         * Update for new !FI->FI transitions, or if continuing to be in !FI:
         * fi_before     fi      update?
         *  0            0       1
         *  0            1       1
         *  1            0       1
         *  1            1       0
         */
        if (!(fi_before && rq->core->core_forceidle_count))
            task_vruntime_update(rq_i, rq_i->core_pick, !!rq->core->core_forceidle_count);

        rq_i->core_pick->core_occupation = occ;

        if (i == cpu) {
            rq_i->core_pick = NULL;
            continue;
        }

        /* Did we break L1TF mitigation requirements? */
        WARN_ON_ONCE(!cookie_match(next, rq_i->core_pick));

        if (rq_i->curr == rq_i->core_pick) {
            rq_i->core_pick = NULL;
            continue;
        }

        resched_curr(rq_i);
    }
```

虽然在上边的逻辑中已经为从属于某个物理cpu的所有逻辑cpu选择了新的要执行的任务，但是这些逻辑cpu目前可能还无法感知这个变化，这个`for`循环遍历所有逻辑cpu，通过调用`resched_curr`函数并传入逻辑cpu的rq，通知这个逻辑cpu需要让新选择的任务抢占当前执行的任务以保持逻辑cpu中执行任务的`core_cookie`完全一致。在通知之前需要考虑这几种情况：1).未能给某个逻辑cpu选择下一个执行的任务，这个cpu可能处于不可用状态，此时填过这个cpu，2).若上次运行`pick_next_task`时没有强制某个逻辑cpu运行idle task但是本次执行却有这个要求，则调用CFS的`task_vruntime_update`函数更新cfs_rq的`min_vruntime_fi`字段（这里忽略这个字段的含义），3).若正在遍历的逻辑cpu为执行`pick_next_task`函数的逻辑cpu，这个cpu已经知道要使用新的任务替换正在执行的任务，不需要额外通知，4).若某个逻辑cpu中正在运行的任务和新选择的任务相同同样不需要额外通知。

```c
out_set_next:
    set_next_task(rq, next);
out:
    if (rq->core->core_forceidle_count && next == rq->idle)
        queue_core_balance(rq);

    return next;
```

`out_set_next`标签中执行`set_next_task`函数，这个函数调用即将运行任务关联的调度类的`set_next_task`方法。`out`标签中在调用`pick_next_task`函数的cpu强制运行idle task的时候调用`queue_core_balance`函数注册一个回调函数，这个回调函数是`sched_core_balance`函数，当进行任务负载均衡的时候会调用此函数，这里忽略`sched_core_balance`函数的内容。

#### `__pick_next_task`函数

```c
/*
 * Pick up the highest-prio task:
 */
static inline struct task_struct *
__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    const struct sched_class *class;
    struct task_struct *p;

    /*
     * Optimization: we know that if all tasks are in the fair class we can
     * call that function directly, but only if the @prev task wasn't of a
     * higher scheduling class, because otherwise those lose the
     * opportunity to pull in more work from other CPUs.
     */
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
           rq->nr_running == rq->cfs.h_nr_running)) {

        p = pick_next_task_fair(rq, prev, rf);
        if (unlikely(p == RETRY_TASK))
            goto restart;

        /* Assume the next prioritized class is idle_sched_class */
        if (!p) {
            put_prev_task(rq, prev);
            p = pick_next_task_idle(rq);
        }

        return p;
    }

restart:
    put_prev_task_balance(rq, prev, rf);

    for_each_class(class) {
        p = class->pick_next_task(rq);
        if (p)
            return p;
    }

    BUG(); /* The idle class should always have a runnable task. */
}
```

如果rq中运行的所有任务都是由CFS来接管、CFS比被抢占任务关联的调度类优先级更高，那么调用`pick_next_task_fair`函数选择接下来运行的任务，如果无法找到接下来运行的任务则选择一个idle task任务运行并且对即将被抢占的任务调用`put_prev_task`函数，在`put_prev_task`函数中会调用即将被抢占的任务关联的调度类的`put_prev_task`方法。当`pick_next_task_fair`返回`RETRY_TASK`时，调用`put_prev_task_balance`函数进行任务负载均衡以明确接下来需要运行的任务、遍历每个调度类选出接下来运行的任务。

#### `put_prev_task_balance`函数

```c
static void put_prev_task_balance(struct rq *rq, struct task_struct *prev,
                  struct rq_flags *rf)
{
#ifdef CONFIG_SMP
    const struct sched_class *class;
    /*
     * We must do the balancing pass before put_prev_task(), such
     * that when we release the rq->lock the task is in the same
     * state as before we took rq->lock.
     *
     * We can terminate the balance pass as soon as we know there is
     * a runnable task of @class priority or higher.
     */
    for_class_range(class, prev->sched_class, &idle_sched_class) {
        if (class->balance(rq, prev, rf))
            break;
    }
#endif

    put_prev_task(rq, prev);
}
```

从被抢占任务的调度类开始向优先级低的调度类遍历，遍历过程中调度每个调度类的`balance`方法进行任务负载均衡寻找接下来运行的任务（忽略任务负载均衡相关的内容），然后对即将抢占的任务执行`put_prev_task`函数，这个函数会调用任务关联调度类的`put_prev_task`方法执行任务被抢占前的调度类内部更新操作。

#### `migrate_disable_switch`函数

```c
static void migrate_disable_switch(struct rq *rq, struct task_struct *p)
{
    if (likely(!p->migration_disabled))
        return;

    if (p->cpus_ptr != &p->cpus_mask)
        return;

    /*
     * Violates locking rules! see comment in __do_set_cpus_allowed().
     */
    __do_set_cpus_allowed(p, cpumask_of(rq->cpu), SCA_MIGRATE_DISABLE);
}
```

这个函数调用`__do_set_cpus_allowed`保证任务`p`无法调度到rq所属cpu之中，第二个参数指定任务在cpu转移过程中排除的cpu，第三个参数为禁用任务转移的标记。在此之前还要进行一些基本情况判断，当没有禁用任务`p`在cpu之间移动时、当任务`p`和其他的任务共享同一组cpu时（`p->cpu_ptr != &p->cpus_mask`）都不进行任何操作。

#### `context_switch`函数

```c
/*
 * context_switch - switch to the new MM and the new thread's register state.
 */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
           struct task_struct *next, struct rq_flags *rf)
{
    prepare_task_switch(rq, prev, next);

    /*
     * For paravirt, this is coupled with an exit in switch_to to
     * combine the page table reload and the switch backend into
     * one hypercall.
     */
    arch_start_context_switch(prev);

    /*
     * kernel -> kernel   lazy + transfer active
     *   user -> kernel   lazy + mmgrab() active
     *
     * kernel ->   user   switch + mmdrop() active
     *   user ->   user   switch
     */
    if (!next->mm) {                                // to kernel
        enter_lazy_tlb(prev->active_mm, next);

        next->active_mm = prev->active_mm;
        if (prev->mm)                           // from user
            mmgrab(prev->active_mm);
        else
            prev->active_mm = NULL;
    } else {                                        // to user
        membarrier_switch_mm(rq, prev->active_mm, next->mm);
        /*
         * sys_membarrier() requires an smp_mb() between setting
         * rq->curr / membarrier_switch_mm() and returning to userspace.
         *
         * The below provides this either through switch_mm(), or in
         * case 'prev->active_mm == next->mm' through
         * finish_task_switch()'s mmdrop().
         */
        switch_mm_irqs_off(prev->active_mm, next->mm, next);
        lru_gen_use_mm(next->mm);

        if (!prev->mm) {                        // from kernel
            /* will mmdrop() in finish_task_switch(). */
            rq->prev_mm = prev->active_mm;
            prev->active_mm = NULL;
        }
    }

    rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);

    prepare_lock_switch(rq, next, rf);

    /* Here we just switch the register state and the stack. */
    switch_to(prev, next, prev);
    barrier();

    return finish_task_switch(prev);
}
```

`prepare_task_switch`函数与`finish_task_switch`函数对应，`prepare_task_switch`执行进程抢占前的准备工作，后边详细记录`prepare_task_switch`函数的具体流程。接下来执行的函数`arch_start_context_switch`是处理器要求在进程抢占之前调用的函数，这个函数有的处理器没有实现（例如`x86_64`）、有的处理器有自己的实现。

当新运行的任务为内核线程（`next->mm`为空），执行`enter_lazy_tlb`函数跳过重新向`TLB`缓存之中写入新任务的页表项，之所以能够跳过这一步是因为在linux中运行的所有的任务（包含内核线程、用户态进程、进程中的线程）使用的内核空间都是相同的并且内核线程不需要访问用户态内存空间，所以复用被抢占任务的`TLB`缓存是没问题的。这里详细介绍`mm`与`active_mm`之间的关系，`mm`为一个进程的虚拟地址空间，对于进程而言`active_mm`以及`mm`字段的值都是相同的，对于内核线程而言`active_mm`直接引用为上一个运行的进程的虚拟地址空间。当被抢占的任务为进程，即将运行的内核线程直接引用进程的虚拟地址空间，递增`mm`的引用计数；当被抢占的任务、新运行的任务都是内核线程，使用`next->active_mm = prev->active_mm`来引用之前运行进程的虚拟地址空间。

当新运行的任务为进程时，将rq的`membarrier_state`设置为进程虚拟地址空间的`membarrier_state`状态，`membarrier_state`为`membarrier`系统调用中的内容，具体含义见这个系统调用的说明文档。`switch_mm_irqs_off`函数负责处理在任务切换时`TLB`以及内存管理相关结构的切换。`lru_gen_use_mm`函数标记新任务的虚拟地址空间中的内存最近被使用过，尽量其中的内存页换出操作。当被抢占的任务为内核线程时，在rq中保存内核线程引用的虚拟地址空间，在`finish_task_switch`将会解除对这个用户态空间的引用。

上边的代码中提到了`switch_mm_irqs_off`函数，这个函数用于切换内存管理相关的内容，这里简要介绍这个函数的内容。若任务切换不会导致`TLB`缓存的内存页对应地址空间不变，则进一步确定`TLB`缓存中内容是否需要更新，在`TLB`缓存的内存页对应地址空间中添加当前运行cpu的使用标记；若发生变化在之前的地址空间中清除当前cpu的使用标记、在新的地址空间中添加当前cpu的使用标记，选择新的`ASID`（用于标识新的地址空间），在选择新的`ASID`的时候可能会要求更新`TLB`缓存中的内容。无论哪种情况，一旦要求更新`TLB`缓存中的内容，调用`load_new_mm_cr3`触发`TLB`缓存更新。在后一种情况中还会调用`cond_mitigation`函数，这个函数用于处理`Spectre`漏洞。

#### `prepare_task_switch`函数

```c
static inline void
prepare_task_switch(struct rq *rq, struct task_struct *prev,
            struct task_struct *next)
{
    kcov_prepare_switch(prev);
    sched_info_switch(rq, prev, next);
    perf_event_task_sched_out(prev, next);
    rseq_preempt(prev);
    fire_sched_out_preempt_notifiers(prev, next);
    kmap_local_sched_out();
    prepare_task(next);
    prepare_arch_switch(next);
}
```

忽略`kcov_prepare_switch`、`sched_info_switch`、`perf_event_task_sched_out`、`kmap_local_sched_out`函数，`rseq_preempt`函数为任务添加`RSEQ`机制专用的任务被抢占的标识、设置进程恢复之后进行提醒，提醒的作用是进程已经恢复让`RSEQ`中的代码片段重新执行。`fire_sched_out_preempt_notifiers`用于提醒抢占事件的监听者，具体方法是调用点监听者指定的`sched_out`方法。`prepare_task`函数将即将运行进程的`on_cpu`字段设置为1，使用了`WRITE_ONCE`保证了缓存一致性。`prepare_arch_switch`为处理器特定的函数，在某些处理器中要求在进行进程抢占时调用此函数，在其他的处理器中没有实现（例如`x86_64`）。

#### `switch_to`宏

```c
#define switch_to(prev, next, last)                    \
do {                                    \
    ((last) = __switch_to_asm((prev), (next)));            \
} while (0)
```

这个宏调用了`__switch_to_asm`函数，这个函数返回被抢占的任务，后边详细记录`__switch_to_asm`函数的具体流程。

#### `finish_task_switch`函数

```c
static struct rq *finish_task_switch(struct task_struct *prev)
    __releases(rq->lock)
{
    struct rq *rq = this_rq();
    struct mm_struct *mm = rq->prev_mm;
    unsigned int prev_state;

    /*
     * The previous task will have left us with a preempt_count of 2
     * because it left us after:
     *
     *    schedule()
     *      preempt_disable();            // 1
     *      __schedule()
     *        raw_spin_lock_irq(&rq->lock)    // 2
     *
     * Also, see FORK_PREEMPT_COUNT.
     */
    if (WARN_ONCE(preempt_count() != 2*PREEMPT_DISABLE_OFFSET,
              "corrupted preempt_count: %s/%d/0x%x\n",
              current->comm, current->pid, preempt_count()))
        preempt_count_set(FORK_PREEMPT_COUNT);

    rq->prev_mm = NULL;

    /*
     * A task struct has one reference for the use as "current".
     * If a task dies, then it sets TASK_DEAD in tsk->state and calls
     * schedule one last time. The schedule call will never return, and
     * the scheduled task must drop that reference.
     *
     * We must observe prev->state before clearing prev->on_cpu (in
     * finish_task), otherwise a concurrent wakeup can get prev
     * running on another CPU and we could rave with its RUNNING -> DEAD
     * transition, resulting in a double drop.
     */
    prev_state = READ_ONCE(prev->__state);
    vtime_task_switch(prev);
    perf_event_task_sched_in(prev, current);
    finish_task(prev);
    tick_nohz_task_switch();
    finish_lock_switch(rq);
    finish_arch_post_lock_switch();
    kcov_finish_switch(current);
    /*
     * kmap_local_sched_out() is invoked with rq::lock held and
     * interrupts disabled. There is no requirement for that, but the
     * sched out code does not have an interrupt enabled section.
     * Restoring the maps on sched in does not require interrupts being
     * disabled either.
     */
    kmap_local_sched_in();

    fire_sched_in_preempt_notifiers(current);
    /*
     * When switching through a kernel thread, the loop in
     * membarrier_{private,global}_expedited() may have observed that
     * kernel thread and not issued an IPI. It is therefore possible to
     * schedule between user->kernel->user threads without passing though
     * switch_mm(). Membarrier requires a barrier after storing to
     * rq->curr, before returning to userspace, so provide them here:
     *
     * - a full memory barrier for {PRIVATE,GLOBAL}_EXPEDITED, implicitly
     *   provided by mmdrop(),
     * - a sync_core for SYNC_CORE.
     */
    if (mm) {
        membarrier_mm_sync_core_before_usermode(mm);
        mmdrop_sched(mm);
    }
    if (unlikely(prev_state == TASK_DEAD)) {
        if (prev->sched_class->task_dead)
            prev->sched_class->task_dead(prev);

        /* Task is done with its stack. */
        put_task_stack(prev);

        put_task_struct_rcu_user(prev);
    }

    return rq;
}
```

这个函数首先判断抢占计数是否为2，若不为2强制设置为2，这是因为在执行`schedule`函数开始到现在会调用两次`preempt_disable`函数，每次调用这个函数会递增抢占计数。`vtime_task_switch`函数涉及到任务和cpu的时间统计、`perf_event_task_sched_in`涉及性记录任务调入到cpu事件的性能记录、`tick_nohz_task_switch`函数涉及到一种特殊的定时器，忽略这几个函数中的内容。`finish_task`函数将被抢占任务的`on_cpu`字段设置为0并且保证`RELEASE`语义的内存同步，`finish_lock_switch`函数触发rq中任务负载均衡相关的回调函数、释放rq锁并恢复中断，`finish_arch_post_lock_switch`在`x86_64`架构中是一个空函数。

忽略`kcov_finish_switch`、`kmap_local_sched_in`函数内容，提醒即将运行任务的事件监听者此任务被调度cpu执行（在`fire_sched_in_preempt_notifiers`中调用监听者注册的`sched_in`方法）。若被抢占任务为一个进程，考虑是否在返回用户态之前执行`SERIALIZE`指令，若新旧任务的地址空间不同或者新任务地址空间的内存屏障状态中包含`PRIVATE_EXPEDITED_SYNC_CORE`标记则跳过执行（忽略这个标记的含义），其他的情况需要执行`SERIALIZE`指令（在intel的指令集手册中，`SERIALIZE`指令的含义为`Before the next instruction is fetched and executed, the SERIALIZE instruction ensures that all modifications to flags, registers, and memory by previous instructions are completed, draining all buffered writes to memory`，即在这个指令之前素有指令都会得到执行，这是为了保证在返回用户态之前所有内核态的操作全部执行完成），随后取消对被抢占任务的地址空间的引用。

当被抢占的任务状态为`TASK_DEAD`，调用调度类的`task_dead`方法、递减此任务的栈空间、递减`RCU`机制中栈空间的引用计数，若栈空间的引用计数在递减之后为0则释放它。

#### `__switch_to_asm`函数

```c
/*
 * %rdi: prev task
 * %rsi: next task
 */
.pushsection .text, "ax"
SYM_FUNC_START(__switch_to_asm)
    /*
     * Save callee-saved registers
     * This must match the order in inactive_task_frame
     */
    pushq    %rbp
    pushq    %rbx
    pushq    %r12
    pushq    %r13
    pushq    %r14
    pushq    %r15

    /* switch stack */
    movq    %rsp, TASK_threadsp(%rdi)
    movq    TASK_threadsp(%rsi), %rsp

#ifdef CONFIG_STACKPROTECTOR
    movq    TASK_stack_canary(%rsi), %rbx
    movq    %rbx, PER_CPU_VAR(fixed_percpu_data) + stack_canary_offset
#endif

    /*
     * When switching from a shallower to a deeper call stack
     * the RSB may either underflow or use entries populated
     * with userspace addresses. On CPUs where those concerns
     * exist, overwrite the RSB with entries which capture
     * speculative execution to prevent attack.
     */
    FILL_RETURN_BUFFER %r12, RSB_CLEAR_LOOPS, X86_FEATURE_RSB_CTXSW

    /* restore callee-saved registers */
    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbx
    popq    %rbp

    jmp    __switch_to
SYM_FUNC_END(__switch_to_asm)
.popsection
```

这个函数执行任务抢占过程中非常关键的一个函数，这个函数中有一些微妙的细节，结合一个例子来理解这些微妙的细节。假设`switch_to`函数的`prev`为任务a、`next`为任务b、任务a此时运行在CPU 3之中、在任务c运行时抢占了任务b的CPU资源，那么在调用`__switch_to_asm`函数时是在CPU 3上执行任务a的过程中，`pushq`使用的栈空间为任务a的栈空间。

先关注任务c抢占任务b时的一些操作，这些操作有助于理解任务b抢占任务a过程中的一个细节，即执行`popq`指令的时候使用的栈究竟是谁的栈。此时这个函数执行的时候`prev`为任务b、`next`为任务c、在任务b所在的CPU中运行此函数。此函数首先将`rbp`、`rbx`、`r12-r15`寄存器内容放入到任务b的栈之中，然后使用`movq %rsp, TASK_THREADSP(%rdi)`将`rsp`寄存器的值放入到任务b的`task_struct->thread_struct->sp`字段之中、使用`movq TASK_threadsp(%rsi), %rsp`将任务c的栈地址写入到`rsp`寄存器中，后续代码开始使用c的栈空间而非b的栈空间。接下来的`popq`指令使用的栈是任务c的栈空间而非任务b的栈空间，恢复寄存器的值（这些值是在任务c被其他任务抢占的时候执行这个函数中前几个`pushq`指令的时候入栈的）。从这里看到这个函数中`pushq`使用的栈与`popq`使用的栈不同同一个栈，`pushq`使用的栈是被抢占任务的栈，而`popq`使用的栈是即将恢复执行任务的栈。接下来`__switch_to`函数返回之后任务b失去CPU资源，导致任务b处于暂停状态，任务c开始执行。

接下来关注任务b抢占任务a的情况，此时`__switch_to_asm`函数的`prev`为任务a、`next`为任务b，此时这个函数使用任务a的栈空间。

函数开始的四个`pushq`指令将6个寄存器的值保存到任务a的栈之中，然后`rsp`寄存器的值至任务a的`task_struct->thread_struct->sp`之中、将`rsp`寄存器的值设置为任务b的栈空间，后续的代码开始使用任务b的栈空间。

`CONFIG_STACKPROTECTOR`启用的代码与一种栈溢出检测技术相关，即在栈空间的结束后的位置放置一个叫做stack canary的值，通过检查栈空间结束后第一个位置的值是否与这个值相同来判断是否存在栈溢出现象，这个宏启用的代码使用任务b的stack canary替换任务a的stack canary。

`FILL_RETURN_BUFFER`指令涉及到现代处理器的RSB（Return stack buffer）特性，RSB中存放了进程执行函数调用过程中每次调用返回之后可能执行的下一条指令的地址，这个指令用于在任务切换时清空被抢占任务的地址信息（使用特定的值替代被抢占进程的地址信息），以防止RSB中存储其他进程地址而带来的潜在安全性问题。

接下来执行的6条`popq`指令从任务b的栈中弹出6个寄存器的值，这些值并不是在任务b抢占任务a时入栈的值，而是在任务c抢占任务b的时候入栈的值。在任务c抢占任务b时将这几个寄存器的值压入栈中，随后任务b失去CPU资源暂停执行，随着时间的流逝任务b得到在CPU 3资源可以继续运行，此时CPU 3使用的栈空间为任务b的栈空间，所以此时是从b的栈空间中出栈任务b被任务c抢占时压入的寄存器的值。

最后执行`__switch_to`函数，当函数返回之后任务a已经失去CPU资源暂停运行，任务b开始恢复运行。

#### `__switch_to`函数

```c
__no_kmsan_checks
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
    struct thread_struct *prev = &prev_p->thread;
    struct thread_struct *next = &next_p->thread;
    struct fpu *prev_fpu = &prev->fpu;
    int cpu = smp_processor_id();

    WARN_ON_ONCE(IS_ENABLED(CONFIG_DEBUG_ENTRY) &&
             this_cpu_read(hardirq_stack_inuse));

    if (!test_thread_flag(TIF_NEED_FPU_LOAD))
        switch_fpu_prepare(prev_fpu, cpu);

    /* We must save %fs and %gs before load_TLS() because
     * %fs and %gs may be cleared by load_TLS().
     *
     * (e.g. xen_load_tls())
     */
    save_fsgs(prev_p);

    /*
     * Load TLS before restoring any segments so that segment loads
     * reference the correct GDT entries.
     */
    load_TLS(next, cpu);

    /*
     * Leave lazy mode, flushing any hypercalls made here.  This
     * must be done after loading TLS entries in the GDT but before
     * loading segments that might reference them.
     */
    arch_end_context_switch(next_p);

    /* Switch DS and ES.
     *
     * Reading them only returns the selectors, but writing them (if
     * nonzero) loads the full descriptor from the GDT or LDT.  The
     * LDT for next is loaded in switch_mm, and the GDT is loaded
     * above.
     *
     * We therefore need to write new values to the segment
     * registers on every context switch unless both the new and old
     * values are zero.
     *
     * Note that we don't need to do anything for CS and SS, as
     * those are saved and restored as part of pt_regs.
     */
    savesegment(es, prev->es);
    if (unlikely(next->es | prev->es))
        loadsegment(es, next->es);

    savesegment(ds, prev->ds);
    if (unlikely(next->ds | prev->ds))
        loadsegment(ds, next->ds);

    x86_fsgsbase_load(prev, next);

    x86_pkru_load(prev, next);

    /*
     * Switch the PDA and FPU contexts.
     */
    this_cpu_write(current_task, next_p);
    this_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p));

    switch_fpu_finish();

    /* Reload sp0. */
    update_task_stack(next_p);

    switch_to_extra(prev_p, next_p);

    if (static_cpu_has_bug(X86_BUG_SYSRET_SS_ATTRS)) {
        /*
         * AMD CPUs have a misfeature: SYSRET sets the SS selector but
         * does not update the cached descriptor.  As a result, if we
         * do SYSRET while SS is NULL, we'll end up in user mode with
         * SS apparently equal to __USER_DS but actually unusable.
         *
         * The straightforward workaround would be to fix it up just
         * before SYSRET, but that would slow down the system call
         * fast paths.  Instead, we ensure that SS is never NULL in
         * system call context.  We do this by replacing NULL SS
         * selectors at every context switch.  SYSCALL sets up a valid
         * SS, so the only way to get NULL is to re-enter the kernel
         * from CPL 3 through an interrupt.  Since that can't happen
         * in the same task as a running syscall, we are guaranteed to
         * context switch between every interrupt vector entry and a
         * subsequent SYSRET.
         *
         * We read SS first because SS reads are much faster than
         * writes.  Out of caution, we force SS to __KERNEL_DS even if
         * it previously had a different non-NULL value.
         */
        unsigned short ss_sel;
        savesegment(ss, ss_sel);
        if (ss_sel != __KERNEL_DS)
            loadsegment(ss, __KERNEL_DS);
    }

    /* Load the Intel cache allocation PQR MSR. */
    resctrl_sched_in(next_p);

    return prev_p;
}
```

`prev_p`为被抢占的任务、`next_p`为即将运行的任务，这个函数首先调用`switch_fpu_prepare`函数在`prev_p`中保存其运行时浮点数相关寄存器的值，记录`prev_p`最后运行所在的cpu。`load_TLS`函数从`next_p`之中加载线程本地数据，这个函数执行的时候会清空`FS`、`GS`寄存器的值，所以在执行此函数之前要先使用`save_fsgs`函数保存`prev_p`运行时两个寄存器的值。有的处理器定义了在完成任务切换时执行的操作，`arch_end_context_switch`函数执行这些操作。接下来是`ES`和`DS`寄存器的操作，这两个寄存器为数据段、扩展段寄存器，保存`prev_p`运行时两个寄存器的值，将`next_p`中保存的两个寄存器的值恢复到对应的寄存器之中。接下来从`next_p`之中恢复`FS`、`GS`寄存器的值，对应的函数是`x86_fsgsbase_load`，`prev_p`运行时这两个寄存器的值已经保存，所以此处可以直接覆盖。

忽略`x86_pkru_load`函数、AMD处理器问题修复，`switch_to_extra`更新进程调试寄存器、io权限位图、推测执行相关的内容，最后调用的`resctrl_sched_in`函数与intel cpu的资源控制功能有关。这些内容涉及到cpu体系结构相关的内容，这里就不展开记录了。
