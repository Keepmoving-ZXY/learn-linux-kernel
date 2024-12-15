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

在第二个`for`循环对于遍历到的每个实体调用`update_load_avg`函数更新实体对应任务和实体所在cfs_rq的平均负载、调用`se_update_runnable`函数更新任务组在某个cpu上实体的`runnable_weight`为任务组在这个cpu上cfs_rq中可运行任务的数量、调用`update_cfs_group`更新任务组在某个cpu上实体的权重。

这个函数最后的部分调用`sub_nr_running`将rq之中可运行任务的数量递减，因为前边两个`for`循环的作用让rq之中所有的任务都是空闲任务时(`!was_sched_idle && sched_idle_rq(rq)`为`True`)将对此rq进行下一次负载均衡的时机提前到现在(`rq->next_balance = jiffies`)，调用`update_hrtick`更新高精度定时器。

以上的流程中提到`update_load_avg`、`se_update_runnable`、`update_cfs_group`、、`hrtick_update`函数流程详细记录见`enqueue_task_cfs.md`，`sched_idle_rq`、`dequeue_entity`、`sub_nr_running`、`set_next_buddy`函数流程在后边记录，其他函数忽略。

