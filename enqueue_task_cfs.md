本文档记录`enqueue_task_fair`函数的流程，这个函数将调度实体`se`插入到CFS运行队列的红黑树之中，这个过程中还会更新一些统计字段。

### `enqueue_task_fair`函数

```c
/*
 * The enqueue_task method is called before nr_running is
 * increased. Here we update the fair scheduling stats and
 * then put the task into the rbtree:
 */
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &p->se;
    int idle_h_nr_running = task_has_idle_policy(p);
    int task_new = !(flags & ENQUEUE_WAKEUP);

    /*
     * The code below (indirectly) updates schedutil which looks at
     * the cfs_rq utilization to select a frequency.
     * Let's add the task's estimated utilization to the cfs_rq's
     * estimated utilization, before we update schedutil.
     */
    util_est_enqueue(&rq->cfs, p);

    /*
     * If in_iowait is set, the code below may not trigger any cpufreq
     * utilization updates, so do it here explicitly with the IOWAIT flag
     * passed.
     */
    if (p->in_iowait)
        cpufreq_update_util(rq, SCHED_CPUFREQ_IOWAIT);

    for_each_sched_entity(se) {
        if (se->on_rq)
            break;
        cfs_rq = cfs_rq_of(se);
        enqueue_entity(cfs_rq, se, flags);

        cfs_rq->h_nr_running++;
        cfs_rq->idle_h_nr_running += idle_h_nr_running;

        if (cfs_rq_is_idle(cfs_rq))
            idle_h_nr_running = 1;

        /* end evaluation on encountering a throttled cfs_rq */
        if (cfs_rq_throttled(cfs_rq))
            goto enqueue_throttle;

        flags = ENQUEUE_WAKEUP;
    }

    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);

        update_load_avg(cfs_rq, se, UPDATE_TG);
        se_update_runnable(se);
        update_cfs_group(se);

        cfs_rq->h_nr_running++;
        cfs_rq->idle_h_nr_running += idle_h_nr_running;

        if (cfs_rq_is_idle(cfs_rq))
            idle_h_nr_running = 1;

        /* end evaluation on encountering a throttled cfs_rq */
        if (cfs_rq_throttled(cfs_rq))
            goto enqueue_throttle;
    }

    /* At this point se is NULL and we are at root level*/
    add_nr_running(rq, 1);

    if (!task_new)
        check_update_overutilized_status(rq);

enqueue_throttle:
    assert_list_leaf_cfs_rq(rq);

    hrtick_update(rq);
}
```

