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

### `newidle_balance`函数

```c
/*
 * newidle_balance is called by schedule() if this_cpu is about to become
 * idle. Attempts to pull tasks from other CPUs.
 *
 * Returns:
 *   < 0 - we released the lock and there are !fair tasks present
 *     0 - failed, no new tasks
 *   > 0 - success, new (fair) tasks present
 */
static int newidle_balance(struct rq *this_rq, struct rq_flags *rf)
{
	unsigned long next_balance = jiffies + HZ;
	int this_cpu = this_rq->cpu;
	u64 t0, t1, curr_cost = 0;
	struct sched_domain *sd;
	int pulled_task = 0;

	update_misfit_status(NULL, this_rq);

	/*
	 * There is a task waiting to run. No need to search for one.
	 * Return 0; the task will be enqueued when switching to idle.
	 */
	if (this_rq->ttwu_pending)
		return 0;

	/*
	 * We must set idle_stamp _before_ calling idle_balance(), such that we
	 * measure the duration of idle_balance() as idle time.
	 */
	this_rq->idle_stamp = rq_clock(this_rq);

	/*
	 * Do not pull tasks towards !active CPUs...
	 */
	if (!cpu_active(this_cpu))
		return 0;

	/*
	 * This is OK, because current is on_cpu, which avoids it being picked
	 * for load-balance and preemption/IRQs are still disabled avoiding
	 * further scheduler activity on it and we're being very careful to
	 * re-start the picking loop.
	 */
	rq_unpin_lock(this_rq, rf);

	rcu_read_lock();
	sd = rcu_dereference_check_sched_domain(this_rq->sd);

	if (!READ_ONCE(this_rq->rd->overload) ||
	    (sd && this_rq->avg_idle < sd->max_newidle_lb_cost)) {

		if (sd)
			update_next_balance(sd, &next_balance);
		rcu_read_unlock();

		goto out;
	}
	rcu_read_unlock();

	raw_spin_rq_unlock(this_rq);

	t0 = sched_clock_cpu(this_cpu);
	update_blocked_averages(this_cpu);

	rcu_read_lock();
	for_each_domain(this_cpu, sd) {
		int continue_balancing = 1;
		u64 domain_cost;

		update_next_balance(sd, &next_balance);

		if (this_rq->avg_idle < curr_cost + sd->max_newidle_lb_cost)
			break;

		if (sd->flags & SD_BALANCE_NEWIDLE) {

			pulled_task = load_balance(this_cpu, this_rq,
						   sd, CPU_NEWLY_IDLE,
						   &continue_balancing);

			t1 = sched_clock_cpu(this_cpu);
			domain_cost = t1 - t0;
			update_newidle_cost(sd, domain_cost);

			curr_cost += domain_cost;
			t0 = t1;
		}

		/*
		 * Stop searching for tasks to pull if there are
		 * now runnable tasks on this rq.
		 */
		if (pulled_task || this_rq->nr_running > 0 ||
		    this_rq->ttwu_pending)
			break;
	}
	rcu_read_unlock();

	raw_spin_rq_lock(this_rq);

	if (curr_cost > this_rq->max_idle_balance_cost)
		this_rq->max_idle_balance_cost = curr_cost;

	/*
	 * While browsing the domains, we released the rq lock, a task could
	 * have been enqueued in the meantime. Since we're not going idle,
	 * pretend we pulled a task.
	 */
	if (this_rq->cfs.h_nr_running && !pulled_task)
		pulled_task = 1;

	/* Is there a task of a high priority class? */
	if (this_rq->nr_running != this_rq->cfs.h_nr_running)
		pulled_task = -1;

out:
	/* Move the next balance forward */
	if (time_after(this_rq->next_balance, next_balance))
		this_rq->next_balance = next_balance;

	if (pulled_task)
		this_rq->idle_stamp = 0;
	else
		nohz_newidle_balance(this_rq);

	rq_repin_lock(this_rq, rf);

	return pulled_task;
}
```

