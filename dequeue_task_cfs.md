本文档记录`dequeue_task_fair`函数的详细流程，这个函数是CFS调度类提供的`dequeue_task`方法的实现，当一个任务进入睡眠状态或者退出时需要将其从rq之中移除，从rq之中移除时会调用CFS提供的`dequeue_task`方法实现，更新CFS调度类内部的数据结构。

### `dequeue_task_fair`函数

```c
/*
 * The dequeue_task method is called before nr_running is
 * decreased. We remove the task from the rbtree and
 * update the fair scheduling stats:
 */
static void dequeue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &p->se;
	int task_sleep = flags & DEQUEUE_SLEEP;
	int idle_h_nr_running = task_has_idle_policy(p);
	bool was_sched_idle = sched_idle_rq(rq);

	util_est_dequeue(&rq->cfs, p);

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		dequeue_entity(cfs_rq, se, flags);

		cfs_rq->h_nr_running--;
		cfs_rq->idle_h_nr_running -= idle_h_nr_running;

		if (cfs_rq_is_idle(cfs_rq))
			idle_h_nr_running = 1;

		/* end evaluation on encountering a throttled cfs_rq */
		if (cfs_rq_throttled(cfs_rq))
			goto dequeue_throttle;

		/* Don't dequeue parent if it has other entities besides us */
		if (cfs_rq->load.weight) {
			/* Avoid re-evaluating load for this entity: */
			se = parent_entity(se);
			/*
			 * Bias pick_next to pick a task from this cfs_rq, as
			 * p is sleeping when it is within its sched_slice.
			 */
			if (task_sleep && se && !throttled_hierarchy(cfs_rq))
				set_next_buddy(se);
			break;
		}
		flags |= DEQUEUE_SLEEP;
	}

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);

		update_load_avg(cfs_rq, se, UPDATE_TG);
		se_update_runnable(se);
		update_cfs_group(se);

		cfs_rq->h_nr_running--;
		cfs_rq->idle_h_nr_running -= idle_h_nr_running;

		if (cfs_rq_is_idle(cfs_rq))
			idle_h_nr_running = 1;

		/* end evaluation on encountering a throttled cfs_rq */
		if (cfs_rq_throttled(cfs_rq))
			goto dequeue_throttle;

	}

	/* At this point se is NULL and we are at root level*/
	sub_nr_running(rq, 1);

	/* balance early to pull high priority tasks */
	if (unlikely(!was_sched_idle && sched_idle_rq(rq)))
		rq->next_balance = jiffies;

dequeue_throttle:
	util_est_update(&rq->cfs, p, task_sleep);
	hrtick_update(rq);
}
```

这个函数的主要流程都在两个`for`循环之中执行，这两个`for`循环都是在遍历调度层级之中的每一级的实体，遍历过程中更新cfs_rq之中可运行任务(`h_nr_running`)与可运行空闲任务(`idle_h_nr_running`)的统计，这里是减小这两个统计的值，而`enqueue_task_fair`是增加这两个统计的值，这两个统计的更新逻辑结合`enqueue_task_fair`之中的解释理解。

在第一个循环中对每个实体调用`dequeue_entity`函数更新任务以及任务组相关统计随后将任务从cfs_rq之中移除，这里需要注意的是并不是遍历到的所有的实体都需要从其所在的cfs_rq之中移除，当实体所在的cfs_rq之中还有任务时停止向上遍历，这里判断cfs_rq之中是否还有任务是通过当前实体所在cfs_rq的权重(`cfs_rq->load.weight`)是否为0来确定的。这里需要留意`parent_entity`返回的实体与`cfs_rq`之间的关系，它们是一个任务组在某个cpu上的实体以及运行队列，实体参与这个cpu上的任务调度、运行队列保存任务组之中可运行的任务。接下来关注cfs_rq的权重不为0时的处理，通过`parent_entity`获取当前实体在调度层级之中的上级实体，若任务`p`在时间片耗尽之前进入睡眠状态并且这一层没有被限流，那么将上级调度实体设置为这个实体所在cfs_rq之中期望接下来执行的任务对应的实体。这里通过`task_sleep`来判断任务`p`是否是一个在时间片耗尽之前就进入睡眠状态的任务，若`task_sleep`为1意味着调用这个函数的时候`flags`之中设置了`DEQUEUE_SLEEP`标志，这意味着正在执行的任务时间片还没有消耗完但要进入睡眠状态、新的任务要抢占正在运行的任务（若任务的时间片已经消耗完时进行抢占，任务处于可运行状态），可以结合`__schedule`函数之中`signal_pending_state(prev_state, prev)`为`False`时执行的代码逻辑理解。

