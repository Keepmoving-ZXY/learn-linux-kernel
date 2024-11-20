本文件中的内容是CFS调度类中`task_fork_pair`函数流程记录，这个函数在使用`fork`创建子进程过程中调用，调用的时候传入的参数是子进程的`struct task_struct`实例。

#### `task_fork_fair`函数

```c
/*
 * called on fork with the child task as argument from the parent's context
 *  - child not yet on the tasklist
 *  - preemption disabled
 */
static void task_fork_fair(struct task_struct *p)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &p->se, *curr;
    struct rq *rq = this_rq();
    struct rq_flags rf;

    rq_lock(rq, &rf);
    update_rq_clock(rq);

    cfs_rq = task_cfs_rq(current);
    curr = cfs_rq->curr;
    if (curr) {
        update_curr(cfs_rq);
        se->vruntime = curr->vruntime;
    }
    place_entity(cfs_rq, se, 1);

    if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
        /*
         * Upon rescheduling, sched_class::put_prev_task() will place
         * 'current' within the tree based on its new key value.
         */
        swap(curr->vruntime, se->vruntime);
        resched_curr(rq);
    }

    se->vruntime -= cfs_rq->min_vruntime;
    rq_unlock(rq, &rf);
}
```

这个函数的流程比较简单：持有rq锁，`update_rq_lock`函数更新rq的时间以及相关的统计字段，在`task_wake_up.md`之中详细记录了这个函数的流程；使用`update_curr`函数更新当前正在运行任务的运行时间统计，在后边详细记录这个函数的流程，同时将任务（子进程）的虚拟运行时间设置成和父进程一致；使用`place_entity`为新创建的任务（新创建的子进程）计算并设置合适的虚拟运行时间，在后边详细记录这个函数的流程；若设置了子进程优先执行，先交换父进程、子进程的虚拟运行时间，使用`resched_curr`函数为父进程和cpu添加进程抢占相关标志，在`task_wake_up.md`之中详细记录了这个函数的流程；为新任务（子进程）计算相对的虚拟运行时间，最后释放rq锁。

注意到在这个函数最后计算了新任务的相对虚拟运行时间，这是因为此时新任务还未执行，新任务执行时的`min_vruntime`很可能会递增，此时使用`min_vruntime`加上新任务的相对虚拟运行时间得到新任务的虚拟运行时间。在CFS之中`min_vruntime`为虚拟运行时间的基准值，是rq上所有可运行任务最小的vruntime，可以理解为rq上当前的虚拟时间，所有任务的虚拟运行时间都是由基准值加上相对时间得来，这样便于进程之间比较过程中尽量保证公平。

#### `update_curr`函数