这里先关注两个循环（`for_each_sched_entity`）中的内容，这两个循环涉及到内核的任务组调度功能，关于这个功能的详细介绍见[task_group](https://blogs.oracle.com/linux/post/cfs-group-scheduling)，从介绍中可以看到在任务组调度功能之中，调度实体被划分到了多个层级，有的层次中调度实体对应一个任务组、有的层级中的调度实体对应一个任务，这两个`for`循环都是对于任务`p`对应的调度实体以及上级每个层级与之关联的调度实体而言的。第一个`for`循环用于将`p`对应的实体以及每个层级中与这个实体关联的实体放入到运行队列之中，第二个`for`循环用于更新平均负载、任务组的可运行任务相关统计字段、重新计算任务组实体的权重。两个循环之后调用`add_nr_running`更新CFS运行队列中正在运行任务的计数并进行超载检查，最后调用`hrtick_update`设置新的高精度定时器。

这里仅仅是简单整理一下这个函数的整体流程，后边会具体记录这其中许多函数的流程，被忽略的函数除外。这个函数中还隐藏着一个有意思的处理方式，这种涉及到了两个字段的更新（`h_nr_running`、`idle_h_nr_running`），这两个字段为CFS运行队列的统计字段，代表了在当前的调度层级以及下层之中正在运行的实体、优先级极低（任务的调度策略被指定为了`SCHED_IDLE`）的任务。先以`h_nr_running`为例，当任务`p`对应的实体以及调度层级中与之关联的实体都放入到CFS运行队列之中时，每个层级之中新增的正在运行的实体为2，但是在第一个`for`循环之中只为每一层的CFS运行队列中这个字段递增，这个看起来很奇怪，因为这里需要2次递增，直到看到第二个`for`循环之中的代码才知道第二次递增在这个循环之中。`idle_h_nr_running`的数值更新逻辑类似，也是分到了两个`for`循环之中完成数值递增，只不过在某个层级调度实体对应的任务组为`SCHED_IDLE`组时才开始更新。

### `enqueue_entity`函数

```c
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
    bool curr = cfs_rq->curr == se;

    /*
     * If we're the current task, we must renormalise before calling
     * update_curr().
     */
    if (renorm && curr)
        se->vruntime += cfs_rq->min_vruntime;

    update_curr(cfs_rq);

    /*
     * Otherwise, renormalise after, such that we're placed at the current
     * moment in time, instead of some random moment in the past. Being
     * placed in the past could significantly boost this task to the
     * fairness detriment of existing tasks.
     */
    if (renorm && !curr)
        se->vruntime += cfs_rq->min_vruntime;

    /*
     * When enqueuing a sched_entity, we must:
     *   - Update loads to have both entity and cfs_rq synced with now.
     *   - For group_entity, update its runnable_weight to reflect the new
     *     h_nr_running of its group cfs_rq.
     *   - For group_entity, update its weight to reflect the new share of
     *     its group cfs_rq
     *   - Add its new weight to cfs_rq->load.weight
     */
    update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH);
    se_update_runnable(se);
    update_cfs_group(se);
    account_entity_enqueue(cfs_rq, se);

    if (flags & ENQUEUE_WAKEUP)
        place_entity(cfs_rq, se, 0);
    /* Entity has migrated, no longer consider this task hot */
    if (flags & ENQUEUE_MIGRATED)
        se->exec_start = 0;

    check_schedstat_required();
    update_stats_enqueue_fair(cfs_rq, se, flags);
    check_spread(cfs_rq, se);
    if (!curr)
        __enqueue_entity(cfs_rq, se);
    se->on_rq = 1;

    if (cfs_rq->nr_running == 1) {
        check_enqueue_throttle(cfs_rq);
        if (!throttled_hierarchy(cfs_rq))
            list_add_leaf_cfs_rq(cfs_rq);
    }
}
```

这个函数将调度实体`se`放入到CFS运行队列之中，在放入到CFS运行队列前后需要更新许多任务相关的统计内容。`update_curr`函数的流程已经在`task_fork_cfs.md`中有记录，这个函数主要是更新正在运行任务的时间统计，在这个函数前后有两个`if`判断，这两个`if`判断成立时执行的操作都是将实体`se`的相对虚拟运行时间调整成虚拟运行时间，但是一个在`update_curr`函数之前、另外一个在这个函数之后，这背后的原因是：当第一个`if`判断成立时`se`对应的任务为CFS运行队列之中正在运行的任务，更新这个任务的虚拟运行时间可能会影响`update_curr`函数中CFS运行队列中虚拟运行时间基准的计算（`min_vruntime`），所以需要先更新任务的虚拟运行时间然后调用`update_curr`；如果`se`对应的任务不是CFS运行队列中正在运行的任务，这个任务目前为止还没有加入到CFS运行队列中，但是需要CFS运行队列之中的虚拟运行时间基准来计算虚拟运行时间，所以需要在`update_curr`更新CFS运行队列的虚拟运行时间基准之后执行。两个`if`判断之中还会涉及到`renorm`是否为`True`，它为`True`意味着`se`对应的任务不是被唤醒的任务或者从其他的运行队列中迁移过来的任务。

从`update_load_avg`函数到`account_entity_queueue`函数更新任务以及CFS运行队列的平均负载、更新任务组对的`runnable_weight`、重新计算任务组的权重、更新CFS运行队列中运行任务的权重之和等操作，这些更新都封装到了不同的函数之中，后边会对这些函数的流程进行记录。如果任务刚刚被唤醒，那么调用`place_entity`函数更新任务的虚拟运行时间，这个函数的流程在`task_fork_cfs.md`之中有记录；如果任务从其他的cpu之中迁移过来则重置`exec_start`字段的值。接下来使用`__enqueue_entity`函数将实体加入到CFS运行队列之中，添加任务在运行队列中的标记(`on_rq`更新为1)，最后将CFS运行队列放入到`leaf_cfs_rq`之中，在`list_add_leaf_cfs_rq`函数中解释这个list的作用。

### `update_load_avg`函数

```c
/* Update task and its cfs_rq load average */
static inline void update_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    u64 now = cfs_rq_clock_pelt(cfs_rq);
    int decayed;

    /*
     * Track task load average for carrying it to new CPU after migrated, and
     * track group sched_entity load average for task_h_load calc in migration
     */
    if (se->avg.last_update_time && !(flags & SKIP_AGE_LOAD))
        __update_load_avg_se(now, cfs_rq, se);

    decayed  = update_cfs_rq_load_avg(now, cfs_rq);
    decayed |= propagate_entity_load_avg(se);

    if (!se->avg.last_update_time && (flags & DO_ATTACH)) {

        /*
         * DO_ATTACH means we're here from enqueue_entity().
         * !last_update_time means we've passed through
         * migrate_task_rq_fair() indicating we migrated.
         *
         * IOW we're enqueueing a task on a new CPU.
         */
        attach_entity_load_avg(cfs_rq, se);
        update_tg_load_avg(cfs_rq);

    } else if (flags & DO_DETACH) {
        /*
         * DO_DETACH means we're here from dequeue_entity()
         * and we are migrating task out of the CPU.
         */
        detach_entity_load_avg(cfs_rq, se);
        update_tg_load_avg(cfs_rq);
    } else if (decayed) {
        cfs_rq_util_change(cfs_rq, 0);

        if (flags & UPDATE_TG)
            update_tg_load_avg(cfs_rq);
    }
}
```

这个函数用于更新任务和CFS运行队列的平均负载，负载可以理解为时间占用，在后边的流程中可以看到时间占用大致分为在运行队列中的时间以及在cpu中运行的时间。这个函数首先使用`cfs_rq_clock_pelt`函数获取现在的时间，这个函数返回的时间不是通用的时间，而是计算任务负载专用的时间，这个时间是经过真实的时间根据cpu的`capacity`以及`frequency`进行缩放之后得到，考虑了一颗cpu中不同核心计算能力的差异，使得在不同的核心中使用相同的标准计算负载。这个时间的更新在`update_rq_clock_pelt`函数之中，这个函数的流程记录在`task_wake_up.md`文件中可以找到。

`se->avg.last_update_time`记录了上一次负载统计更新的时间，不为0时表示这个实体对应的任务不是从其他运行队列之中迁移过来的，此时若没有要求跳过负载统计更新调用`___update_load_avg_se`更新实体的平均负载统计。这个函数里涉及到了计算任务负载的指数加权移动平均，在后边会详细记录这个函数中的流程。更新完实体`se`的负载统计之后，调用`update_cfs_rq_load_avg`函数更新当前CFS运行队列的负载统计。考虑到组调度特性，调度实体`se`位于调度层级中的某一层次，调度实体负载统计的变化也需要向上级传递，更新上级中的负载统计，这个过程由`propagate_entity_load_avg`函数完成。

当`se->avg.last_update_time`为0表示这个任务是从其他的运行队列之中迁移过来的，当`flags & DO_ATTACH`不为0时表示这个函数是在任务放入到CFS运行队列过程中调用的，调用`attach_entity_load_avg`函数将任务的平均负载添加到CFS运行队列的平均负载统计之中，调用`update_tg_load_avg`函数更新任务组的平均负载统计。当`flags & DO_DETACH`不为0时表示这个任务将要迁移到其他的运行队列之中，这个时候要从CFS运行队列的平均负载统计中减去此任务的贡献，然后更新任务组的平均负载统计。如果在之前的更新过程执行过指数衰减加权平均计算，调用`cfs_rq_util_change`更新cpu调频所需数据、更新线程组的平均负载。

上边的逻辑中涉及到了许多函数，大部分函数流程都在后边详细记录，剩下的函数忽略。

### `__update_load_avg_se`函数

```c
int __update_load_avg_se(u64 now, struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    if (___update_load_sum(now, &se->avg, !!se->on_rq, se_runnable(se),
                cfs_rq->curr == se)) {

        ___update_load_avg(&se->avg, se_weight(se));
        cfs_se_util_change(&se->avg);
        trace_pelt_se_tp(se);
        return 1;
    }

    return 0;
}
```

这个函数用于更新实体`se`的平均负载，首先调用`___update_load_sum`函数计算实体`se`的`load_sum`、`runnable_sum`、`util_sum`这三个指标的值，这三个指标之中load_sum基于可运行时间计算（包含在CFS运行队列的时间、cpu上执行的时间），`util_sum`基于在cpu上的执行时间计算，`runnable_sum`同样基于任务的可运行时间计算但是计算细节与`load_sum`的计算有所不同，这三个sum指标为关于实体两次更新三个指标时间差的几何级数之和，具体计算方式在`___update_load_sum`函数流程记录之中说明。更新完``{load、runnable、util}_sum`之后调用`__update_load_avg`计算`{load,runnable, util}_avg`三个指标，这三个指标为三个对应几何级数之后的均值，具体计算方式在这个函数的流程记录中说明。最后调用`cfs_se_util_change`函数标记`util_avg`已经更新过。

### `___update_load_sum`函数

```c
static __always_inline int
___update_load_sum(u64 now, struct sched_avg *sa,
          unsigned long load, unsigned long runnable, int running)
{
    u64 delta;

    delta = now - sa->last_update_time;
    /*
     * This should only happen when time goes backwards, which it
     * unfortunately does during sched clock init when we swap over to TSC.
     */
    if ((s64)delta < 0) {
        sa->last_update_time = now;
        return 0;
    }

    /*
     * Use 1024ns as the unit of measurement since it's a reasonable
     * approximation of 1us and fast to compute.
     */
    delta >>= 10;
    if (!delta)
        return 0;

    sa->last_update_time += delta << 10;

    /*
     * running is a subset of runnable (weight) so running can't be set if
     * runnable is clear. But there are some corner cases where the current
     * se has been already dequeued but cfs_rq->curr still points to it.
     * This means that weight will be 0 but not running for a sched_entity
     * but also for a cfs_rq if the latter becomes idle. As an example,
     * this happens during idle_balance() which calls
     * update_blocked_averages().
     *
     * Also see the comment in accumulate_sum().
     */
    if (!load)
        runnable = running = 0;

    /*
     * Now we know we crossed measurement unit boundaries. The *_avg
     * accrues by two steps:
     *
     * Step 1: accumulate *_sum since last_update_time. If we haven't
     * crossed period boundaries, finish.
     */
    if (!accumulate_sum(delta, sa, load, runnable, running))
        return 0;

    return 1;
}
```

这个函数计算`load_sum`、`runnable_sum`、`util_sum`三个指标，这里先记录这三个指标的计算过程，然后解释这三个指标的含义，前边提到的负载统计其实就是这三个指标。这个函数将时间按照1024 ns为一个周期划分，`delta`为实体两个统计更新之间的时间差，这个时间差可能会出现在多个周期之中，第一个周期为距离现在最远的周期，最后一个周期为`now`所在的周期，在第一个周期的时间部分并不一定会占用整个周期，最后一个周期的时间也并不一定会占用整个周期。这里假设时间差出现在p个完整的周期，加上两个特殊的周期，一共出现了在p+2个周期之中，将在每个周期中的时间设置$u_i$，其中有p个$u_i$为1024而额外两个$u_i$的值小于1024，$u_0$为在第一个周期的时间占用，$u_{p+1}$为在最后一个周期的时间占用。接下来将这些$p+2$个$u_i$当作$p+2$个几何级数的系数进行求和，具体计算方式为$u_0 + y_1 * y + u_2 * y^2 + u_3 * ^3+ ...$，y的取值满足$y^{32}$为0.5。由于y的取值小于1，使用几何级数求和意味着在结果中距离现在越近的周期对应的u_i在结果中的比例越高，距离现在越远的周期对应的u_i在结果中的比例越低，例如$u_0$的系数为1而$u_{31}$的系数为0.5。

在`task_fork_cfs.md`之中记录`__calc_delta`函数的时候提到过内核无法直接进行浮点数计算，因此这里的几何级数求和也无法直接进行计算，内核解决这个问题的方法是选择y使得$y^{32}$为0.5、在内核源码中保存${y, y^2,...,y^{31}}$的数值，这个时候若要计算$u_i * y^p$，需要考虑p是否大于32：如果$p < 32$，则$y^p$可以直接获取；如果 $p > 32$，则可以使用如下方法：假设 $p = m * 32 + n$，那么$y^p = (y^{32})^m * y^n$ 由于 $y^{32} = \frac{1}{2}$，所以$y^p = (\frac{1}{2})^m * y^n$（因为$n < 32$所以$y^n$的值可以直接获取）。上述过程只能计算出来单个$y^n$的值，在几何级数求和公式中还存在多个不同技术结果的求和，如下所示：
$$
\sum_{n=1}^{p-1} y^n
$$

求和公式的计算可以转换为：

$$
等式1：\sum_{n=1}^{p-1} y^n = \sum_{n=1}^{∞}y^n - \sum_{n=p}^{∞}y^n - 1
$$

其中：

$$
等式2：\sum_{n=1}^{∞}y^n = \frac{1}{1 - y}
$$

$$
等式3：\sum_{n=p}^{∞}y^n = \frac{1}{1 - y} - \sum_{n = 1}^{p - 1} y^n
$$

根据等比数列求和公式有：

$$
等式4： \sum_{n=1}^{p - 1} y^n = \frac{1-y^p}{1-y}
$$

根据等式3、4有：

$$
等式5： \sum_{n=p}^{∞}y^n = \frac{1}{1 - y} - \sum_{n = 1}^{p - 1} y^n = \frac{1}{1 - y} - \frac{1-y^p}{1-y} = \frac{y^p}{1-y}
$$

等式2来源于极限计算，综合等式1、2、5得到：

$$
等式6： \sum_{n=1}^{p-1} y^n = \frac{1}{1 - y} - \frac{1}{1 - y} * y^p - 1
$$
接下来关注这个函数的流程，这个函数进行一些判断最后调用`accumulate_sum`进行几何级数求和计算。这个函数首先计算两次更新之间的时间差存入`delta`之中，若`delta`小于0或者`delta`跨越的时间不足一个周期直接退出函数，其他情况下继续执行。接下来的代码涉及到`load`、`runnable`、`running`三个参数：当实体`se`在CFS运行队列之中时`load`为1；当实体对应一个任务并且实体在CFS运行队列之中时`runnable`为1，当实体对饮一个线程组并且实习在CFS运行队列之中时`runnable`为任务组中可运行的任务数量；当实体对应的任务正在运行时`running`为1。当实体对应任务时，`load`和`runnable`的含义是一致的，即任务是否处于可运行状态；当实体对应任务组时，`load`对应任务组是否处于可运行状态、`runnable`为任务组中可运行的任务，在接下来的流程之中默认实体既可以对应任务也可以对应任务组，当对应任务与任务组处理方式不同时会有特殊说明。代码注释之中提到一种特殊的情况需要处理，即当`load`为0时意味着实体不在CFS运行队列之中，那么在一些情况之中可能会认为实体对应的任务正在运行中，这个时候需要纠正错误的判断。接下来调用`accumulate_sum`进行几何级数求和计算，接下来记录的函数流程就是这个函数的流程，在这个函数之中能真正看到`load_sum`、`runnable_sum`、`util_sum`的计算方式，也会明确这三个指标的含义是什么。

### `accumulate_sum`函数

```c
/*
 * Accumulate the three separate parts of the sum; d1 the remainder
 * of the last (incomplete) period, d2 the span of full periods and d3
 * the remainder of the (incomplete) current period.
 *
 *           d1          d2           d3
 *           ^           ^            ^
 *           |           |            |
 *         |<->|<----------------->|<--->|
 * ... |---x---|------| ... |------|-----x (now)
 *
 *                           p-1
 * u' = (u + d1) y^p + 1024 \Sum y^n + d3 y^0
 *                           n=1
 *
 *    = u y^p +					(Step 1)
 *
 *                     p-1
 *      d1 y^p + 1024 \Sum y^n + d3 y^0		(Step 2)
 *                     n=1
 */
static __always_inline u32
accumulate_sum(u64 delta, struct sched_avg *sa,
	       unsigned long load, unsigned long runnable, int running)
{
	u32 contrib = (u32)delta; /* p == 0 -> delta < 1024 */
	u64 periods;

	delta += sa->period_contrib;
	periods = delta / 1024; /* A period is 1024us (~1ms) */

	/*
	 * Step 1: decay old *_sum if we crossed period boundaries.
	 */
	if (periods) {
		sa->load_sum = decay_load(sa->load_sum, periods);
		sa->runnable_sum =
			decay_load(sa->runnable_sum, periods);
		sa->util_sum = decay_load((u64)(sa->util_sum), periods);

		/*
		 * Step 2
		 */
		delta %= 1024;
		if (load) {
			/*
			 * This relies on the:
			 *
			 * if (!load)
			 *	runnable = running = 0;
			 *
			 * clause from ___update_load_sum(); this results in
			 * the below usage of @contrib to disappear entirely,
			 * so no point in calculating it.
			 */
			contrib = __accumulate_pelt_segments(periods,
					1024 - sa->period_contrib, delta);
		}
	}
	sa->period_contrib = delta;

	if (load)
		sa->load_sum += load * contrib;
	if (runnable)
		sa->runnable_sum += runnable * contrib << SCHED_CAPACITY_SHIFT;
	if (running)
		sa->util_sum += contrib << SCHED_CAPACITY_SHIFT;

	return periods;
}
```

参照这个函数前的注释，当`delta`分布在`p+2`个周期时函数执行的计算如下：
$$
等式7：u' = (u_{p+1} + d_1) * y^p + 1024 * \sum_{n=1}^{p} y^n + p_0
$$
其中$u_{p+1}$、$p_0$的含义见`___update_load_sum`函数流程记录、$d_1$的含义为之前计算出来的指标值，代码实现的时候将这个公式的计算分成了两部分，第一部分为$d1 * y^p$、第二部分为等式7之中剩余的部分，这两部分的计算都依赖于`delta`以及`periods`两个变量的值：`delta`为两次统计之间的时间差，将`delta`与上一次统计时$p_0$的值`sa->peroid_contrib`相加得到更准确的时间，之所以这样处理时因为在`___update_load_sum`函数中更新`last_update_time`时直接使用左移操作进行时间换算而忽略了这部分时间；`peroids`为`delta`分布在多少个周期之中。

第一部分计算分别将任务之前计算得到的`load_sum`、`runnable_sum`、`util_sum`当作$d_1$ 计算$d_1 * y^p$，具体的计算由`delay_load`函数完成，这个函数的流程与`___update_load_sum`函数中记录的计算$u_i * y^p$的思路一致，不再详细记录`delay_load`这个函数的代码流程了。第二部分调用`__accumulate_pelt_segments`函数完成计算得到`delta`对应这段时间的几何级数求和，这个函数实现流程涉及到前边提到的等式1-6，会在后边会记录这个函数的流程。

`__accumulate_pelt_segments`函数会给出`delta`对应的这段时间的几何级数之和，几何级数之和会存入到`contrib`之中，最后的三个`if`判断中的代码更新实体的三个指标，指标的更新取决于`load`、`runnable`、`running`三个参数是否为0，若为0则跳过指标更新。从这三个指标更新的代码之中可以看到：`load_sum`为实体处于可运行状态（`load`不为0）时占用时间的几何级数之和；当实体对应一个可运行任务时`runnable_sum`与`load_sum`含义一致，对应`runnable`不为0，只不过把它当作定点数然后缩放成整数；当实体对应可运行的任务组时，同样对应`runnable`不为0，`runnable_sum`为任务组中为任务组中可运行任务数量、任务组处于可运行状态时占用时间的几何级数之和两个因素的乘积，将乘积结果当作定点数随后转换成整数；`util_sum`为任务处于实体处于运行状态时占用时间的几何级数之和。

最后注意到将`delta % 1024`赋值给了`sa->period_contrib`，这个对应保存本次更新时的$P_0$，在下次更新时会将下次更新时计算出来的`delta`与本次更新时计算出来的$P_0$相加作为更精确的时间，见本函数最开始部分对`delta`变量的解释。

### `__accmulate_pelt_segments`函数

```c
static u32 __accumulate_pelt_segments(u64 periods, u32 d1, u32 d3)
{
	u32 c1, c2, c3 = d3; /* y^0 == 1 */

	/*
	 * c1 = d1 y^p
	 */
	c1 = decay_load((u64)d1, periods);

	/*
	 *            p-1
	 * c2 = 1024 \Sum y^n
	 *            n=1
	 *
	 *              inf        inf
	 *    = 1024 ( \Sum y^n - \Sum y^n - y^0 )
	 *              n=0        n=p
	 */
	c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024;

	return c1 + c2 + c3;
}
```

这个函数计算等式7之中除去$d_1 * y^p$ 之后剩余部分的和，这个函数之中`c2`的计算对应等式6，`c1`的计算对应等式7中$u_{p+1} * y^n$，涉及到幂计算的时候直接调用`delay_load`函数进行，这个函数的思路在`accumulate_sum`函数中有记录。这个函数的参数之中`periods`为`delta`分布在多少个周期之中，`d1`、`d3`分别对应`___update_load_sum`函数记录中的$u_{p+1}$、$u_0$。

### `___update_load_avg`函数

```c
/*
 * When syncing *_avg with *_sum, we must take into account the current
 * position in the PELT segment otherwise the remaining part of the segment
 * will be considered as idle time whereas it's not yet elapsed and this will
 * generate unwanted oscillation in the range [1002..1024[.
 *
 * The max value of *_sum varies with the position in the time segment and is
 * equals to :
 *
 *   LOAD_AVG_MAX*y + sa->period_contrib
 *
 * which can be simplified into:
 *
 *   LOAD_AVG_MAX - 1024 + sa->period_contrib
 *
 * because LOAD_AVG_MAX*y == LOAD_AVG_MAX-1024
 *
 * The same care must be taken when a sched entity is added, updated or
 * removed from a cfs_rq and we need to update sched_avg. Scheduler entities
 * and the cfs rq, to which they are attached, have the same position in the
 * time segment because they use the same clock. This means that we can use
 * the period_contrib of cfs_rq when updating the sched_avg of a sched_entity
 * if it's more convenient.
 */
static __always_inline void
___update_load_avg(struct sched_avg *sa, unsigned long load)
{
	u32 divider = get_pelt_divider(sa);

	/*
	 * Step 2: update *_avg.
	 */
	sa->load_avg = div_u64(load * sa->load_sum, divider);
	sa->runnable_avg = div_u64(sa->runnable_sum, divider);
	WRITE_ONCE(sa->util_avg, sa->util_sum / divider);
}
```

计算出来三个`sum`指标之后，这个函数计算这三个指标对应的`avg`指标，三个`avg`指标涉及到三个`sum`指标除以`get_pelt_divider`函数返回的值，这个函数返回了几何级数和的极限值与`___update_load_sum`函数流程记录中提到的`u_0`的和，这个值可以理解为最大值。这里几何级数之和的极限可以理解为几何级数之和的最大值，结合注释可以理解为将这个最大值和$u_0$求和为了防止将$u_0$这段时间当做空闲时间而带来不必要的振荡，至于为什么这么做会产生振荡可能是作者在做实验的过程中发现的。

`load_avg`指标是实体权重与`load_sum`之积然后除以最大值的结果，`runnable_avg`以及`util_avg`为对应的`sum`指标除以最大值的结果。三个`avg`指标计算很简单，但是为什么除以最大值更值得关注，查阅资料之后认为通过除以最大值可以将计算结果标准化、实现旧的值随着时间推移的衰减、防止累计值无限扩大。

### `get_pelt_divider`函数

```c
#define PELT_MIN_DIVIDER	(LOAD_AVG_MAX - 1024)

static inline u32 get_pelt_divider(struct sched_avg *avg)
{
	return PELT_MIN_DIVIDER + avg->period_contrib;
}
```

这个函数用于返回一个最大值，计算方法为：
$$
\sum_{n=1}^{∞}y^n * y * 1024 + u_0
$$
其中：
$$
\sum_{n=1}^{∞} y^n = \frac{1}{1-y}
$$

$$
等式8： \sum_{n=1}^{∞} y^n * y * 1024 = \frac{1}{1-y} * y * 1024 = \frac{1}{1-y} * 1024 - 1024
$$

`PELT_MIN_DIVIDER`代表的值对应等式8的结果，`avg->period_contrib`对应$u_0$。

### `update_cfs_rq_load_avg`函数

```c
/**
 * update_cfs_rq_load_avg - update the cfs_rq's load/util averages
 * @now: current time, as per cfs_rq_clock_pelt()
 * @cfs_rq: cfs_rq to update
 *
 * The cfs_rq avg is the direct sum of all its entities (blocked and runnable)
 * avg. The immediate corollary is that all (fair) tasks must be attached.
 *
 * cfs_rq->avg is used for task_h_load() and update_cfs_share() for example.
 *
 * Return: true if the load decayed or we removed load.
 *
 * Since both these conditions indicate a changed cfs_rq->avg.load we should
 * call update_tg_load_avg() when this function returns true.
 */
static inline int
update_cfs_rq_load_avg(u64 now, struct cfs_rq *cfs_rq)
{
	unsigned long removed_load = 0, removed_util = 0, removed_runnable = 0;
	struct sched_avg *sa = &cfs_rq->avg;
	int decayed = 0;

	if (cfs_rq->removed.nr) {
		unsigned long r;
		u32 divider = get_pelt_divider(&cfs_rq->avg);

		raw_spin_lock(&cfs_rq->removed.lock);
		swap(cfs_rq->removed.util_avg, removed_util);
		swap(cfs_rq->removed.load_avg, removed_load);
		swap(cfs_rq->removed.runnable_avg, removed_runnable);
		cfs_rq->removed.nr = 0;
		raw_spin_unlock(&cfs_rq->removed.lock);

		r = removed_load;
		sub_positive(&sa->load_avg, r);
		sub_positive(&sa->load_sum, r * divider);
		/* See sa->util_sum below */
		sa->load_sum = max_t(u32, sa->load_sum, sa->load_avg * PELT_MIN_DIVIDER);

		r = removed_util;
		sub_positive(&sa->util_avg, r);
		sub_positive(&sa->util_sum, r * divider);
		/*
		 * Because of rounding, se->util_sum might ends up being +1 more than
		 * cfs->util_sum. Although this is not a problem by itself, detaching
		 * a lot of tasks with the rounding problem between 2 updates of
		 * util_avg (~1ms) can make cfs->util_sum becoming null whereas
		 * cfs_util_avg is not.
		 * Check that util_sum is still above its lower bound for the new
		 * util_avg. Given that period_contrib might have moved since the last
		 * sync, we are only sure that util_sum must be above or equal to
		 *    util_avg * minimum possible divider
		 */
		sa->util_sum = max_t(u32, sa->util_sum, sa->util_avg * PELT_MIN_DIVIDER);

		r = removed_runnable;
		sub_positive(&sa->runnable_avg, r);
		sub_positive(&sa->runnable_sum, r * divider);
		/* See sa->util_sum above */
		sa->runnable_sum = max_t(u32, sa->runnable_sum,
					      sa->runnable_avg * PELT_MIN_DIVIDER);

		/*
		 * removed_runnable is the unweighted version of removed_load so we
		 * can use it to estimate removed_load_sum.
		 */
		add_tg_cfs_propagate(cfs_rq,
			-(long)(removed_runnable * divider) >> SCHED_CAPACITY_SHIFT);

		decayed = 1;
	}

	decayed |= __update_load_avg_cfs_rq(now, cfs_rq);
	u64_u32_store_copy(sa->last_update_time,
			   cfs_rq->last_update_time_copy,
			   sa->last_update_time);
	return decayed;
}
```

这个函数用于更新CFS运行队列上的`runnable_avg`、`util_avg`、`load_avg`三个指标的值。这个函数首先处理当CFS运行队列之中有实体离开的情况（`cfs_rq->removed.nr`不为0），分别更新`runnable`、`util`、`load`这三个指标的`avg`以及`sum`的值，更新`avg`时直接减去因任务离开CFS运行队列而减少的值均值，更新`sum`时则要减去减少的均值与指标最大值的乘积。这里需要注意的是在`sum`指标更新之后需要保证`sum`指标的最小值，如代码中的注释所说，这是因为在两次更新这些`sum`指标的时候可能会有大量的任务从CFS运行队列之中移除，这就会导致CFS运行队列的`avg`指标不为0但是`sum`指标为0的情况，此时需要根据`avg`指标的值来修正`sum`指标的值。当这一层之中有实体移除时，不仅仅是这一层的指标会发生变化，上层的指标也会发生变化，`add_tg_cfs_propagate`函数将被移除实体的`runnable_avg`暂存起来，在后续的流程中更高的调度层级之中传递这个变化。

更新CFS运行队列的`avg`指标的流程在`__update_load_avg_cfs_rq`函数之中，在后边记录这个函数的流程。

### `add_tg_cfs_propagate`函数

```c
static inline void add_tg_cfs_propagate(struct cfs_rq *cfs_rq, long runnable_sum)
{
	cfs_rq->propagate = 1;
	cfs_rq->prop_runnable_sum += runnable_sum;
}
```

这个函数之中将需要向调度层级之中的上级传播的`runnable_sum`指标缓存到CFS运行队列的`prop_runnable_sum`，在`update_tg_cfs_load`函数之中会使用这个字段向上级传播`runnable_sum`的变化，详见这个函数的流程记录。

### `__upate_load_avg_cfs_rq`函数

```c
int __update_load_avg_cfs_rq(u64 now, struct cfs_rq *cfs_rq)
{
	if (___update_load_sum(now, &cfs_rq->avg,
				scale_load_down(cfs_rq->load.weight),
				cfs_rq->h_nr_running,
				cfs_rq->curr != NULL)) {

		___update_load_avg(&cfs_rq->avg, 1);
		trace_pelt_cfs_tp(cfs_rq);
		return 1;
	}

	return 0;
}
```

这个函数之中调用的函数与`__update_load_avg_se`函数调用的函数完全一致，只不过传入的参数有所不同。调用`___update_load_sum`时`load`为CFS运行队列的权重、`runnable`为CFS运行队列之中可运行的任务，调用`___update_load_avg`的时候传入的是CFS运行队列的指标，具体计算逻辑见前边这两个函数的流程记录。从这个函数中可以看到`___update_load_sum`、`___update_load_avg`函数既可以更新任务的指标又可以更新CFS运行队列的指标，在前边记录这两个函数的流程时仅仅关注了这两种情况下`runnable`参数的不同，这里可以看到如果更新的是CFS运行队列的指标`load`参数的值为CFS运行队列的权重（CFS运行队列之中所有任务的权重和），如果更新的是可运行的任务任务则为1。

### `propagate_entity_load_avg`函数

```c
/* Update task and its cfs_rq load average */
static inline int propagate_entity_load_avg(struct sched_entity *se)
{
	struct cfs_rq *cfs_rq, *gcfs_rq;

	if (entity_is_task(se))
		return 0;

	gcfs_rq = group_cfs_rq(se);
	if (!gcfs_rq->propagate)
		return 0;

	gcfs_rq->propagate = 0;

	cfs_rq = cfs_rq_of(se);

	add_tg_cfs_propagate(cfs_rq, gcfs_rq->prop_runnable_sum);

	update_tg_cfs_util(cfs_rq, se, gcfs_rq);
	update_tg_cfs_runnable(cfs_rq, se, gcfs_rq);
	update_tg_cfs_load(cfs_rq, se, gcfs_rq);

	trace_pelt_cfs_tp(cfs_rq);
	trace_pelt_se_tp(se);

	return 1;
}
```

这个函数在实体对应一个任务组时向调度层级之中的上级传递`util`、`runnable`、`load`指标的变化，在记录记录的更新函数之前需要先明确三个变量的含义：`cfs_rq`为任务组所在的CFS运行队列，`se`为任务组自身、`gcfs_rq`之中保存了任务组之中的任务。

首先使用`add_tg_cfs_propagate`函数将任务组之中任务的`runnable_sum`指标变动暂存到任务组所在的CFS运行队列之中，然后调用`update_tg_cfs_{util, runnable, load}`三个函数更新任务组、任务组所在CFS运行队列中三个指标的值，具体更新流程在这三个函数之中说明。

### `update_tg_cfs_util`函数

```c
static inline void
update_tg_cfs_util(struct cfs_rq *cfs_rq, struct sched_entity *se, struct cfs_rq *gcfs_rq)
{
	long delta_sum, delta_avg = gcfs_rq->avg.util_avg - se->avg.util_avg;
	u32 new_sum, divider;

	/* Nothing to update */
	if (!delta_avg)
		return;

	/*
	 * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
	 * See ___update_load_avg() for details.
	 */
	divider = get_pelt_divider(&cfs_rq->avg);


	/* Set new sched_entity's utilization */
	se->avg.util_avg = gcfs_rq->avg.util_avg;
	new_sum = se->avg.util_avg * divider;
	delta_sum = (long)new_sum - (long)se->avg.util_sum;
	se->avg.util_sum = new_sum;

	/* Update parent cfs_rq utilization */
	add_positive(&cfs_rq->avg.util_avg, delta_avg);
	add_positive(&cfs_rq->avg.util_sum, delta_sum);

	/* See update_cfs_rq_load_avg() */
	cfs_rq->avg.util_sum = max_t(u32, cfs_rq->avg.util_sum,
					  cfs_rq->avg.util_avg * PELT_MIN_DIVIDER);
}
```

这个函数更新任务组以及任务组所在CFS运行队列的`util`指标，三个参数的含义在`propagate_entity_load_avg`函数的流程之中已经说明。这个函数分为5步：1).计算任务组与任务组之中所有任务的`util_avg`的变化，2).将任务组的`util_avg`指标设置为任务组之中所有任务的`util_avg`指标，3).计算任务组的`util_sum`指标并且得到这个指标的变化，4).将变化更新到任务组所在的CFS运行队列之中，5).保证任务组所在CFS运行队列之中`util_sum`的最小值。`update_tg_cfs_runnable`函数更新任务组以及任务组所在CFS运行队列的`runnable`相关指标，计算流程与这个函数完全一样，只不过是处理的指标不同，不在单独记录这个函数的流程。

### `update_tg_cfs_load`函数

```c
/*
 * When on migration a sched_entity joins/leaves the PELT hierarchy, we need to
 * propagate its contribution. The key to this propagation is the invariant
 * that for each group:
 *
 *   ge->avg == grq->avg						(1)
 *
 * _IFF_ we look at the pure running and runnable sums. Because they
 * represent the very same entity, just at different points in the hierarchy.
 *
 * Per the above update_tg_cfs_util() and update_tg_cfs_runnable() are trivial
 * and simply copies the running/runnable sum over (but still wrong, because
 * the group entity and group rq do not have their PELT windows aligned).
 *
 * However, update_tg_cfs_load() is more complex. So we have:
 *
 *   ge->avg.load_avg = ge->load.weight * ge->avg.runnable_avg		(2)
 *
 * And since, like util, the runnable part should be directly transferable,
 * the following would _appear_ to be the straight forward approach:
 *
 *   grq->avg.load_avg = grq->load.weight * grq->avg.runnable_avg	(3)
 *
 * And per (1) we have:
 *
 *   ge->avg.runnable_avg == grq->avg.runnable_avg
 *
 * Which gives:
 *
 *                      ge->load.weight * grq->avg.load_avg
 *   ge->avg.load_avg = -----------------------------------		(4)
 *                               grq->load.weight
 *
 * Except that is wrong!
 *
 * Because while for entities historical weight is not important and we
 * really only care about our future and therefore can consider a pure
 * runnable sum, runqueues can NOT do this.
 *
 * We specifically want runqueues to have a load_avg that includes
 * historical weights. Those represent the blocked load, the load we expect
 * to (shortly) return to us. This only works by keeping the weights as
 * integral part of the sum. We therefore cannot decompose as per (3).
 *
 * Another reason this doesn't work is that runnable isn't a 0-sum entity.
 * Imagine a rq with 2 tasks that each are runnable 2/3 of the time. Then the
 * rq itself is runnable anywhere between 2/3 and 1 depending on how the
 * runnable section of these tasks overlap (or not). If they were to perfectly
 * align the rq as a whole would be runnable 2/3 of the time. If however we
 * always have at least 1 runnable task, the rq as a whole is always runnable.
 *
 * So we'll have to approximate.. :/
 *
 * Given the constraint:
 *
 *   ge->avg.running_sum <= ge->avg.runnable_sum <= LOAD_AVG_MAX
 *
 * We can construct a rule that adds runnable to a rq by assuming minimal
 * overlap.
 *
 * On removal, we'll assume each task is equally runnable; which yields:
 *
 *   grq->avg.runnable_sum = grq->avg.load_sum / grq->load.weight
 *
 * XXX: only do this for the part of runnable > running ?
 *
 */
static inline void
update_tg_cfs_load(struct cfs_rq *cfs_rq, struct sched_entity *se, struct cfs_rq *gcfs_rq)
{
	long delta_avg, running_sum, runnable_sum = gcfs_rq->prop_runnable_sum;
	unsigned long load_avg;
	u64 load_sum = 0;
	s64 delta_sum;
	u32 divider;

	if (!runnable_sum)
		return;

	gcfs_rq->prop_runnable_sum = 0;

	/*
	 * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
	 * See ___update_load_avg() for details.
	 */
	divider = get_pelt_divider(&cfs_rq->avg);

	if (runnable_sum >= 0) {
		/*
		 * Add runnable; clip at LOAD_AVG_MAX. Reflects that until
		 * the CPU is saturated running == runnable.
		 */
		runnable_sum += se->avg.load_sum;
		runnable_sum = min_t(long, runnable_sum, divider);
	} else {
		/*
		 * Estimate the new unweighted runnable_sum of the gcfs_rq by
		 * assuming all tasks are equally runnable.
		 */
		if (scale_load_down(gcfs_rq->load.weight)) {
			load_sum = div_u64(gcfs_rq->avg.load_sum,
				scale_load_down(gcfs_rq->load.weight));
		}

		/* But make sure to not inflate se's runnable */
		runnable_sum = min(se->avg.load_sum, load_sum);
	}

	/*
	 * runnable_sum can't be lower than running_sum
	 * Rescale running sum to be in the same range as runnable sum
	 * running_sum is in [0 : LOAD_AVG_MAX <<  SCHED_CAPACITY_SHIFT]
	 * runnable_sum is in [0 : LOAD_AVG_MAX]
	 */
	running_sum = se->avg.util_sum >> SCHED_CAPACITY_SHIFT;
	runnable_sum = max(runnable_sum, running_sum);

	load_sum = se_weight(se) * runnable_sum;
	load_avg = div_u64(load_sum, divider);

	delta_avg = load_avg - se->avg.load_avg;
	if (!delta_avg)
		return;

	delta_sum = load_sum - (s64)se_weight(se) * se->avg.load_sum;

	se->avg.load_sum = runnable_sum;
	se->avg.load_avg = load_avg;
	add_positive(&cfs_rq->avg.load_avg, delta_avg);
	add_positive(&cfs_rq->avg.load_sum, delta_sum);
	/* See update_cfs_rq_load_avg() */
	cfs_rq->avg.load_sum = max_t(u32, cfs_rq->avg.load_sum,
					  cfs_rq->avg.load_avg * PELT_MIN_DIVIDER);
}
```

在`6.1.116`版本的内核中，这个函数前的注释其实是在`update_tg_cfs_util`函数之前，在阅读这些代码的时候发现这些注释有助于理解`update_tg_cfs_load`函数，于是就把注释的内容放到这个函数之前，在记录函数的流程之前先简单记录一下这些注释中的内容。当有实体离开或者进入任务组的CFS运行队列之中时，会导致CFS运行队列的三个指标发生变化，这个时候需要向调度层级的上级传递这些变化，传递过程中最重要的是维持任务组之中CFS运行队列的`avg`相关指标与任务组的`avg`相关指标相同，作者提到这个仅适用于`util`、`runnable`指标但不适用于`load`指标。注释中给出了不合适`load_avg`指标计算的理由，即如果对于`load`指标也保持这个相等关系则意味着`ge->avg.load_avg == grp->avg.load_avg`，结合注释中等式2、3可以通过简单的除法计算得到`ge->avg.load_avg`的值，这种计算方式忽略了两个关键点：1.对于实体在进行`load_avg`计算过程中更关注未来会占用多少时间而不关注权重，但对于CFS运行队列来讲更关注权重并且还会把权重当作计算`load_avg`的一部分，关注点不同无法简单的数值相等关系表达，2.CFS运行队列的`runnable`指标计算比较复杂，无法简单用一个相同关系来表示二者之间的关系（例如若一个CFS运行队列之中有两个任务并且这两个任务时间在其时间的$\frac{2}{3}$处于`runnable`状态，那么无论两个任务处于`runnable`状态的时间如何重叠，CFS运行队列在时间的($\frac{2}{3}, 1$)这部分的某个位置处于`runnable`状态）。针对不适用`load_avg`指标的问题，作者假设CFS运行队列中任务的`runnable`时间尽量不重叠，在有任务离开CFS运行队列之后假设所有CFS运行队列之中所有的任务处于`runnable`状态的时间都相同，这样得到了注释之中最后一个等式，这个等式会在代码之中有所体现。

这个函数更新任务组以及任务组所在CFS运行队列的`util`指标，三个参数的含义在`propagate_entity_load_avg`函数的流程之中已经说明。这个函数的计算流程都涉及到`runnable_sum`变量的值，这个变量的值为任务组之中`runnable_sum`指标的变动，这个指标意味着任务组中有任务离开、进入了CFS运行队列之中。`runnable_sum`的值可能会有小于0的情况，在小于0的情况与大于0的情况中计算逻辑完全不同，记录两种情况的计算逻辑之前需要明确CFS运行队列以及实体的`load_sum`计算之间的不同。

注意到`__upate_load_avg_cfs_rq`以及`__update_load_avg_se`这两个函数调用`___update_load_sum`函数时`load`参数的值的含义是不同的，前者是CFS运行队列之中任务的权重之和，后者是任务是否处于可运行状态，结合`___update_load_sum`函数的含义之中`load_sum`的计算可以发现，CFS运行队列的`load_sum`为两次更新时间差的几何级数之和与权重之和的乘积，而实体（例如对应任务组的实体）的`load_sum`在实体处于可运行状态是为两次更新时间差的几何级数之和，这个差别导致了在`runnable_sum`小于0和大于0时不同的处理。

当`runnable_sum`小于0时任务组中有任务离开了CFS运行队列、变成了不可运行的状态了，这个时候计算任务组中每个可运行任务的平均`load_sum`并限制其最小值为任务组的`load_sum`，计算结果保存至`runnable_sum`之中，计算过程中出现除法计算的原因参见第一段和第三段的内容。当`runnable_sum`大于0时将`runnable_sum`更新为任务组目前`load_sum`与任务组之中`runnable_sum`指标变动的和并且限制其最大值，这种情况下代码逻辑很容易理解。无论`runnable_sum`更新前的值是大于0还是小于0，更新之后的值为任务组的`load_sum`的值。

考虑到对于一个实体（实体可以对应一个任务组）来讲，可运行时间之中包含了在CFS运行队列之中等待的时间、在cpu之中运行的时间，所以在接下来的代码之中比较任务组的`load_sum`以及`util_sum`之间的值，保证`load_sum`的值不小于`util_sum`的值。然后计算任务组的`load_avg`的值（`load_sum`与任务组的权重之积除以几何级数之和的最大值，这种计算方式与`___update_load_avg`中`load_avg`的计算方式相同）、更新任务组的`load_sum`的值（仅仅是一个简单的赋值操作），最后更新任务组所在的CFS运行队列的指标。

更新任务组所在CFS运行队列的指标时，先计算任务组的`load_sum`、`load_avg`指标的变化，然后将变化更新到CFS运行队列的对应指标之中去，从这里可以看到CFS运行队列中指标的值为其中实体的对应指标之和。

### `attach_entity_load_avg`函数

```c
/**
 * attach_entity_load_avg - attach this entity to its cfs_rq load avg
 * @cfs_rq: cfs_rq to attach to
 * @se: sched_entity to attach
 *
 * Must call update_cfs_rq_load_avg() before this, since we rely on
 * cfs_rq->avg.last_update_time being current.
 */
static void attach_entity_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	/*
	 * cfs_rq->avg.period_contrib can be used for both cfs_rq and se.
	 * See ___update_load_avg() for details.
	 */
	u32 divider = get_pelt_divider(&cfs_rq->avg);

	/*
	 * When we attach the @se to the @cfs_rq, we must align the decay
	 * window because without that, really weird and wonderful things can
	 * happen.
	 *
	 * XXX illustrate
	 */
	se->avg.last_update_time = cfs_rq->avg.last_update_time;
	se->avg.period_contrib = cfs_rq->avg.period_contrib;

	/*
	 * Hell(o) Nasty stuff.. we need to recompute _sum based on the new
	 * period_contrib. This isn't strictly correct, but since we're
	 * entirely outside of the PELT hierarchy, nobody cares if we truncate
	 * _sum a little.
	 */
	se->avg.util_sum = se->avg.util_avg * divider;

	se->avg.runnable_sum = se->avg.runnable_avg * divider;

	se->avg.load_sum = se->avg.load_avg * divider;
	if (se_weight(se) < se->avg.load_sum)
		se->avg.load_sum = div_u64(se->avg.load_sum, se_weight(se));
	else
		se->avg.load_sum = 1;

	enqueue_load_avg(cfs_rq, se);
	cfs_rq->avg.util_avg += se->avg.util_avg;
	cfs_rq->avg.util_sum += se->avg.util_sum;
	cfs_rq->avg.runnable_avg += se->avg.runnable_avg;
	cfs_rq->avg.runnable_sum += se->avg.runnable_sum;

	add_tg_cfs_propagate(cfs_rq, se->avg.load_sum);

	cfs_rq_util_change(cfs_rq, 0);

	trace_pelt_cfs_tp(cfs_rq);
}
```

将实体放入到CFS运行队列时要更新更新实体以及CFS运行队列的`load`、`util`、`runnable`三个指标，这个函数就是在进行这些指标的更新，从这个函数之中可以看到CFS运行队列指标的值与单个指标值的对应关系。这个函数首先将CFS运行队列和实体的衰减窗口对其，这里的衰减窗口涉及到`last_update_time`以及`period_contrib`两个字段，这两个字段在计算几何级数之和的时候很重要，详见本文档中`accumulate_sum`函数流程记录。

接下来重新计算实体的`sum`相关指标，`util_sum`以及`runnable_sum`直接使用实体对应的`avg`指标乘以这个CFS运行队列的`divider`（`divider`的含义以及使用它背后的原因见`___update_load_avg`、`get_pelt_divider`函数），`load_sum`的计算过程是由`load_sum`计算`load_avg`的反过程（由`load_sum`计算`load_avg`的过程见`___update_load_avg`函数），计算过程中涉到实体权重与中间计算结果（`se->avg.load_sum = se->avg.load_avg * divider`）的除法，这里要考虑二者之间的大小关系，若实体权重小于中间计算结果直接默认二者是相同的。

接下来更新CFS运行队列的三个指标，其中CFS运行队列的`util`、`runnable`指标的`sum`、`avg`直接加上实体对应的值即可，`load`指标的更新在`enqueue_load_avg`函数之中进行，在后边会详细记录这个函数的流程。完成了CFS运行队列的三个指标更新之后，调用`cfs_rq_util_change`更新cpu调频所需数据。

### `enqueue_load_avg`函数

```c
enqueue_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	cfs_rq->avg.load_avg += se->avg.load_avg;
	cfs_rq->avg.load_sum += se_weight(se) * se->avg.load_sum;
}
```

从这个函数之中可以看到实体的`load`指标与其所在CFS运行队列中`load`指标的关系：CFS运行队列的`load_avg`为CFS运行队列之中所有实体的`load_avg`之和，CFS运行队列的`load_sum`指标为CFS运行队列中所有实体的`load_sum`与其权重之积的和。

### `update_tg_load_avg`函数

```c
/**
 * update_tg_load_avg - update the tg's load avg
 * @cfs_rq: the cfs_rq whose avg changed
 *
 * This function 'ensures': tg->load_avg := \Sum tg->cfs_rq[]->avg.load.
 * However, because tg->load_avg is a global value there are performance
 * considerations.
 *
 * In order to avoid having to look at the other cfs_rq's, we use a
 * differential update where we store the last value we propagated. This in
 * turn allows skipping updates if the differential is 'small'.
 *
 * Updating tg's load_avg is necessary before update_cfs_share().
 */
