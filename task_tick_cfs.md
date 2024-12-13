本文档主要记录`task_tick_fair`函数的流程，这个函数是CFS调度类提供的`task_tick`方法实现，`task_tick`方法在内核周期性定时器、高精度定时器触发的函数中执行，本文档中会详细记录这两种不同调用情况的差异。

### `task_tick_fair`函数

```c
/*
 * scheduler tick hitting a task of our scheduling class.
 *
 * NOTE: This function can be called remotely by the tick offload that
 * goes along full dynticks. Therefore no local assumption can be made
 * and everything must be accessed through the @rq and @curr passed in
 * parameters.
 */
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		entity_tick(cfs_rq, se, queued);
	}

	if (static_branch_unlikely(&sched_numa_balancing))
		task_tick_numa(rq, curr);

	update_misfit_status(curr, rq);
	check_update_overutilized_status(task_rq(curr));

	task_tick_core(rq, curr);
}
```

在Linux调度框架中由定时器来驱动正在运行任务的运行时间更新、任务切换，调度框架规定每个调度类要提供`task_tick`方法的实现。这个函数可能会在`scheduler_tick`函数中调用，也可能在`hrtick`函数中调用，前者是周期性定时器触发的函数，后者是由高精度定时器触发的函数。这个函数的主要流程是从实体`se`所在的层级开始遍历更高层级中的实体，对每个实体以及所属的cfs_rq调用`entity_tick`函数更新各种统计内容，这里的`se`为其所在调度层次之中正在运行的实体，接下来会详细记录`entity_tick`函数的流程。这个函数中还有其他的流程，这些流程不是任务调度的主要流程，直接忽略它们。

### `entity_tick`函数

```c
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
	/*
	 * Update run-time statistics of the 'current'.
	 */
	update_curr(cfs_rq);

	/*
	 * Ensure that runnable average is periodically updated.
	 */
	update_load_avg(cfs_rq, curr, UPDATE_TG);
	update_cfs_group(curr);

#ifdef CONFIG_SCHED_HRTICK
	/*
	 * queued ticks are scheduled to match the slice, so don't bother
	 * validating it and just reschedule.
	 */
	if (queued) {
		resched_curr(rq_of(cfs_rq));
		return;
	}
	/*
	 * don't let the period tick interfere with the hrtick preemption
	 */
	if (!sched_feat(DOUBLE_TICK) &&
			hrtimer_active(&rq_of(cfs_rq)->hrtick_timer))
		return;
#endif

	if (cfs_rq->nr_running > 1)
		check_preempt_tick(cfs_rq, curr);
}
```

这个函数更新cfs_rq以及实体所在任务组的统计字段、确定正在执行的任务是否会被抢占，若确定任务会被抢占则为任务添加`TIF_NEED_RESCHED`标记。这个函数首先更新正在运行任务的时间统计、更新任务以及所在的cfs_rq的平均负载、更新任务组在某个cpu上实体的权重，`update_curr`函数的流程记录见`task_fork_cfs.md`、`update_load_avg`以及`update_cfs_group`两个函数的流程记录见`enqueue_task_fair.md`。接下来的内容涉及到周期性定时器与高精度定时器：若`queued`为1则表示此函数是从高精度定时器触发的`hrtick`函数之中调用，直接调用`resched_curr`设置任务抢占标记然后返回，可以直接进行抢占而不需要检查的原因是在设置高精度定时器时将触发时间设置为了rq中正在执行任务的时间片耗尽的时刻；若`queued`为0则表示此函数是从周期性的定时器触发的函数`scheduler_tick`之中调用的，此时若cfs_rq中有活跃的高精度定时器直接退出，原因会在`hrtick_update`函数之中提到；若cfs_rq没有活跃的高精度定时器，调用`check_preempt_tick`函数确定是否可以抢占当前正在运行的任务。上述流程中提到的`resched_curr`函数的详细流程在`task_wake_up.md`中记录，`hrtick`函数的详细流程在`schedule.md`文件中记录，`hrtick_start_fair`函数详细流程在`enqueue_task_fair.md`文件中记录，`check_preempt_tick`两个函数的详细流程在后边记录，其他的函数忽略。

### `check_preempt_tick`函数

```c
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	unsigned long ideal_runtime, delta_exec;
	struct sched_entity *se;
	s64 delta;

	ideal_runtime = sched_slice(cfs_rq, curr);
	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
	if (delta_exec > ideal_runtime) {
		resched_curr(rq_of(cfs_rq));
		/*
		 * The current task ran long enough, ensure it doesn't get
		 * re-elected due to buddy favours.
		 */
		clear_buddies(cfs_rq, curr);
		return;
	}

	/*
	 * Ensure that a task that missed wakeup preemption by a
	 * narrow margin doesn't have to wait for a full slice.
	 * This also mitigates buddy induced latencies under load.
	 */
	if (delta_exec < sysctl_sched_min_granularity)
		return;

	se = __pick_first_entity(cfs_rq);
	delta = curr->vruntime - se->vruntime;

	if (delta < 0)
		return;

	if (delta > ideal_runtime)
		resched_curr(rq_of(cfs_rq));
}
```

这个函数确定cfs_rq之中正在运行的任务是否可以被抢占，这个函数调用`sched_slice`函数得到正在运行任务获得的时间片，`delta_exec`为任务时间片中已经消耗的时间，当已经消耗的时间大于这个任务的时间片意味着任务的时间片已经耗尽，调用`resched_curr`函数为这个任务设置抢占标志，随后函数退出。当任务的时间片还没有耗尽时，获取cfs_rq之中虚拟运行时间最小的任务，计算两个任务虚拟运行时间的差距，如果差距比正在运行任务的时间片还大则直接为正在运行任务设置抢占标记。`resched_curr`函数详细流程记录见`task_wake_up.md`，`sched_slice`函数详细流程记录见`task_fork_cfs.md`，`clear_buddies`函数直接忽略。

上述流程记录中忽略了任务已经消耗的时间与`sysctl_sched_min_granulrity`的比较，当前者小于后者时意味着任务运行时间比较短，此时不需要进行任何处理，这就意味着`sysctl_sched_min_granularity`为任务运行的最短运行时间。当前者大于后者的时候任务可能会被中断，这里正好对应了代码之中这个比较之前的注释内容。注释内容提到因为调度器的原因一个任务可能没有及时得到运行机会时，这个任务不需要等待一个完整的时间片才有机会继续执行，可以理解代码理解这些注释：代码之中判断只要任务的运行时间大于`sysctl_sched_min_granularity`时继续执行后续的流程，在后续的流程中会存在为此任务设置抢占标记，这个时候之前未得到运行机会的任务就有可能抢占正在运行的任务。
