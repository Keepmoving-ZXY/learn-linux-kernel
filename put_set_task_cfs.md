本文档记录`put_prev_task_fair`以及`set_next_task_fair`两个函数的详细流程，这两个函数是内核调度框架之中`put_prev_task`以及`set_next_task`方法的CFS实现。`put_prev_task`的作用是告诉调度类另外一个任务即将取代正在执行的任务，这个时候可以执行一些统计信息的更新；在任务在任务组或调度类之前转移时`set_next_task`设置调度类内部rq的正在运行的任务(`current`)并更新统计信息。

### `put_prev_task_fair`函数

```c
/*
 * Account for a descheduled task:
 */
static void put_prev_task_fair(struct rq *rq, struct task_struct *prev)
{
	struct sched_entity *se = &prev->se;
	struct cfs_rq *cfs_rq;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		put_prev_entity(cfs_rq, se);
	}
}
```

这个函数遍历调度层级之中的每一级中的实体，对每个实体调用`put_prev_entity`函数，这个函数在实体对应的任务离开cfs_rq之前更新cfs_rq之中的统计信息，接下来详细记录这个函数的流程。

### `put_prev_entity`函数

```c
static void put_prev_entity(struct cfs_rq *cfs_rq, struct sched_entity *prev)
{
	/*
	 * If still on the runqueue then deactivate_task()
	 * was not called and update_curr() has to be done:
	 */
	if (prev->on_rq)
		update_curr(cfs_rq);

	/* throttle cfs_rqs exceeding runtime */
	check_cfs_rq_runtime(cfs_rq);

	check_spread(cfs_rq, prev);

	if (prev->on_rq) {
		update_stats_wait_start_fair(cfs_rq, prev);
		/* Put 'current' back into the tree. */
		__enqueue_entity(cfs_rq, prev);
		/* in !on_rq case, update occurred at dequeue */
		update_load_avg(cfs_rq, prev, 0);
	}
	cfs_rq->curr = NULL;
}
```

当任务`p`还在rq之中时调用`update_curr`更新正在运行任务的时间统计、调用`__enqueue_entity`将任务`p`加入到`cfs_rq`的红黑树之中、调用`upload_load_avg`同步`cfs_rq`以及`prev`之间的统计信息，最后清空`cfs_rq`之中正在运行的任务(`curr = NULL`)。这里提到的`update_curr`函数详细记录见`task_fork_cfs.md`，`update_load_avg`函数详细记录见`enqueue_task_cfs.md`，`__enqueue_entity`函数详细记录见`enqueue_task_cfs.md`，其他的函数忽略。

这个函数中的大部分逻辑都是在`prev->on_rq`不为0的情况下触发的，直觉上调用这个函数的时候`p`应该已经从rq之中移除了，这里记录一种在`prev`没有从rq中移除时调用此函数的情况。在`schedule.md`之中记录了`__schedule`函数，这个函数中当被抢占的任务不再处于正在运行状态、没有禁止任务抢占的情况下(`!(sched_mode & SM_MASK_PREEMPT) && prev_state`为`1`)，会判断被抢占的任务是否有阻塞的信号需要处理，当有阻塞的信号需要处理时就会发生调用此函数的时候传入的任务的`on_rq`不为0。

