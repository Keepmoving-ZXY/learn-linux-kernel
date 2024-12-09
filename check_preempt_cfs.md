本文档记录`check_preempt_wakeup`函数的流程，这个函数是CFS调度类用于判断新唤醒的任务是否可以抢占正在运行任务。

### `check_preempt_wakeup`函数

```c
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
	struct task_struct *curr = rq->curr;
	struct sched_entity *se = &curr->se, *pse = &p->se;
	struct cfs_rq *cfs_rq = task_cfs_rq(curr);
	int scale = cfs_rq->nr_running >= sched_nr_latency;
	int next_buddy_marked = 0;
	int cse_is_idle, pse_is_idle;

	if (unlikely(se == pse))
		return;

	/*
	 * This is possible from callers such as attach_tasks(), in which we
	 * unconditionally check_preempt_curr() after an enqueue (which may have
	 * lead to a throttle).  This both saves work and prevents false
	 * next-buddy nomination below.
	 */
	if (unlikely(throttled_hierarchy(cfs_rq_of(pse))))
		return;

	if (sched_feat(NEXT_BUDDY) && scale && !(wake_flags & WF_FORK)) {
		set_next_buddy(pse);
		next_buddy_marked = 1;
	}

	/*
	 * We can come here with TIF_NEED_RESCHED already set from new task
	 * wake up path.
	 *
	 * Note: this also catches the edge-case of curr being in a throttled
	 * group (e.g. via set_curr_task), since update_curr() (in the
	 * enqueue of curr) will have resulted in resched being set.  This
	 * prevents us from potentially nominating it as a false LAST_BUDDY
	 * below.
	 */
	if (test_tsk_need_resched(curr))
		return;

	/* Idle tasks are by definition preempted by non-idle tasks. */
	if (unlikely(task_has_idle_policy(curr)) &&
	    likely(!task_has_idle_policy(p)))
		goto preempt;

	/*
	 * Batch and idle tasks do not preempt non-idle tasks (their preemption
	 * is driven by the tick):
	 */
	if (unlikely(p->policy != SCHED_NORMAL) || !sched_feat(WAKEUP_PREEMPTION))
		return;

	find_matching_se(&se, &pse);
	WARN_ON_ONCE(!pse);

	cse_is_idle = se_is_idle(se);
	pse_is_idle = se_is_idle(pse);

	/*
	 * Preempt an idle group in favor of a non-idle group (and don't preempt
	 * in the inverse case).
	 */
	if (cse_is_idle && !pse_is_idle)
		goto preempt;
	if (cse_is_idle != pse_is_idle)
		return;

	update_curr(cfs_rq_of(se));
	if (wakeup_preempt_entity(se, pse) == 1) {
		/*
		 * Bias pick_next to pick the sched entity that is
		 * triggering this preemption.
		 */
		if (!next_buddy_marked)
			set_next_buddy(pse);
		goto preempt;
	}

	return;

preempt:
	resched_curr(rq);
	/*
	 * Only set the backward buddy when the current task is still
	 * on the rq. This can happen when a wakeup gets interleaved
	 * with schedule on the ->pre_schedule() or idle_balance()
	 * point, either of which can * drop the rq lock.
	 *
	 * Also, during early boot the idle thread is in the fair class,
	 * for obvious reasons its a bad idea to schedule back to it.
	 */
	if (unlikely(!se->on_rq || curr == rq->idle))
		return;

	if (sched_feat(LAST_BUDDY) && scale && entity_is_task(se))
		set_last_buddy(se);
}
```

忽略`NEXT_BUDDY`与`LAST_BUDDY`两个调度特定启用时执行的函数、`throttled_hierarchy`函数，这个函数确定新唤醒的任务`p`是否可以抢占cfs_rq之中正在运行的任务，函数中`se`为正在运行任务对应的调度实体，`pse`为新唤醒任务对应的调度实体。代码之中先处理一些基本情况：`se`与`pse`相同意味着被唤醒的任务和正在运行的任务是同一个任务，这种情况不存在任务抢占这种情况所以直接返回，这种情况很少见；当正在运行的任务设置了`TIF_NEED_RESCHED`标记，即意味着任务要被抢占时，不再执行任何操作直接返回。内核之中只能非空闲任务抢占空闲任务，因此只有当正在运行的任务为空闲任务、被唤醒的任务不是空闲任务时跳转到`preempt`标记处继续执行，而当正在运行的任务为非空闲任务、被唤醒的任务为空闲任务时直接返回。`find_match_se`保证`se`以及`pse`对应两个实体在调度层级的同一级之中，这个函数可能会将`pse`或`se`对应的实体改变为更高层级的实体，后边会有这个函数流程的详细记录。`find_match_se`函数改变`se`或`pse`对应的调度实体可能会导致只允许非空闲任务抢占空闲任务这一规则被破坏，因此需要重新进行检查，`cse_is_idle`以及`pse_is_idle`两个变量的检查就是在保证这个规则不被破坏。`update_curr`函数更新正在运行任务的时间统计，最后调用`wakeup_preempt_entity`确定被唤醒的任务能否抢占正在运行的任务，若可以抢占则跳转到`preempt`标记出执行，否则直接退出。`update_curr`函数的流程记录见`task_fork_cfs.md`，`wakeup_preempt_entity`函数流程在后边详细记录。

`preempt`标记处的代码调用`resched_curr`函数为正在运行的任务添加`TIF_NEED_RESCHED`标记，这个函数的流程在`task_wake_up.md`文件中有详细记录。若正在运行的任务没有在rq之中并且不是空闲任务才考虑设置`last_buddy`，`last_buddy`相关的内容直接忽略。这里值得注意的是`se->on_rq`可能为0，`se`对应正在运行任务的调度实体所以此时`on_rq`本不应该为0，这是一种不一致的状态，在内核执行任务切换过程中可能会出现这种不一致的状态。

