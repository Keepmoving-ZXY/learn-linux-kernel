本文档记录`__pick_next_task_fair`函数流程，这个函数是Linux内核调度框架之中`pick_next_task`方法的CFS实现，这个函数的作用是选出接下来运行的任务。

### `__pick_next_task_fair`函数

```c
static struct task_struct *__pick_next_task_fair(struct rq *rq)
{
	return pick_next_task_fair(rq, NULL, NULL);
}
```

这个函数主要调用`pick_next_task_fair`并将后两个参数设置为空，接下来重点关注`pick_next_task`函数的内容。

### `pick_next_task_fair`函数

```c
struct task_struct *
pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	struct cfs_rq *cfs_rq = &rq->cfs;
	struct sched_entity *se;
	struct task_struct *p;
	int new_tasks;

again:
	if (!sched_fair_runnable(rq))
		goto idle;

#ifdef CONFIG_FAIR_GROUP_SCHED
	if (!prev || prev->sched_class != &fair_sched_class)
		goto simple;

	/*
	 * Because of the set_next_buddy() in dequeue_task_fair() it is rather
	 * likely that a next task is from the same cgroup as the current.
	 *
	 * Therefore attempt to avoid putting and setting the entire cgroup
	 * hierarchy, only change the part that actually changes.
	 */

	do {
		struct sched_entity *curr = cfs_rq->curr;

		/*
		 * Since we got here without doing put_prev_entity() we also
		 * have to consider cfs_rq->curr. If it is still a runnable
		 * entity, update_curr() will update its vruntime, otherwise
		 * forget we've ever seen it.
		 */
		if (curr) {
			if (curr->on_rq)
				update_curr(cfs_rq);
			else
				curr = NULL;

			/*
			 * This call to check_cfs_rq_runtime() will do the
			 * throttle and dequeue its entity in the parent(s).
			 * Therefore the nr_running test will indeed
			 * be correct.
			 */
			if (unlikely(check_cfs_rq_runtime(cfs_rq))) {
				cfs_rq = &rq->cfs;

				if (!cfs_rq->nr_running)
					goto idle;

				goto simple;
			}
		}

		se = pick_next_entity(cfs_rq, curr);
		cfs_rq = group_cfs_rq(se);
	} while (cfs_rq);

	p = task_of(se);

	/*
	 * Since we haven't yet done put_prev_entity and if the selected task
	 * is a different task than we started out with, try and touch the
	 * least amount of cfs_rqs.
	 */
	if (prev != p) {
		struct sched_entity *pse = &prev->se;

		while (!(cfs_rq = is_same_group(se, pse))) {
			int se_depth = se->depth;
			int pse_depth = pse->depth;

			if (se_depth <= pse_depth) {
				put_prev_entity(cfs_rq_of(pse), pse);
				pse = parent_entity(pse);
			}
			if (se_depth >= pse_depth) {
				set_next_entity(cfs_rq_of(se), se);
				se = parent_entity(se);
			}
		}

		put_prev_entity(cfs_rq, pse);
		set_next_entity(cfs_rq, se);
	}

	goto done;
simple:
#endif
	if (prev)
		put_prev_task(rq, prev);

	do {
		se = pick_next_entity(cfs_rq, NULL);
		set_next_entity(cfs_rq, se);
		cfs_rq = group_cfs_rq(se);
	} while (cfs_rq);

	p = task_of(se);

done: __maybe_unused;
#ifdef CONFIG_SMP
	/*
	 * Move the next running task to the front of
	 * the list, so our cfs_tasks list becomes MRU
	 * one.
	 */
	list_move(&p->se.group_node, &rq->cfs_tasks);
#endif

	if (hrtick_enabled_fair(rq))
		hrtick_start_fair(rq, p);

	update_misfit_status(p, rq);

	return p;

idle:
	if (!rf)
		return NULL;

	new_tasks = newidle_balance(rq, rf);

	/*
	 * Because newidle_balance() releases (and re-acquires) rq->lock, it is
	 * possible for any higher priority task to appear. In that case we
	 * must re-start the pick_next_entity() loop.
	 */
	if (new_tasks < 0)
		return RETRY_TASK;

	if (new_tasks > 0)
		goto again;

	/*
	 * rq is about to be idle, check if we need to update the
	 * lost_idle_time of clock_pelt
	 */
	update_idle_rq_clock_pelt(rq);

	return NULL;
}
```

#### 最简单的情况

当rq之中CFS管理的可运行任务数量为0时，代码跳转到标签`idle`处继续执行；若传入的`prev`为空或者`prev`关联的调度类不是CFS，代码跳转到标签`simple`处执行，这里的`prev`不为空时指向运行队列`rq`中正在运行的任务，这意味着当正在运行的任务关联的调度类不是CFS但是新执行的任务要关联CFS。

