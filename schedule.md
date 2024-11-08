## 简介

在linux任务调度过程中，当为某个CPU中rq上正在运行的任务设置了`TIF_NEED_RESCHED`标志，意味着这个任务即将被其他任务抢占，即另外一个任务替代这个任务占用CPU资源，完成这个动作的函数是`schedule`函数。在`try_to_wake_up`中会将rq中正在执行的任务加上`TIF_NEED_RESCHED`标志，另外一个流程是定时器由驱动的rq的时间更新流程，在这个流程中可能会为rq中正在运行的任务添加`TIF_NEED_RESCHED`标记。在记录`schedule`函数之前，先记录高精度定时器如何影响任务调度过程的。

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