static inline void update_tg_load_avg(struct cfs_rq *cfs_rq)
{
	long delta = cfs_rq->avg.load_avg - cfs_rq->tg_load_avg_contrib;

	/*
	 * No need to update load_avg for root_task_group as it is not used.
	 */
	if (cfs_rq->tg == &root_task_group)
		return;

	if (abs(delta) > cfs_rq->tg_load_avg_contrib / 64) {
		atomic_long_add(delta, &cfs_rq->tg->load_avg);
		cfs_rq->tg_load_avg_contrib = cfs_rq->avg.load_avg;
	}
}
```

这个函数更新任务组的`load_avg`，在记录函数流程之前先说明任务组与CFS运行队列、实体的关系。在`struct task_group`之中包含`se`、`CFS运行队列`两个指针，这两个指针之中的内容为任务组在每个cpu之中的实体、任务组在每个cpu上的CFS运行队列，任务组在每个cpu上的实体用于任务组参与每个cpu上的任务调度过程，任务组在每个cpu上的CFS运行队列用于保存任务组的任务中在每个cpu上可运行的任务。这个函数的`CFS运行队列`参数为任务组在某个cpu上的CFS运行队列，这个函数仅仅是将CFS运行队列的`load_avg`加到任务组的`load_avg`之中，只不过是在保证正确性的情况下使CFS运行队列之中`load_avg`变动积累到比较大的时候再向任务组更新，即只有在本次、上次更新期间CFS运行队列的`load_avg`差距超过上次更新时`load_avg`的$\frac{1}{64}$时才向任务组更新一次，这样减小了更新频率，可以参见这个函数之中的注释帮助理解。

### `detach_entity_load_avg`函数

```c
/**
 * detach_entity_load_avg - detach this entity from its cfs_rq load avg
 * @cfs_rq: cfs_rq to detach from
 * @se: sched_entity to detach
 *
 * Must call update_cfs_rq_load_avg() before this, since we rely on
 * cfs_rq->avg.last_update_time being current.
 */
