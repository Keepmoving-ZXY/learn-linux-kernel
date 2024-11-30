本文档记录`enqueue_task_fair`函数的流程，这个函数将调度实体`se`插入到cfs_rq的红黑树之中，这个过程中还会更新一些统计字段。

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

这里先关注两个循环（`for_each_sched_entity`）中的内容，这两个循环涉及到内核的任务组调度功能，关于这个功能的详细介绍见[task_group](https://blogs.oracle.com/linux/post/cfs-group-scheduling)，从介绍中可以看到在任务组调度功能之中，调度实体被划分到了多个层级，有的层次中调度实体对应一个任务组、有的层级中的调度实体对应一个任务，这两个`for`循环都是对于任务`p`对应的调度实体以及上级每个层级与之关联的调度实体而言的。第一个`for`循环用于将`p`对应的实体以及每个层级中与这个实体关联的实体放入到rq之中，第二个`for`循环用于更新平均负载、任务组的可运行任务相关统计字段、重新计算任务组实体的权重。两个循环之后调用`add_nr_running`更新cfs_rq中正在运行任务的计数并进行超载检查，最后调用`hrtick_update`更新高精度定时器。

这里仅仅是简单整理一下这个函数的整体流程，后边会具体记录这其中许多函数的流程，被忽略的函数除外。这个函数中还隐藏着一个有意思的处理方式，这种涉及到了两个字段的更新（`h_nr_running`、`idle_h_nr_running`），这两个字段为cfs_rq的统计字段，代表了在当前的调度层级以及下层之中正在运行的实体、优先级极低（任务的调度策略被指定为了`SCHED_IDLE`）的任务。先以`h_nr_running`为例，当任务`p`对应的实体以及调度层级中与之关联的实体都放入到cfs_rq之中时，每个层级之中新增的正在运行的实体为2，但是在第一个`for`循环之中只为每一层的cfs_rq中这个字段递增，这个看起来很奇怪，因为这里需要2次递增，直到看到第二个`for`循环之中的代码才知道第二次递增在这个循环之中。`idle_h_nr_running`的数值更新逻辑类似，也是分到了两个`for`循环之中完成数值递增，只不过在某个层级调度实体对应的任务组为`SCHED_IDLE`组时才开始更新。

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

这个函数将调度实体`se`放入到cfs_rq之中，在放入到cfs_rq前后需要更新许多任务相关的统计内容。`update_curr`函数的流程已经在`task_fork_cfs.md`中有记录，这个函数主要是更新正在运行任务的时间统计，在这个函数前后有两个`if`判断，这两个`if`判断成立时执行的操作都是将实体`se`的相对虚拟运行时间调整成虚拟运行时间，但是一个在`update_curr`函数之前、另外一个在这个函数之后，这背后的原因是：当第一个`if`判断成立时`se`对应的任务为cfs_rq之中正在运行的任务，更新这个任务的虚拟运行时间可能会影响`update_curr`函数中cfs_rq中虚拟运行时间基准的计算（`min_vruntime`），所以需要先更新任务的虚拟运行时间然后调用`update_curr`；如果`se`对应的任务不是cfs_rq中正在运行的任务，这个任务目前为止还没有加入到cfs_rq中，但是需要cfs_rq之中的虚拟运行时间基准来计算虚拟运行时间，所以需要在`update_curr`更新cfs_rq的虚拟运行时间基准之后执行。两个`if`判断之中还会涉及到`renorm`是否为`True`，它为`True`意味着`se`对应的任务不是被唤醒的任务或者从其他的rq中迁移过来的任务。

从`update_load_avg`函数到`account_entity_queueue`函数更新任务以及cfs_rq的平均负载、更新任务组对的`runnable_weight`、重新计算任务组的权重、更新cfs_rq中运行任务的权重之和等操作，这些更新都封装到了不同的函数之中国，后边会对这些函数的流程进行记录。接下来使用`__enqueue_entity`函数将实体加入到cfs_rq之中，添加任务在rq中的标记(`on_rq`更新为1)，最后将cfs_rq放入到leaf_cfs_rq_list之中，在`list_add_leaf_cfs_rq`函数中解释这个list的作用。

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

这个函数用于更新任务和cfs_rq的平均负载，负载可以理解为时间占用，在后边的流程中可以看到时间占用大致分为在rq中的时间以及在cpu中运行的时间。这个函数首先使用`cfs_rq_clock_pelt`函数获取现在的时间，这个函数返回的时间不是通用的时间，而是计算任务负载专用的时间，这个时间是经过真实的时间根据cpu的`capacity`以及`frequency`进行缩放之后得到，考虑了一颗cpu中不同核心计算能力的差异，使得在不同的核心中使用相同的标准计算负载。这个时间的更新在`update_rq_clock_pelt`函数之中，这个函数的流程记录在`task_wake_up.md`文件中可以找到。

`se->avg.last_update_time`记录了上一次负载统计更新的时间，不为0时表示这个实体对应的任务不是从其他rq之中迁移过来的，此时若没有要求跳过负载统计更新调用`___update_load_avg_se`更新实体的平均负载统计。这个函数里涉及到了计算任务负载的指数加权移动平均，在后边会详细记录这个函数中的流程。更新完实体`se`的负载统计之后，调用`update_cfs_rq_load_avg`函数更新当前cfs_rq的负载统计。考虑到组调度特性，调度实体`se`位于调度层级中的某一层次，调度实体负载统计的变化也需要向上级传递，更新上级中的负载统计，这个过程由`propagate_entity_load_avg`函数完成。

当`se->avg.last_update_time`为0表示这个任务是从其他的rq之中迁移过来的，当`flags & DO_ATTACH`不为0时表示这个函数是在任务放入到cfs_rq过程中调用的，调用`attach_entity_load_avg`函数将任务的平均负载添加到cfs_rq的平均负载统计之中，调用`update_tg_load_avg`函数更新任务组的平均负载统计。当`flags & DO_DETACH`不为0时表示这个任务将要迁移到其他的rq之中，这个时候要从cfs_rq的平均负载统计中减去此任务的贡献，然后更新任务组的平均负载统计。如果在之前的更新过程执行过指数衰减加权平均计算，调用`cfs_rq_util_change`更新cpu调频所需数据、更新线程组的平均负载。

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

这个函数用于更新实体`se`的平均负载，首先调用`___update_load_sum`函数计算实体`se`的`load_sum`、`runnable_sum`、`util_sum`这三个指标的值，这三个指标之中load_sum基于可运行时间计算（包含在cfs_rq的时间、cpu上执行的时间），`util_sum`基于在cpu上的执行时间计算，`runnable_sum`同样基于任务的可运行时间计算但是计算细节与`load_sum`的计算有所不同，这三个sum指标为关于实体两次更新三个指标时间差的几何级数之和，具体计算方式在`___update_load_sum`函数流程记录之中说明。更新完``{load、runnable、util}_sum`之后调用`__update_load_avg`计算`{load,runnable, util}_avg`三个指标，这三个指标为三个对应几何级数之后的均值，具体计算方式在这个函数的流程记录中说明。最后调用`cfs_se_util_change`函数标记`util_avg`已经更新过。

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

这个函数计算`load_sum`、`runnable_sum`、`util_sum`三个指标，这里先记录这三个指标的计算过程，然后解释这三个指标的含义。这个函数将时间按照1024 ns为一个周期划分，`delta`为实体两个统计更新之间的时间差，这个时间差可能会出现在多个周期之中，第一个周期为距离现在最远的周期，最后一个周期为`now`所在的周期，在第一个周期的时间部分并不一定会占用整个周期，最后一个周期的时间也并不一定会占用整个周期。这里假设时间差出现在p个完整的周期，加上两个特殊的周期，一共出现了在p+2个周期之中，将在每个周期中的时间设置为`u_i`，其中有p个`u_i`为1024而额外两个`u_i`的值小于1024，`u_0`为在第一个周期的时间占用，`u_{p+1}`为在最后一个周期的时间占用。接下来将这些`p+2`个`u_i`当作`p+2`个几何级数的系数进行求和，具体计算方式为`u_0 + u_1*y + u_2*y^2 + u_3*y^3 + ...`，y的取值满足`y^32`为0.5。由于`y`的取值小于1，使用几何级数求和意味着在结果中距离现在越近的周期对应的u_i在结果中的比例越高，距离现在越远的周期对应的u_i在结果中的比例越低，例如`u_0`的系数为1而`u_31`的系数为0.5。

在`task_fork_cfs.md`之中记录`__calc_delta`函数的时候提到过内核无法直接进行浮点数计算，因此这里的几何级数求和也无法直接进行计算，内核解决这个问题的方法是选择y使得`y^32`为0.5、在内核源码中保存{`y`,`y^2`,...,`y^31`}的数值，这个时候若要计算`u_i * y^p`，需要考虑`p`是否大于32：如果`p < 32`，则`y^p`可以直接获取；如果 `p >= 32`，则可以使用如下方法：假设 `p = m * 32 + n`，那么`y^p = (y^{32})^m * y^n` 由于 `y^{32} = 1/2`，所以`y^p = (1/2)^m * y^n`（因为`n < 32`所以`y^n` 的值可以直接获取）。上述过程只能计算出来单个`y^n`的值，在几何级数求和公式中还存在多个不同技术结果的求和，如下所示：

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
接下来关注这个函数的流程，这个函数进行一些判断最后调用`accumulate_sum`进行几何级数求和计算。这个函数首先计算两次更新之间的时间差存入`delta`之中，若`delta`小于0或者`delta`跨越的时间不足一个周期直接退出函数，其他情况下继续执行。接下来的代码涉及到`load`、`runnable`、`running`三个参数：当实体`se`在cfs_rq之中时`load`为1；当实体对应一个任务并且实体在cfs_rq之中时`runnable`为1，当实体对饮一个线程组并且实习在cfs_rq之中时`runnable`为任务组中可运行的任务数量；当实体对应的任务正在运行时`running`为1。当实体对应任务时，`load`和`runnable`的含义是一致的，即任务是否处于可运行状态；当实体对应任务组时，`load`对应任务组是否处于可运行状态、`runnable`为任务组中可运行的任务，在接下来的流程之中默认实体既可以对应任务也可以对应任务组，当对应任务与任务组处理方式不同时会有特殊说明。代码注释之中提到一种特殊的情况需要处理，即当`load`为0时意味着实体不在cfs_rq之中，那么在一些情况之中可能会认为实体对应的任务正在运行中，这个时候需要纠正错误的判断。接下来调用`accumulate_sum`进行几何级数求和计算，接下来记录的函数流程就是这个函数的流程，在这个函数之中能真正看到`load_sum`、`runnable_sum`、`util_sum`的计算方式，也会明确这三个指标的含义是什么。

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

`__accumulate_pelt_segments`函数会给出`delta`对应的这段时间的几何级数之和，几何级数之和会存入到`contrib`之中，最后的三个`if`判断中的代码更新实体的三个指标，指标的更新取决于`load`、`runnable`、`running`三个参数是否为0，若为0则跳过指标更新。从这三个指标更新的代码之中可以看到：`load_sum`为实体处于可运行状态（`load`不为0）时占用时间的几何级数之和；当实体对应一个可运行任务时`runnable_sum`与`load_sum`含义一致，对应`runnable`不为0，只不过把它当作定点数然后缩放成整数；当实体对应可运行的任务组时，同样对应`runnable`不为0，``runnable_sum`为任务组中为任务组中可运行任务数量、任务组处于可运行状态时占用时间的几何级数之和两个因素的乘积，将乘积结果当作定点数随后转换成整数；`util_sum`为任务处于实体处于运行状态时占用时间的几何级数之和。

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