这个函数的内容涉及到了Linux内核的调度域等概念，这些概念参见[内核调度域说明](https://docs.kernel.org/scheduler/sched-domains.html)。

#### 简单情况处理

这部分对应的代码为从函数开始到`raw_spin_rq_unlock`函数调用位置，代码流程为一些比较简单情况的处理。若运行队列`rq`之中已经有任务正在等待运行或者当前cpu处于不活跃状态(`!cpu_active(this_cpu)`)则直接返回，这里通过`ttwu_pending`来标记是否存在任务需要等待运行，`ttwu_pending`的设置`try_to_wake_up`函数中设置，这个函数的详细记录见`task_wake_up.md`。运行队列中`idle_stamp`字段保存运行队列进入空闲状态的时间，这个时间不是系统的当前时间而是运行队列的当前时间，运行队列之中需一个相对稳定而不是一直在变化的时间用于各种统计计算，所以运行队列会单独维护一个当前时间。使用`rcu_read_lock`、`rcu_read_unlock`标记一个RCU读临界区，在这个临界区中若根调度域没有出现过载的情况、当前运行队列处于空闲状态的时间小于任务负载均衡自身消耗的时间(``max_newidle_lb_cost`)，调用`update_next_balance`更新下一次执行负载均衡的时间，跳转到标签`out`处执行。上述流程中提到的`update_next_balance`函数的详细流程在后边记录，其他的函数忽略。

#### 从其他cpu中拉取任务

这部分对应代码中`raw_spin_rq_unlock`与`raw_spin_rq_lock`这两个函数调用位置之间的代码，这两个函数用于释放、获取运行队列之中的锁：这个过程中不需要对运行队列自身进行修改，所以先释放在`__schedule`函数间接调用此函数之前获取到的运行队列之中的锁；尝试从调度域其他cpu中拉取任务过程结束之后需要重新获取运行队列之中的锁，一方面是因为接下来要对运行队列进行修改、另外一方面是保持`__schedule`间接调用此函数时锁的一致性。

设置`t0`为尝试从调度域其他cpu中拉取任务的开始时间，调用`update_blocked_averages`函数更新运行队列之中平均负载统计，这个函数名之中`blocked`含义为`blocked load`，结合`update_tg_cfs_util`之前的注释认为`blocked load`为运行队列之中任务的历史权重，在计算平均负载的时候会考虑任务的历史权重。从调度域其他cpu之中拉取任务对应的代码在`for_each_domain`这个循环之中，这个循环遍历当前cpu所在的调度域，对于每个调度域调用`update_next_balance`函数更新此调度域下次执行负载均衡的时间、调用`load_balance`函数尝试从调度域中其他cpu拉取一个任务、调用`update_newidle_cost`更新调度域负载均衡消耗的时间，若已经拉取到任务或者运行队列中已经有可运行的任务或者运行队列中有等待运行的任务直接退出调度域遍历过程。在循环过程中还会记录对正在遍历的调度域中调用`load_balance`函数进行负载均衡的时间，更新正在遍历的调度域的负载均衡所需时间，还会判断运行队列处于空闲状态时间(`avg_idle`)与尝试从其他cpu中拉取任务累计消耗时间(`curr_cost`)、负载均衡操作消耗的最长时间(`max_newidle_lb_cost`)，若前者小于后者则认为从其他的cpu中拉取任务是低效的、直接放弃未遍历到的调度域。

在后边详细记录上述流程中提到的`update_blocked_averages`、`update_newidle_cost`、`update_next_balance`函数，`load_balance`内容十分复杂需要在一个文档之中单独记录流程，其他的函数忽略。

#### 任务拉取结束的检查

这部分对应代码中调用`raw_spin_rq_lock`位置到标签`out`之间的代码，这部分代码更新当前运行队列之中负载均衡操作消耗的最长时间统计(`this_rq->max_idle_balance_cost`)，对两种特殊情况进行判断：若CFS运行队列之中有可运行的任务并且没能从其他的cpu中拉取任务，设置`pulled_task`为1表示已经有新的、可运行的任务了；若运行队列中可运行的任务大于CFS运行队列中可运行的任务，这意味着运行队列之中有任务关联了优先级更高的调度类，设置`pulled_task`为`-1`提醒调用者这种情况。

#### 标签`out`

若当前的运行队列设置的下次负载均衡运行(`this_rq->max_idle_balance_cost`)时间晚于负载均衡过程中计算出来的时间(`next_balance`)，则提前当前运行队列下次负载均衡运行时间至负载均衡过程中计算出来的时间。若已经拉取到任务或者当前运行队列之中有优先级更高的任务，运行队列即将退出空闲状态，重置`idle_stamp`为0；若没有拉取到任务则调用`nohz_newidle_balance`函数进行负载均衡相关的检查工作。在后边详细记录`nohz_newidle_balance`函数流程，其他的函数忽略。

### `update_next_balance`函数

```c
static inline void
update_next_balance(struct sched_domain *sd, unsigned long *next_balance)
{
	unsigned long interval, next;

	/* used by idle balance, so cpu_busy = 0 */
	interval = get_sd_balance_interval(sd, 0);
	next = sd->last_balance + interval;

	if (time_after(*next_balance, next))
		*next_balance = next;
}
```

这个函数计算下一次运行负载均衡的时间，由`get_sd_balance_interval`计算调度域`sd`上负载均衡的运行间隔，加上调度域`sd`最近一次运行负载均衡的时间得到下一次负载均衡的运行时间。在`newidle_balance`函数会对多个调度域计算下一次负载均衡运行的时间，从函数之中最后一个`if`中的代码可以看到下一次负载均衡运行的时间总是目前处理的调度域中最早触发负载均衡的时间。`get_sd_balance_interval`函数是这个函数的重点，接下来关注这个函数的内容。

### `get_sd_balance_interval`函数

```c
static inline unsigned long
get_sd_balance_interval(struct sched_domain *sd, int cpu_busy)
{
	unsigned long interval = sd->balance_interval;

	if (cpu_busy)
		interval *= sd->busy_factor;

	/* scale ms to jiffies */
	interval = msecs_to_jiffies(interval);

	/*
	 * Reduce likelihood of busy balancing at higher domains racing with
	 * balancing at lower domains by preventing their balancing periods
	 * from being multiples of each other.
	 */
	if (cpu_busy)
		interval -= 1;

	interval = clamp(interval, 1UL, max_load_balance_interval);

	return interval;
}
```

这个函数基于调度域默认的负载均衡时间间隔(`balance_interval`)，结合当前cpu是否处于忙碌状态，计算出一个负载均衡时间间隔。若当前cpu处于忙碌状态，将调度域默认的负载均衡时间间隔放大，减少负载均衡的运行次数。在当前cpu处于忙碌状态的时候，为了防止不同层次的调度域执行负载均衡的时间成为倍数关系造成的不同层次调度域连续运行负载均衡操作，递减调度域`sd`的时间间隔。最后为调度域`sd`运行负载均衡的时间间隔加一个上限、下限限制作为最终的结果并返回。