static void detach_entity_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	dequeue_load_avg(cfs_rq, se);
	sub_positive(&cfs_rq->avg.util_avg, se->avg.util_avg);
	sub_positive(&cfs_rq->avg.util_sum, se->avg.util_sum);
	/* See update_cfs_rq_load_avg() */
	cfs_rq->avg.util_sum = max_t(u32, cfs_rq->avg.util_sum,
					  cfs_rq->avg.util_avg * PELT_MIN_DIVIDER);

	sub_positive(&cfs_rq->avg.runnable_avg, se->avg.runnable_avg);
	sub_positive(&cfs_rq->avg.runnable_sum, se->avg.runnable_sum);
	/* See update_cfs_rq_load_avg() */
	cfs_rq->avg.runnable_sum = max_t(u32, cfs_rq->avg.runnable_sum,
					      cfs_rq->avg.runnable_avg * PELT_MIN_DIVIDER);

	add_tg_cfs_propagate(cfs_rq, -se->avg.load_sum);

	cfs_rq_util_change(cfs_rq, 0);

	trace_pelt_cfs_tp(cfs_rq);
}
```

当一个实体要离开CFS运行队列时需要从CFS运行队列的指标之中减去这个实体贡献的部分，这个函数就是在执行这样的操作。首先调用`dequeue_load_avg`函数从CFS运行队列的`load_sum`、`load_avg`减去实体的贡献，后边详细记录此函数的内容；然后从CFS运行队列的`util`、`runnable`指标之中减去实体的贡献，`util_sum`以及`avg_sum`要保证最小值，这背后的原因在`update_cfs_rq_load_avg`函数之中有说明；最后从CFS运行队列需要向调度层级中的上级传递的`load_sum`指标值之中减去这个实体的`load_sum`指标的值、调用`cfs_rq_util_change`更新cpu调频所需数据。

### `dequeue_load_avg`函数

```c
static inline void
dequeue_load_avg(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	sub_positive(&cfs_rq->avg.load_avg, se->avg.load_avg);
	sub_positive(&cfs_rq->avg.load_sum, se_weight(se) * se->avg.load_sum);
	/* See update_cfs_rq_load_avg() */
	cfs_rq->avg.load_sum = max_t(u32, cfs_rq->avg.load_sum,
					  cfs_rq->avg.load_avg * PELT_MIN_DIVIDER);
}
```

这个函数的计算与`enqueue_load_avg`函数相反，从CFS运行队列的`load_sum`、`load_avg`之中减去实体对应的指标，最后保证CFS运行队列的`load_sum`指标的最小值。

### `se_update_runnable`函数

```c
static inline void se_update_runnable(struct sched_entity *se)
{
	if (!entity_is_task(se))
		se->runnable_weight = se->my_q->h_nr_running;
}
```

当实体对应一个任务组时，即实体为任务组在某个cpu上的调度实体，将实体的`runnable_weight`设置为任务组在同样cpu上CFS运行队列中可运行任务的数量统计。这里的`se`与`se->my_q`为任务组在同一个cpu上的调度实体以及CFS运行队列，任务组在cpu上的调度实体以及CFS运行队列及其含义见`update_tg_load_avg`函数之中对`struct task_group`之中`se`、`cfs_rq`两个指针的含义的解释。

### `update_cfs_group`函数

```c
static void update_cfs_group(struct sched_entity *se)
{
	struct cfs_rq *gcfs_rq = group_cfs_rq(se);
	long shares;

	if (!gcfs_rq)
		return;

	if (throttled_hierarchy(gcfs_rq))
		return;

#ifndef CONFIG_SMP
	shares = READ_ONCE(gcfs_rq->tg->shares);

	if (likely(se->load.weight == shares))
		return;
#else
	shares   = calc_group_shares(gcfs_rq);
#endif

	reweight_entity(cfs_rq_of(se), se, shares);
}
```

这个函数用于更新任务组在某个cpu上实体的权重，在记录函数流程的时候仅考虑定义了`CONFIG_SMP`宏情况下执行的代码。这个函数在确保实体对应一个任务组时，即实体为任务组在某个cpu上的调度实体时，才会继续执行真正的计算逻辑。真正的计算逻辑封装在了两个函数之中：`calc_group_shares`函数重新计算实体的权重、`reweight_entity`更新实体权重，这两步操作关联到了许多内容因此将具体的操作放到了两个函数之中，接下来详细记录这两个函数的内容。

### `calc_group_shares`函数

```c
/*
 * All this does is approximate the hierarchical proportion which includes that
 * global sum we all love to hate.
 *
 * That is, the weight of a group entity, is the proportional share of the
 * group weight based on the group runqueue weights. That is:
 *
 *                     tg->weight * grq->load.weight
 *   ge->load.weight = -----------------------------               (1)
 *                       \Sum grq->load.weight
 *
 * Now, because computing that sum is prohibitively expensive to compute (been
 * there, done that) we approximate it with this average stuff. The average
 * moves slower and therefore the approximation is cheaper and more stable.
 *
 * So instead of the above, we substitute:
 *
 *   grq->load.weight -> grq->avg.load_avg                         (2)
 *
 * which yields the following:
 *
 *                     tg->weight * grq->avg.load_avg
 *   ge->load.weight = ------------------------------              (3)
 *                             tg->load_avg
 *
 * Where: tg->load_avg ~= \Sum grq->avg.load_avg
 *
 * That is shares_avg, and it is right (given the approximation (2)).
 *
 * The problem with it is that because the average is slow -- it was designed
 * to be exactly that of course -- this leads to transients in boundary
 * conditions. In specific, the case where the group was idle and we start the
 * one task. It takes time for our CPU's grq->avg.load_avg to build up,
 * yielding bad latency etc..
 *
 * Now, in that special case (1) reduces to:
 *
 *                     tg->weight * grq->load.weight
 *   ge->load.weight = ----------------------------- = tg->weight   (4)
 *                         grp->load.weight
 *
 * That is, the sum collapses because all other CPUs are idle; the UP scenario.
 *
 * So what we do is modify our approximation (3) to approach (4) in the (near)
 * UP case, like:
 *
 *   ge->load.weight =
 *
 *              tg->weight * grq->load.weight
 *     ---------------------------------------------------         (5)
 *     tg->load_avg - grq->avg.load_avg + grq->load.weight
 *
 * But because grq->load.weight can drop to 0, resulting in a divide by zero,
 * we need to use grq->avg.load_avg as its lower bound, which then gives:
 *
 *
 *                     tg->weight * grq->load.weight
 *   ge->load.weight = -----------------------------		   (6)
 *                             tg_load_avg'
 *
 * Where:
 *
 *   tg_load_avg' = tg->load_avg - grq->avg.load_avg +
 *                  max(grq->load.weight, grq->avg.load_avg)
 *
 * And that is shares_weight and is icky. In the (near) UP case it approaches
 * (4) while in the normal case it approaches (3). It consistently
 * overestimates the ge->load.weight and therefore:
 *
 *   \Sum ge->load.weight >= tg->weight
 *
 * hence icky!
 */