在第二个`for`循环对于遍历到的每个实体调用`update_load_avg`函数更新实体对应任务和实体所在cfs_rq的平均负载、调用`se_update_runnable`函数更新任务组在某个cpu上实体的可运行任务统计、调用`update_cfs_group`更新任务组在某个cpu上实体的权重。

这个函数最后的部分调用`sub_nr_running`将rq之中可运行任务的数量递减，因为前边两个`for`循环的作用让rq之中所有的任务都是空闲任务时(`!was_sched_idle && sched_idle_rq(rq)`为`True`)将对此rq进行下一次负载均衡的时机提前到现在(`rq->next_balance = jiffies`)，调用`update_hrtick`更新高精度定时器。

以上的流程中提到的`update_load_avg`、`se_update_runnable`、`update_cfs_group`、`hrtick_update`函数流程详细记录见`enqueue_task_cfs.md`，`sched_idle_rq`、`dequeue_entity`、`sub_nr_running`函数流程在后边记录，其他函数忽略。

### `sched_idle_rq`函数

```c
/* Runqueue only has SCHED_IDLE tasks enqueued */
static int sched_idle_rq(struct rq *rq)
{
	return unlikely(rq->nr_running == rq->cfs.idle_h_nr_running &&
			rq->nr_running);
}
```

这个函数判断一个rq之中是否只包含空闲任务，判断的方式为比较rq中可运行任务以及可运行空闲任务的数量，二者相等则意味着rq之中所有任务都是空闲任务。当rq中可运行任务为0时这种判断会误认为rq之中所有任务为空闲任务，需要在判断判断之中排除此种情况，因此会要求`nr_running`不为0。

### `dequeue_entity`函数

```c
static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
	int action = UPDATE_TG;

	if (entity_is_task(se) && task_on_rq_migrating(task_of(se)))
		action |= DO_DETACH;

	/*
	 * Update run-time statistics of the 'current'.
	 */
	update_curr(cfs_rq);

	/*
	 * When dequeuing a sched_entity, we must:
	 *   - Update loads to have both entity and cfs_rq synced with now.
	 *   - For group_entity, update its runnable_weight to reflect the new
	 *     h_nr_running of its group cfs_rq.
	 *   - Subtract its previous weight from cfs_rq->load.weight.
	 *   - For group entity, update its weight to reflect the new share
	 *     of its group cfs_rq.
	 */
	update_load_avg(cfs_rq, se, action);
	se_update_runnable(se);

	update_stats_dequeue_fair(cfs_rq, se, flags);

	clear_buddies(cfs_rq, se);

	if (se != cfs_rq->curr)
		__dequeue_entity(cfs_rq, se);
	se->on_rq = 0;
	account_entity_dequeue(cfs_rq, se);

	/*
	 * Normalize after update_curr(); which will also have moved
	 * min_vruntime if @se is the one holding it back. But before doing
	 * update_min_vruntime() again, which will discount @se's position and
	 * can move min_vruntime forward still more.
	 */
	if (!(flags & DEQUEUE_SLEEP))
		se->vruntime -= cfs_rq->min_vruntime;

	/* return excess runtime on last dequeue */
	return_cfs_rq_runtime(cfs_rq);

	update_cfs_group(se);

	/*
	 * Now advance min_vruntime if @se was the entity holding it back,
	 * except when: DEQUEUE_SAVE && !DEQUEUE_MOVE, in this case we'll be
	 * put back on, and if we advance min_vruntime, we'll be placed back
	 * further than we started -- ie. we'll be penalized.
	 */
	if ((flags & (DEQUEUE_SAVE | DEQUEUE_MOVE)) != DEQUEUE_SAVE)
		update_min_vruntime(cfs_rq);

	if (cfs_rq->nr_running == 0)
		update_idle_cfs_rq_clock_pelt(cfs_rq);
}
```