#### 选择一个任务

这部分的内容对应代码之中`do { ... } while (cfs_rq)`这个循环，这个循环在调度层级中的任务组之中寻找一个可运行的任务。循环中`curr`为CFS运行队列`cfs_rq`中正在运行的任务，如果这个任务依然在`rq`之中(`on_rq`不为0)则意味着此时没有调用`put_prev_entity`，调用`update_curr`函数更新正在运行任务的运行时间统计，其他情况下设置`curr`为空。接下来调用`pick_next_entity`函数从`cfs_rq`之中选择一个可运行的实体，如果实体对应的是一个任务组则`cfs_rq`为任务组在当前cpu上的运行队列，这种情况下会继续进行循环；如果实体对应的是一个任务则不再继续执行循环，开始执行后边的流程。这里提到的`update_curr`函数详细记录见`task_fork_cfs.md`，`put_prev_entity`函数详细记录见`put_set_task_cfs.md`文件，在后边详细记录`pick_next_task`函数流程，其他的内容忽略。

#### 不同的调度层级

这部分的内容对应代码之中`if (prev != p) {... }`中的代码，`prev`为当前cpu中正在运行的任务、`p`为接下来运行的任务，如果二者不属于同一个任务组那么说明二者在不同的调度层级之中，`se_depth`为接下来运行任务所在的调度层级、`pse_depth`为正在运行任务所在的调度层级，这个数值越小意味着越靠近任务组的最高层次，即靠近根任务组；反之意味着越靠近任务组的最低层次，即靠近只有任务的层次。代码之中关于`se_depth`、`pse_depth`大小比较的两个`if`成立时执行的代码说明了二者不太同一个调度层级的成立方式：若接下来运行的任务所在层级更高，需要将调度层级中正在运行任务的上级从运行队列之中移除直到某个上级与接下来运行的任务在同一个调度层级，将正在运行的任务设置为这个上级；若正在运行的任务所在层级更高，需要将调度层级之中接下来运行任务的上级加入到运行队列之中直到某个上级与正在运行的任务在同一个调度层级，将即将运行的任务设置为这个上级。完成调整之后正在运行的任务与即将运行的任务处于同一个调度层级，将正在运行的任务从运行队列之中移除、将即将运行的任务加入到运行队列之中，跳转到标签`done`处执行。这里提到的从运行队列中移除、加入运行队列两个操作由`put_prev_entity`、`set_next_entity`函数执行，这两个函数的详细记录见`put_set_task_cfs.md`，其他内容忽略。

#### 标签`simple`

这部分的内容对应代码之中标签`simple`处的代码，若`prev`不为空即传入正在运行的任务执行`put_prev_task`函数将任务对应的实体以及上级实体从对应的运行队列之中移除，这部分代码主要内容是一个`for`循环，这个循环在`se`对应任务组时保持`cfs_rq`为任务组在当前cpu中的CFS运行队列，不断的从`cfs_rq`之中选择一个新的实体，直到这个实体对应的是一个任务而非任务组。找到这样一个实体之后，执行标签`done`处的代码。这里提到的`put_prev_task`函数详细记录见`put_set_task_cfs.md`，在后边详细记录`pick_next_task`函数流程，其他的内容忽略。

#### 标签`done`

这部分的内容对应代码之中标签`done`处的代码，这些代码将接下来运行的任务放到运行队列`rq`之中`cfs_tasks`列表的表头，这样`cfs_tasks`之中的任务满足MRU(最近经常使用)算法的要求；若允许使用高精度定时器，设置一个定时器并且触发的时间是接下来运行任务时间片耗尽的时间。完成这些操作之后返回接下来运行的任务。这里提到的`hrtick_start_fair`函数详细记录见`enqueue_task_cfs.md`，其他的内容忽略。

#### 标签`idle`

这部分的内容对应代码之中`idle`处的代码，这些代码在运行队列`rq`不为空的前提下执行。调用`newidle_balance`函数进行cpu之间任务的负载均衡，尝试从其他的cpu之中拉去可运行的任务到当前的cpu之中，若返回`-1`意味着没有找到关联CFS调度类的任务，返回-1；若返回值大于0则意味着拉去到了新的任务，代码跳转到标签`again`处即函数开始的位置继续执行；若返回值等于0意味着没有找到可以运行的任务，则调用`update_idle_rq_clock_pelt`函数更新空闲运行队列的相关统计并返回0。在后边记录这里提到的`newidle_balance`、`update_idle_rq_clock_pelt`两个函数流程，其他的内容忽略。