static long calc_group_shares(struct cfs_rq *cfs_rq)
{
	long tg_weight, tg_shares, load, shares;
	struct task_group *tg = cfs_rq->tg;

	tg_shares = READ_ONCE(tg->shares);

	load = max(scale_load_down(cfs_rq->load.weight), cfs_rq->avg.load_avg);

	tg_weight = atomic_long_read(&tg->load_avg);

	/* Ensure tg_weight >= load */
	tg_weight -= cfs_rq->tg_load_avg_contrib;
	tg_weight += load;

	shares = (tg_shares * load);
	if (tg_weight)
		shares /= tg_weight;

	/*
	 * MIN_SHARES has to be unscaled here to support per-CPU partitioning
	 * of a group with small tg->shares value. It is a floor value which is
	 * assigned as a minimum load.weight to the sched_entity representing
	 * the group on a CPU.
	 *
	 * E.g. on 64-bit for a group with tg->shares of scale_load(15)=15*1024
	 * on an 8-core system with 8 tasks each runnable on one CPU shares has
	 * to be 15*1024*1/8=1920 instead of scale_load(MIN_SHARES)=2*1024. In
	 * case no task is runnable on a CPU MIN_SHARES=2 should be returned
	 * instead of 0.
	 */
	return clamp_t(long, shares, MIN_SHARES, tg_shares);
}
```

在记录这个函数的执行逻辑之前先记录注释中的内容，这些内容有助于理解代码中的计算逻辑，注释中`grq->load.weight`、`grp->avg.load_avg`之中`grp`指的是任务组在某个cpu上的CFS运行队列。任务组在某个cpu上的实体的权重计算公式为注释中如等式(1)所示，但是等式(1)之中存在计算比较慢求和计算，使用`load_avg`替换等式(1)之中的实体权重`weight`得到等式(3)。使用了`load_avg`提到`weight`带来的弊端是`load_avg`的变化是比较缓慢的，这在一些边界情况中会引来误差。注释中还提到了一种特殊情况，在任务组之中只有一个任务的情况下只有一个cpu需要执行任务而其他的cpu都是空闲状态，此时可以对等式(1)进行化简得到等式(4)，此时结果永远是任务组权重这一个恒定的值，这种情况类似于在单核cpu之中运行。值得注意的是等式(4)为等式(1)在特殊情况下推导出来的，但想要使用等式(3)替代等式(1)，这就意味着需要使用等式(3)来近似等式(4)的结果，于是有了在等式(3)中除数之中使用任务组在某个cpu上CFS运行队列的权重替代其`load_avg`的想法，形成了等式(5)。在某些情况下等式(5)中会出现除数为0的情况，例如任务组之中只有一个任务等，使用等式(6)来替代等式(5)修复这个问题，等式(6)就是代码中实现的计算逻辑。

结合注释中的内容理解这个函数的代码实现比较容易，弄清楚代码中几个变量的含义就可以明确整个计算流程中每一步在做什么，其中代码之中`tg_shares`为任务组的权重，对应等式(6)之中`tg->weight`项；`load`对应等式(6)之中的`max(grq->load.weight, grq->avg.load_avg)`的结果；`tg_weight`最终的结果对应等式(6)之中`tg_load_avg'`；进行除法计算之前的`shares`对应等式(6)中的乘法计算结果，注意这里进行乘法的时候使用的是`load`而非等式中要求的CFS运行队列的权重，这是考虑到CFS运行队列的权重可能为0；进行除法之后的`shares`对应等式(6)的计算结果，得到最终的计算结果之后要将结果限制在合理的取值范围之中。