若实体`se`对应一个任务并且这个任务要从运行此函数的当前cpu之中迁移出去，需要为`action`追加`DO_DETACH`标记，这个标记会导致在`update_load_avg`函数中从cfs_rq的`avg`以及`sum`指标中减去来自实体`se`的贡献。一个任务从cfs_rq之中移除时同步这个任务与其所在的cfs_rq、任务组统计信息的最后时机，这里调用`update_curr`函数更新正在运行任务的时间统计、调用`update_load_avg`将实体`se`对`avg`、`sum`指标的贡献同步到cfs_rq之中、调用`se_update_runnable`更新任务组在当前cpu上的实体的可运行任务统计。如果`se`对应的不是当前cpu中正在运行的任务，调用`__dequeue_entity`函数将实体`se`从当前cpu的cfs_rq之中将实体`se`的`on_rq`设置为0以表明实体已经不在rq之中、调用`account_entity_dequeue`函数将任务权重从cfs_rq之中移除。若实体`se`对的任务并非进入睡眠状态而从cfs_rq移除，要从任务的虚拟运行时间之中减去cfs_rq的最小运行时间，这是将任务的虚拟运行时间转换成相对虚拟运行时间。若实体`se`为任务组在当前cpu上的实体，重新计算这个实体的权重。若在`flags`之中同时指定了`DEQUEUE_SAVE`和`DEQUEUE_MOVE`标记，则意味着实体`se`对应的任务要离开当前的rq，此时调用`update_min_vruntime`函数更新cfs_rq的基准虚拟运行时间，结合调用场景理解：在`sched_move_task`函数要为将任务转移到新的rq之中，此时调用此函数时同时指定了`DEQUEUE_SAVE`和`DEQUEUE_MOVE`标记；在`set_user_nice`函数中调用此函数时指定了`DEQUEUE_SAVE`而没有指定`DEQUEUE_MOVE`。

以上的流程之中涉及到的`update_curr`函数详细记录见`task_fork_cfs.md`，`update_load_avg`、`se_update_runnable`、`update_cfs_group`函数详细记录见`enqueue_task_cfs.md`，`update_min_vruntime`函数详细记录见`task_fork_cfs.md`，在后边详细记录`account_entity_dequeue`、`__dequeue_entity`这两个函数的流程，其他的函数忽略。

### `account_entity_dequeue`函数

```c
static void
account_entity_dequeue(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	update_load_sub(&cfs_rq->load, se->load.weight);
#ifdef CONFIG_SMP
	if (entity_is_task(se)) {
		account_numa_dequeue(rq_of(cfs_rq), task_of(se));
		list_del_init(&se->group_node);
	}
#endif
	cfs_rq->nr_running--;
	if (se_is_idle(se))
		cfs_rq->idle_nr_running--;
}
```

这个函数从cfs_rq的权重之中减去实体`se`的贡献，递减cfs_rq之中可运行任务的统计，若`se`对应的任务为一个空闲任务则递减cfs_rq中可运行的空闲任务统计，其他的内容忽略。

### `__dequeue_entity`函数

```c
static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	rb_erase_cached(&se->run_node, &cfs_rq->tasks_timeline);
}
```

这个函数将`se`从cfs_rq之中移除，这意味这将任务从cfs_rq之中移除，这个函数的逻辑正好与`__enqueue_entity`函数相反。

### `sub_nr_running`函数

```c
static inline void sub_nr_running(struct rq *rq, unsigned count)
{
	rq->nr_running -= count;
	if (trace_sched_update_nr_running_tp_enabled()) {
		call_trace_sched_update_nr_running(rq, -count);
	}

	/* Check if we still need preemption */
	sched_update_tick_dependency(rq);
}
```

这个函数修改rq之中可运行任务的数量，从中减去`count`的值，其他的内容忽略。
