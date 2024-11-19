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