### `account_entity_enqueue`函数

```c
static void
account_entity_enqueue(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	update_load_add(&cfs_rq->load, se->load.weight);
#ifdef CONFIG_SMP
	if (entity_is_task(se)) {
		struct rq *rq = rq_of(cfs_rq);

		account_numa_enqueue(rq, task_of(se));
		list_add(&se->group_node, &rq->cfs_tasks);
	}
#endif
	cfs_rq->nr_running++;
	if (se_is_idle(se))
		cfs_rq->idle_nr_running++;
}
```

这个函数执行调度实体进入CFS运行队列时CFS运行队列之中的统计字段更新，首先使用`update_load_weight`函数将实体的权重加入到CFS运行队列的权重之中，随后递增CFS运行队列之中可运行的任务统计，最后若任务为一个空闲任务更新CFS运行队列之中可运行的空闲任务统计。上述更新过程中若发现实体对应一个任务，则将其加入到运行队列的`cfs_tasks`队列之中，这个队列管理运行队列之中有`CFS`调度类管理的任务。

### `__enqueue_entity`函数

```c
/*
 * Enqueue an entity into the rb-tree:
 */
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	rb_add_cached(&se->run_node, &cfs_rq->tasks_timeline, __entity_less);
}
```

最终将实体放入到CFS运行队列是在这个函数之中进行的，这个函数将实体放入到CFS运行队列内部维护的一颗红黑树之中，红黑树之中实体比较使用了`__entity_less`函数，这个函数比较的是两个实体的虚拟运行时间。