```c
/*
 * Update the current task's runtime statistics.
 */
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    if (unlikely(!curr))
        return;

    delta_exec = now - curr->exec_start;
    if (unlikely((s64)delta_exec <= 0))
        return;

    curr->exec_start = now;

    if (schedstat_enabled()) {
        struct sched_statistics *stats;

        stats = __schedstats_from_se(curr);
        __schedstat_set(stats->exec_max,
                max(delta_exec, stats->exec_max));
    }

    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq->exec_clock, delta_exec);

    curr->vruntime += calc_delta_fair(delta_exec, curr);
    update_min_vruntime(cfs_rq);

    if (entity_is_task(curr)) {
        struct task_struct *curtask = task_of(curr);

        trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
        cgroup_account_cputime(curtask, delta_exec);
        account_group_exec_runtime(curtask, delta_exec);
    }

    account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

这里假设`curr`实体对应的为一个任务，这个函数用于更新正在运行任务的时间统计，`delta_exec`指的是上次更新与这次更新的时间间隔，也就是两个更新期间进程运行的时间，任务的`exec_start`字段保存最近一次统计更新的时间，任务的`sum_exec_runtime`字段含义为累计任务累计执行时间。得到进程的执行时间之后，使用`calc_delta_fair`函数结合任务执行时间、任务权重计算出来任务虚拟运行时间的增量，更新任务的虚拟运行时间。调用`update_min_vruntime`函数更新CFS中任务所在rq对的虚拟运行时间基准，即更新rq的`min_vruntime`字段。`calc_delta_fair`、`update_min_vruntime`函数的内容在后边详细记录。

上述流程基于实体对应一个任务的情况，最后的`if`分支判断若当前运行的实体对应的不是一个任务而是其他级别更高的实体，例如正在运行的是一个线程组，则更新cgroup以及线程组的cpu占用时间，这里忽略具体的更新过程。判断一个实体是一个任务还是一个线程组是根据实体的`my_q`字段确定的，若这个字段为空则是一个任务，不为空则为一个线程组。

#### `calc_delta_fair`函数

```c
/*
 * delta /= w
 */
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

    return delta;
}
```

这里假设实体`se`对应的是一个任务，CFS之中`nice`值为0的任务比较特殊，它的虚拟运行时间和实际运行时间是相同的，不需要进行任何缩放，所以当任务的`nice`值为0的时候直接把运行时间当作虚拟运行时间。对于`nice`不为0的任务则需要对运行时间进行缩放得到对应对的虚拟运行时间，这个缩放操作主要在`__calc_delta`函数中进行，这个函数计算`delta * NICE_0_LOAD / load_of_task`，接下来记录这个函数的流程。

#### `__calc_delta`函数

```c
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
    u64 fact = scale_load_down(weight);
    u32 fact_hi = (u32)(fact >> 32);
    int shift = WMULT_SHIFT;
    int fs;

    __update_inv_weight(lw);

    if (unlikely(fact_hi)) {
        fs = fls(fact_hi);
        shift -= fs;
        fact >>= fs;
    }

    fact = mul_u32_u32(fact, lw->inv_weight);

    fact_hi = (u32)(fact >> 32);
    if (fact_hi) {
        fs = fls(fact_hi);
        shift -= fs;
        fact >>= fs;
    }

    return mul_u64_u32_shr(delta_exec, fact, shift);
}
```

- [ ] 补充这个函数的流程；

#### `update_min_vruntime`函数

```c
static void update_min_vruntime(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    struct rb_node *leftmost = rb_first_cached(&cfs_rq->tasks_timeline);

    u64 vruntime = cfs_rq->min_vruntime;

    if (curr) {
        if (curr->on_rq)
            vruntime = curr->vruntime;
        else
            curr = NULL;
    }

    if (leftmost) { /* non-empty tree */
        struct sched_entity *se = __node_2_se(leftmost);

        if (!curr)
            vruntime = se->vruntime;
        else
            vruntime = min_vruntime(vruntime, se->vruntime);
    }

    /* ensure we never gain time by being placed backwards. */
    u64_u32_store(cfs_rq->min_vruntime,
              max_vruntime(cfs_rq->min_vruntime, vruntime));
}
```

这里假设调度实体对应的是任务，在cfs_rq之中任务按照任务的`vruntime`的值存放在红黑树之中，红黑树中最左侧的任务为`vruntime`最小的任务。这个函数首先在当前运行任务、红黑树中最左侧任务的`vruntime`之中选择最小的值，然后选择当前cfs_rq之中的基准虚拟时间与刚刚选择出来的虚拟时间之中的最大值为当前cfs_rq之中的最新基准虚拟时间。

这里有一些特殊情况需要考虑，例如当cfs_rq之中没有任务的时候`curr`为空、红黑树为空、当前任务正在被调度过程中，导致无法在cfs_rq之中当前正在执行任务、红黑树最左侧任务中选出来一个最小的虚拟运行时间，此时只考虑前者或者后者的虚拟运行时间参与到最后的最大值比较中去。

#### `place_entity`函数

```c
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
    u64 vruntime = cfs_rq->min_vruntime;

    /*
     * The 'current' period is already promised to the current tasks,
     * however the extra weight of the new task will slow them down a
     * little, place the new task so that it fits in the slot that
     * stays open at the end.
     */
    if (initial && sched_feat(START_DEBIT))
        vruntime += sched_vslice(cfs_rq, se);

    /* sleeps up to a single latency don't count. */
    if (!initial) {
        unsigned long thresh;

        if (se_is_idle(se))
            thresh = sysctl_sched_min_granularity;
        else
            thresh = sysctl_sched_latency;

        /*
         * Halve their sleep time's effect, to allow
         * for a gentler effect of sleepers:
         */
        if (sched_feat(GENTLE_FAIR_SLEEPERS))
            thresh >>= 1;

        vruntime -= thresh;
    }

    /*
     * Pull vruntime of the entity being placed to the base level of
     * cfs_rq, to prevent boosting it if placed backwards.
     * However, min_vruntime can advance much faster than real time, with
     * the extreme being when an entity with the minimal weight always runs
     * on the cfs_rq. If the waking entity slept for a long time, its
     * vruntime difference from min_vruntime may overflow s64 and their
     * comparison may get inversed, so ignore the entity's original
     * vruntime in that case.
     * The maximal vruntime speedup is given by the ratio of normal to
     * minimal weight: scale_load_down(NICE_0_LOAD) / MIN_SHARES.
     * When placing a migrated waking entity, its exec_start has been set
     * from a different rq. In order to take into account a possible
     * divergence between new and prev rq's clocks task because of irq and
     * stolen time, we take an additional margin.
     * So, cutting off on the sleep time of
     *     2^63 / scale_load_down(NICE_0_LOAD) ~ 104 days
     * should be safe.
     */
    if (entity_is_long_sleeper(se))
        se->vruntime = vruntime;
    else
        se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```

假设这里的调度实体对应的是一个任务，这个函数计算并设置任务的虚拟运行时间。在任务模拟的虚拟运行时间为cfs_rq中的基准虚拟运行时间情况下，如果这个任务是新创建的任务、开启了`START_DEBIT`特性，为任务的虚拟运行时间增加一个时间片，该时间片由`sched_vslice`函数计算得到，后边详细记录此函数的流程。若任务刚刚被唤醒，则考虑减小任务的虚拟运行时间以增加任务运行的机会，根据任务是否是空闲任务、是否开启了`GENTLE_FAIR_SLEEPERS`确定虚拟运行时间减小的程度。若任务休眠时间很长则直接将任务虚拟运行时间直接设置为此函数中计算出来的`vruntime`，其他情况下取本函数中计算出来的虚拟运行时间、任务初始设置的虚拟运行时间（与当前正在运行任务的虚拟运行时间相同）。

在最后的流程中涉及到如何量化任务休眠时间很长这种情况，结合最后一个`if`分支前的注释内容理解。在不考虑溢出的情况下直接使用`max_vruntime`函数得到任务的虚拟运行时间，这个函数将两个时间做一个减法计算并判断结果是否小于0得到最大值。在考虑溢出的情况下则会直接给出一个错误的结果，面对这个问题内核考虑在什么情况下会发生溢出。最快的虚拟运行时间增加速度为`nice`值为0的任务带来的，当这样的任务运行大约104天时任务的虚拟运行时间接近最大值并且即将发生溢出，这个时候如何直接进行减法操作进行比较很容易得到错误的结果，104天就是内核对休眠很长时间的量化。对于这种情况内核直接将人物的虚拟运行时间设置为计算得到的虚拟运行时间以避免潜在错误。

#### `sched_vslice`函数

```c
/*
 * We calculate the vruntime slice of a to-be-inserted task.
 *
 * vs = s/w
 */
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    return calc_delta_fair(sched_slice(cfs_rq, se), se);
}
```

假设这里的调度实体对应的是一个任务，这个函数计算任务的虚拟时间片，具体思路是计算一个调度周期中这个任务获取到的运行时间并转换成虚拟运行时间。这个函数的一个用途是增加新创建任务的虚拟运行时间，防止新创建任务抢占cfs_rq中正在运行的任务。获取一个调度周期内任务获取到的时间由`sched_slice`函数给出，将这个时间转换成虚拟运行时间由`calc_delta_fair`函数给出，`calc_delta_fair`函数流程在前边记录过，接下来记录`sched_slice`函数的流程。

#### `sched_slice`函数

```c
/*
 * We calculate the wall-time slice from the period by taking a part
 * proportional to the weight.
 *
 * s = p*P[w/rw]
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    unsigned int nr_running = cfs_rq->nr_running;
    struct sched_entity *init_se = se;
    unsigned int min_gran;
    u64 slice;

    if (sched_feat(ALT_PERIOD))
        nr_running = rq_of(cfs_rq)->cfs.h_nr_running;

    slice = __sched_period(nr_running + !se->on_rq);

    for_each_sched_entity(se) {
        struct load_weight *load;
        struct load_weight lw;
        struct cfs_rq *qcfs_rq;

        qcfs_rq = cfs_rq_of(se);
        load = &qcfs_rq->load;

        if (unlikely(!se->on_rq)) {
            lw = qcfs_rq->load;

            update_load_add(&lw, se->load.weight);
            load = &lw;
        }
        slice = __calc_delta(slice, se->load.weight, load);
    }

    if (sched_feat(BASE_SLICE)) {
        if (se_is_idle(init_se) && !sched_idle_cfs_rq(cfs_rq))
            min_gran = sysctl_sched_idle_min_granularity;
        else
            min_gran = sysctl_sched_min_granularity;

        slice = max_t(u64, slice, min_gran);
    }

    return slice;
}
```

`__sched_period`函数返回一个调度周期对应的时间，这个时间的具体计算方式如下：如果cfs_rq之中运行的任务数量比较多，保证在平均意义下每个任务都获得最小的执行时间；其他情况下使用系统默认的调度周期对应的时间。接下来涉及到调度实体的层级结构，每个调度实体有自己的父实体，父实体也可能会有自己的父实体，从正在运行任务的实体开始验证子-父关系不断寻找父实体直到寻找到根实体（没有父实体），对于这个过程中寻找到的每个实体：若实体在cfs_rq之中则根据实体的权重、实体所在的cfs_rq中总权重对调度周期对应的事件进行缩放；其他情况下需要将实体的权重添加到cfs_rq的总权重之中，随后使用前一种情况中的处理方式。完成时间缩放之后得到了让任务的在这个调度周期中的运行时间。若启用了`BASE_SLICE`功能，为此任务的运行时间设置一个下限：若任务是一个空闲任务并且队列是一个空闲队列，下限设置为`sysctl_sched_idle_min_granularity`，其他情况下设置为`sysctl_sched_min_granularity`。

这里提到了调度实体的层级结构，这里提供一个层级结构的例子。在Linux桌面中可能会有多个会话、每个会话中启动shell、在shell中运行进程，这样就形成了一个从进程到会话的层级结构，就会产生调度实体之间的父-子关系。