### `list_add_leaf_cfs_rq`函数

```c
static inline bool list_add_leaf_cfs_rq(struct cfs_rq *cfs_rq)
{
	struct rq *rq = rq_of(cfs_rq);
	int cpu = cpu_of(rq);

	if (cfs_rq->on_list)
		return rq->tmp_alone_branch == &rq->leaf_cfs_rq_list;

	cfs_rq->on_list = 1;

	/*
	 * Ensure we either appear before our parent (if already
	 * enqueued) or force our parent to appear after us when it is
	 * enqueued. The fact that we always enqueue bottom-up
	 * reduces this to two cases and a special case for the root
	 * cfs_rq. Furthermore, it also means that we will always reset
	 * tmp_alone_branch either when the branch is connected
	 * to a tree or when we reach the top of the tree
	 */
	if (cfs_rq->tg->parent &&
	    cfs_rq->tg->parent->cfs_rq[cpu]->on_list) {
		/*
		 * If parent is already on the list, we add the child
		 * just before. Thanks to circular linked property of
		 * the list, this means to put the child at the tail
		 * of the list that starts by parent.
		 */
		list_add_tail_rcu(&cfs_rq->leaf_cfs_rq_list,
			&(cfs_rq->tg->parent->cfs_rq[cpu]->leaf_cfs_rq_list));
		/*
		 * The branch is now connected to its tree so we can
		 * reset tmp_alone_branch to the beginning of the
		 * list.
		 */
		rq->tmp_alone_branch = &rq->leaf_cfs_rq_list;
		return true;
	}

	if (!cfs_rq->tg->parent) {
		/*
		 * cfs rq without parent should be put
		 * at the tail of the list.
		 */
		list_add_tail_rcu(&cfs_rq->leaf_cfs_rq_list,
			&rq->leaf_cfs_rq_list);
		/*
		 * We have reach the top of a tree so we can reset
		 * tmp_alone_branch to the beginning of the list.
		 */
		rq->tmp_alone_branch = &rq->leaf_cfs_rq_list;
		return true;
	}

	/*
	 * The parent has not already been added so we want to
	 * make sure that it will be put after us.
	 * tmp_alone_branch points to the begin of the branch
	 * where we will add parent.
	 */
	list_add_rcu(&cfs_rq->leaf_cfs_rq_list, rq->tmp_alone_branch);
	/*
	 * update tmp_alone_branch to points to the new begin
	 * of the branch
	 */
	rq->tmp_alone_branch = &cfs_rq->leaf_cfs_rq_list;
	return false;
}
```

`leaf cfs_rq`指的是调度层次之中一类特殊的CFS运行队列，这类CFS运行队列之中保存的实体对应的是任务，这个函数将一个cpu之中的`leaf cfs_rq`都放到一个cpu的运行队列之中的列表之中，若之后记录`CFS`的其他函数使用到这个列表，再详细记录这个列表的作用以及相关机制，这里仅仅记录这个函数的流程。

`rq`为CFS运行队列所在cpu的任务队列，这个队列是Linux内核之中调度框架之中的任务队列，不是CFS调度类使用的任务队列，`rq`中的`leaf_cfs_rq_list`就是保存`leaf cfs_rq`队列的头节点。结合注释可以看到，`tmp_alone_branch`主要是在指向新的`cfs_rq`插入`rq`的`leaf_cfs_rq_list`时的位置，在一些情况之中需要借助它进行判断：若`cfs_rq`已经在队列之中则返回调度层级中的父队列是否已经在`rq`的`leaf_cfs_rq_list`之中；若这个`cfs_rq`有父队列并且父队列已经在`rq`的`leaf_cfs_rq_list`之中，直接将`cfs_rq`加入到父队列之中`leaf_cfs_rq_list`的队尾，并且将`rq`的`tmp_alone_branch`指向`rq`的`leaf_cfs_rq_list`表示父队列已经在`rq`的`leaf_cfs_rq_list`之中；若父队列不存在，这意味着`cfs_rq`位于调度层级之中最顶层，则先将`cfs_rq`加入到`rq`的`leaf_cfs_rq_list`之中；若父队列还没有放到`rq`的`leaf_cfs_rq_list`之中，则将`cfs_rq`加入到`tmp_alone_branch`指向的列表之中位置之后并且让`tmp_alone_branch`指向新加入的`cfs_rq`，这样当`cfs_rq`的父队列需要入队的时候可以快速找到自己所在的位置。

这里结合一个例子说明`rq`的`leaf_cfs_rq_list`是怎样的，例如任务组层级为：

```c
Root CFS RQ
│
├── Parent CFS RQ (Group A)
│   ├── Child1 CFS RQ
│   └── Child2 CFS RQ
│
└── Sibling CFS RQ (Group B)
    ├── Child3 CFS RQ
    └── Child4 CFS RQ
```

层级之中`Child CFS RQ`为`leaf cfs_rq`，那么`rq`中`leaf_cfs_rq_list`队列的内容为：

```
rq->leaf_cfs_rq_list = [
  Root CFS RQ,
  Parent CFS RQ (Group A),
  Child1 CFS RQ,
  Child2 CFS RQ,
  Sibling CFS RQ (Group B),
  Child3 CFS RQ,
  Child4 CFS RQ
]
```

### `hrtick_update`函数

```c
/*
 * called from enqueue/dequeue and updates the hrtick when the
 * current task is from our class and nr_running is low enough
 * to matter.
 */
static void hrtick_update(struct rq *rq)
{
	struct task_struct *curr = rq->curr;

	if (!hrtick_enabled_fair(rq) || curr->sched_class != &fair_sched_class)
		return;

	if (cfs_rq_of(&curr->se)->nr_running < sched_nr_latency)
		hrtick_start_fair(rq, curr);
}
```

这个函数只有在启用了高精度定时器、参数`rq`中正在运行任务使用的调度类为CFS时才会起作用，当正在运行任务所在的CFS运行队列中可运行任务数量小于`sched_nr_latency`时执行`hrtick_start_fair`函数并rq之中传入正在运行的任务，`hrtick_start_fair`函数会设置一个新的高精度定时器，这个定时器的触发时间为正在运行任务时间片耗尽的时间，接下来会详细记录这个函数的流程。从代码之中可以看到，不是所有的情况下都会执行`hrtick_start_fair`函数设置新的高精度定时器，这里使用了`sched_nr_latency`当作阈值，`sched_nr_latency`的值由$\frac{sysctl\_sched\_latency}{sysctl\_sched\_min\_granularity}$计算的来，接下来详细记录`sysctl_sched_latency`以及`sysct_sched_min_granularity`，通过这两个值来理解`sysctl_nr_latency`的含义。

`sysctl_sched_latency`为内核认为比较理想的调度延迟，调度延迟指的是rq之中每个任务至少运行一次的时间，但是如果内核之中任务数量过大可能会导致每个任务得到的运行时间很短、任务调度过程频繁发生，任务调度占用的时间会比较高。这个时候约定一个任务最少的运行时间`sysctl_sched_min_granularity`，这个运行时间不考虑任务权重，然后由这个时间乘以运行队列中的可运行任务得到调度延迟。结合这些信息可以得到`sched_nr_latency`为运行队列中任务数量是多还是少的一个阈值，超过这个值基于`sysctl_sched_min_granularity`计算调度延迟，没有超过这个值则使用内核认为比较理想的调度延迟。

### `hrtick_start_fair`函数

```c
static void hrtick_start_fair(struct rq *rq, struct task_struct *p)
{
	struct sched_entity *se = &p->se;
	struct cfs_rq *cfs_rq = cfs_rq_of(se);

	SCHED_WARN_ON(task_rq(p) != rq);

	if (rq->cfs.h_nr_running > 1) {
		u64 slice = sched_slice(cfs_rq, se);
		u64 ran = se->sum_exec_runtime - se->prev_sum_exec_runtime;
		s64 delta = slice - ran;

		if (delta < 0) {
			if (task_current(rq, p))
				resched_curr(rq);
			return;
		}
		hrtick_start(rq, delta);
	}
}
```

当这个函数从`hrtick_udpate`函数之中调用时，`p`为运行队列之中正在运行的任务，`p`所在的运行队列由函数参数`运行队列`给出。这个函数调用`sched_slice`函数计算任务`p`获得到的时间片、计算任务`p`的时间片中已经使用的时间，这样可以推断出`delta`的含义为任务`p`的时间片中还未使用的时间。若`delta`小于0则认为任务`p`的时间统计需要更新，调用`resched_curr`函数更新任务的时间统计然后返回；若`delta`大于0则调用`hrtick_start`设置一个新的高精度定时器，定时器触发的时间正好是任务`p`时间片耗尽的时候。上述过程中提到的`resched_curr`函数详细流程在`task_wake_up.md`之中记录、`sched_slice`函数详细流程在`task_fork_cfs.md`之中记录、`hrtick_start`函数详细流程在`schedule.md`之中记录。
