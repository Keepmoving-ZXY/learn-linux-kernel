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

这部分的内容对应代码之中`do { ... } while (cfs_rq)`这个循环，这个循环在调度层级中的任务组之中寻找一个可运行的任务。循环中`curr`为CFS运行队列`cfs_rq`中正在运行的任务，如果这个任务依然在`rq`之中(`on_rq`不为0)则意味着此时没有调用`put_prev_entity`，调用`update_curr`函数更新正在运行任务的运行时间统计，其他情况下设置`curr`为空。接下来调用`pick_next_entity`函数从`cfs_rq`之中选择一个可运行的实体，如果实体对应的是一个任务组则`cfs_rq`为任务组在当前cpu上的运行队列，这种情况下会继续进行循环；如果实体对应的是一个任务则不再继续执行循环，开始执行后边的流程。这里提到的`update_curr`函数详细记录见`task_fork_cfs.md`，`put_prev_entity`函数详细记录见`put_set_task_cfs.md`文件，在后边详细记录`pick_next_entity`函数流程，其他的内容忽略。

#### 不同的调度层级

这部分的内容对应代码之中`if (prev != p) {... }`中的代码，`prev`为当前cpu中正在运行的任务、`p`为接下来运行的任务，如果二者不属于同一个任务组那么说明二者在不同的调度层级之中，`se_depth`为接下来运行任务所在的调度层级、`pse_depth`为正在运行任务所在的调度层级，这个数值越小意味着越靠近任务组的最高层次，即靠近根任务组；反之意味着越靠近任务组的最低层次，即靠近只有任务的层次。代码之中关于`se_depth`、`pse_depth`大小比较的两个`if`成立时执行的代码说明了二者不太同一个调度层级的成立方式：若接下来运行的任务所在层级更高，需要将调度层级中正在运行任务的上级从运行队列之中移除直到某个上级与接下来运行的任务在同一个调度层级，将正在运行的任务设置为这个上级；若正在运行的任务所在层级更高，需要将调度层级之中接下来运行任务的上级加入到运行队列之中直到某个上级与正在运行的任务在同一个调度层级，将即将运行的任务设置为这个上级。完成调整之后正在运行的任务与即将运行的任务处于同一个调度层级，将正在运行的任务从运行队列之中移除、将即将运行的任务加入到运行队列之中，跳转到标签`done`处执行。这里提到的从运行队列中移除、加入运行队列两个操作由`put_prev_entity`、`set_next_entity`函数执行，这两个函数的详细记录见`put_set_task_cfs.md`，其他内容忽略。

#### 标签`simple`

这部分的内容对应代码之中标签`simple`处的代码，若`prev`不为空即传入正在运行的任务执行`put_prev_task`函数将任务对应的实体以及上级实体从对应的运行队列之中移除，这部分代码主要内容是一个`for`循环，这个循环在`se`对应任务组时保持`cfs_rq`为任务组在当前cpu中的CFS运行队列，不断的从`cfs_rq`之中选择一个新的实体，直到这个实体对应的是一个任务而非任务组。找到这样一个实体之后，执行标签`done`处的代码。这里提到的`put_prev_task`函数详细记录见`put_set_task_cfs.md`，在后边详细记录`pick_next_entity`函数流程，其他的内容忽略。

#### 标签`done`

这部分的内容对应代码之中标签`done`处的代码，这些代码将接下来运行的任务放到运行队列`rq`之中`cfs_tasks`列表的表头，这样`cfs_tasks`之中的任务满足MRU(最近经常使用)算法的要求；若允许使用高精度定时器，设置一个定时器并且触发的时间是接下来运行任务时间片耗尽的时间。完成这些操作之后返回接下来运行的任务。这里提到的`hrtick_start_fair`函数详细记录见`enqueue_task_cfs.md`，其他的内容忽略。

#### 标签`idle`

这部分的内容对应代码之中`idle`处的代码，这些代码在运行队列`rq`不为空的前提下执行。调用`newidle_balance`函数进行cpu之间任务的负载均衡，尝试从其他的cpu之中拉去可运行的任务到当前的cpu之中，若返回`-1`意味着没有找到关联CFS调度类的任务，返回-1；若返回值大于0则意味着拉去到了新的任务，代码跳转到标签`again`处即函数开始的位置继续执行；若返回值等于0意味着没有找到可以运行的任务，则调用`update_idle_rq_clock_pelt`函数更新空闲运行队列的相关统计并返回0。在后边记录这里提到的`newidle_balance`、`update_idle_rq_clock_pelt`两个函数流程，其他的内容忽略。

### `pick_next_entity`函数

```c
/*
 * Pick the next process, keeping these things in mind, in this order:
 * 1) keep things fair between processes/task groups
 * 2) pick the "next" process, since someone really wants that to run
 * 3) pick the "last" process, for cache locality
 * 4) do not run the "skip" process, if something else is available
 */
static struct sched_entity *
pick_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	struct sched_entity *left = __pick_first_entity(cfs_rq);
	struct sched_entity *se;

	/*
	 * If curr is set we have to see if its left of the leftmost entity
	 * still in the tree, provided there was anything in the tree at all.
	 */
	if (!left || (curr && entity_before(curr, left)))
		left = curr;

	se = left; /* ideally we run the leftmost entity */

	/*
	 * Avoid running the skip buddy, if running something else can
	 * be done without getting too unfair.
	 */
	if (cfs_rq->skip && cfs_rq->skip == se) {
		struct sched_entity *second;

		if (se == curr) {
			second = __pick_first_entity(cfs_rq);
		} else {
			second = __pick_next_entity(se);
			if (!second || (curr && entity_before(curr, second)))
				second = curr;
		}

		if (second && wakeup_preempt_entity(second, left) < 1)
			se = second;
	}

	if (cfs_rq->next && wakeup_preempt_entity(cfs_rq->next, left) < 1) {
		/*
		 * Someone really wants this to run. If it's not unfair, run it.
		 */
		se = cfs_rq->next;
	} else if (cfs_rq->last && wakeup_preempt_entity(cfs_rq->last, left) < 1) {
		/*
		 * Prefer last buddy, try to return the CPU to a preempted task.
		 */
		se = cfs_rq->last;
	}

	return se;
}
```

这个函数从CFS运行队列`cfs_rq`之中选择接下来运行的任务，`curr`为`cfs_rq`之中正在运行任务对应的调度实体，这个函数的主要流程如下：

1).确定虚拟运行时间最短的任务，对比CFS运行队列之中虚拟时间最短的任务以及正在运行的任务的虚拟运行时间即可得到；

2).如果这个任务需要跳过，那么选择一个新的任务替代它，`wakeup_preempt_entity`判断新选择的任务替换第1步中选择任务是否会破坏公平性；

3).考虑`cfs_rq`之中有考虑接下来运行的任务，若运行这个任务不会破坏公平性则返回这个任务；

4).考虑`cfs_rq`之中指定了最后考虑运行的任务，即在其他条件都无法找到时考虑的任务，若运行这个任务不会破坏公平性则返回这个任务；

这个函数中公平性判断使用了`wakeup_preempt_entity`函数，这个函数的详细记录见`check_preempt_cfs.md`。

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

设置`t0`为尝试从调度域其他cpu中拉取任务的开始时间，调用`update_blocked_averages`函数更新运行队列之中平均负载统计，这个函数名之中`blocked`含义为`blocked load`，这个概念的含义见[Per-entity load tracking](https://lwn.net/Articles/531853/)，简而言之内核用`blocked load`指的是那些被阻塞任务的负载。从调度域其他cpu之中拉取任务对应的代码在`for_each_domain`这个循环之中，这个循环遍历当前cpu所在的调度域，对于每个调度域调用`update_next_balance`函数更新此调度域下次执行负载均衡的时间、调用`load_balance`函数尝试从调度域中其他cpu拉取一个任务、调用`update_newidle_cost`更新调度域负载均衡消耗的时间，若已经拉取到任务或者运行队列中已经有可运行的任务或者运行队列中有等待运行的任务直接退出调度域遍历过程。在循环过程中还会记录对正在遍历的调度域中调用`load_balance`函数进行负载均衡的时间，更新正在遍历的调度域的负载均衡所需时间，还会判断运行队列处于空闲状态时间(`avg_idle`)与尝试从其他cpu中拉取任务累计消耗时间(`curr_cost`)、负载均衡操作消耗的最长时间(`max_newidle_lb_cost`)，若前者小于后者则认为从其他的cpu中拉取任务是低效的、直接放弃未遍历到的调度域。

在后边详细记录上述流程中提到的`update_blocked_averages`、`update_newidle_cost`、`update_next_balance`函数，`load_balance`内容十分复杂需要在一个文档之中单独记录流程，其他的函数忽略。

#### 任务拉取结束的检查

这部分对应代码中调用`raw_spin_rq_lock`位置到标签`out`之间的代码，这部分代码更新当前运行队列之中负载均衡操作消耗的最长时间统计(`this_rq->max_idle_balance_cost`)，对两种特殊情况进行判断：若CFS运行队列之中有可运行的任务并且没能从其他的cpu中拉取任务，设置`pulled_task`为1表示已经有新的、可运行的任务了；若运行队列中可运行的任务大于CFS运行队列中可运行的任务，这意味着运行队列之中有任务关联了优先级更高的调度类，设置`pulled_task`为`-1`提醒调用者这种情况。

#### 标签`out`

若当前的运行队列设置的下次负载均衡运行(`this_rq->max_idle_balance_cost`)时间晚于负载均衡过程中计算出来的时间(`next_balance`)，则提前当前运行队列下次负载均衡运行时间至负载均衡过程中计算出来的时间。若已经拉取到任务或者当前运行队列之中有优先级更高的任务，运行队列即将退出空闲状态，重置`idle_stamp`为0；若没有拉取到任务则调用`nohz_newidle_balance`函数进行负载均衡相关的检查工作。在后边详细记录`nohz_newidle_balance`函数流程，其他的函数忽略。

### `update_blocked_averages`函数

```c
static void update_blocked_averages(int cpu)
{
	bool decayed = false, done = true;
	struct rq *rq = cpu_rq(cpu);
	struct rq_flags rf;

	rq_lock_irqsave(rq, &rf);
	update_blocked_load_tick(rq);
	update_rq_clock(rq);

	decayed |= __update_blocked_others(rq, &done);
	decayed |= __update_blocked_fair(rq, &done);

	update_blocked_load_status(rq, !done);
	if (decayed)
		cpufreq_update_util(rq, 0);
	rq_unlock_irqrestore(rq, &rf);
}
```

这个函数在持有运行队列锁并且禁用软中断的前提下进行的，主要流程如下：

1).调用`update_blocked_load_tick`函数更新运行队列`rq`中最近一次更新`blocked_load`的时间，`blocked_load`的含义见`newidle_balance`函数的详细流程说明；

2).调用`update_rq_lock`更新运行队列`rq`的时间相关的字段，在后边的更新平均负载等统计字段的时候需要依赖这个函数执行的结果；

3).调用`__udpate_blocked_others`函数更新运行队列`rq`中实时调度类、`deadline`调度类运行队列中被阻塞任务平均负载等指标，这个函数之中返回的`done`表示是否有阻塞的任务；

4).调用`__update_blocked_fair`函数更新运行队列`rq`中CFS调度类的平均负载指标，这个函数之中返回的`done`表示是否有阻塞的任务；

5).调用`update_blocked_load_status`函数更新运行队列`rq`之中是否有阻塞任务的标记，需要依赖`done`的值；

上述流程中涉及到好几个函数，其中`__update_blocked_fair`、`__update_blocked_others`函数的流程在后边详细记录，`update_rq_lock`函数流程详细记录见`task_wake_up.md`，其他的内容忽略。

### `__update_blocked_others`函数

```c
static bool __update_blocked_others(struct rq *rq, bool *done)
{
	const struct sched_class *curr_class;
	u64 now = rq_clock_pelt(rq);
	unsigned long thermal_pressure;
	bool decayed;

	/*
	 * update_load_avg() can call cpufreq_update_util(). Make sure that RT,
	 * DL and IRQ signals have been updated before updating CFS.
	 */
	curr_class = rq->curr->sched_class;

	thermal_pressure = arch_scale_thermal_pressure(cpu_of(rq));

	decayed = update_rt_rq_load_avg(now, rq, curr_class == &rt_sched_class) |
		  update_dl_rq_load_avg(now, rq, curr_class == &dl_sched_class) |
		  update_thermal_load_avg(rq_clock_thermal(rq), rq, thermal_pressure) |
		  update_irq_load_avg(rq, 0);

	if (others_have_blocked(rq))
		*done = false;

	return decayed;
}
```

`rq_clock_pelt`函数给出现在的时间(移除了运行队列处于空闲的时间)保存在`now`之中，`curr_class`为运行队列`rq`中正在运行的任务关联的调度类，`update_rt_rq_load_avg`更新实时调度类对应运行队列的平均负载等统计指标，`update_dl_rq_load_avg`更新`deadline`调度类对应运行队列的平均负载统计等指标，调用这两个函数的时候第三个参数为是否有关联此调度类的任务正在运行。`others_have_blocked`函数给出运行队列`rq`中这两个调度类的队列之中是否有阻塞的任务，如果有阻塞的任务通过`done`标记此种情况。接下来记录这个函数中调用的`rq_clock_pelt`、`update_rt_rq_load_avg`、`update_dl_rq_load_avg`、`others_have_blocked`函数的详细流程，其他的内容忽略。

### `rq_clock_pelt`函数

```c
static inline u64 rq_clock_pelt(struct rq *rq)
{
	lockdep_assert_rq_held(rq);
	assert_clock_updated(rq);

	return rq->clock_pelt - rq->lost_idle_time;
}


```

`clock_pelt`为计算任务负载专用的当前时间，这个时间的计算见`task_wake_up.md`之中记录的的`update_rq_clock_pelt`函数，`lost_idle_time`为累计的空闲时间，这个函数返回二者的差值用于平均负载的计算。

### `update_rt_rq_load_avg`函数

```c
int update_rt_rq_load_avg(u64 now, struct rq *rq, int running)
{
	if (___update_load_sum(now, &rq->avg_rt,
				running,
				running,
				running)) {

		___update_load_avg(&rq->avg_rt, 1);
		trace_pelt_rt_tp(rq);
		return 1;
	}

	return 0;
}
```

忽略`trace_pelt_rt_tp`函数的内容，这个函数更新实时调度类运行队列的平均负载等统计指标，`running`为运行队列之中正在执行的任务是否关联了试试调度类，计算过程调用的`___upate_load_sum`、`___update_load_avg`两个函数流程在`enqueue_task_cfs.md`之中有详细记录，需要住的是当`running`为0时`___update_load_sum`函数仅仅对之前的`util_sum`、`load_sum`、`runnable_sum`指标进行衰减而不考虑最近的运行时间，这三个`sum`指标变化之后需要更新对应的`avg`指标。

### `update_dl_rq_load_avg`函数

```c
int update_dl_rq_load_avg(u64 now, struct rq *rq, int running)
{
	if (___update_load_sum(now, &rq->avg_dl,
				running,
				running,
				running)) {

		___update_load_avg(&rq->avg_dl, 1);
		trace_pelt_dl_tp(rq);
		return 1;
	}

	return 0;
}
```

这个函数的逻辑与`update_rt_rq_load_avg`函数一致，只不过这个函数更新的是`deadline`调度类的统计指标。

### `__update_blocked_fair`函数

```c
static bool __update_blocked_fair(struct rq *rq, bool *done)
{
	struct cfs_rq *cfs_rq, *pos;
	bool decayed = false;
	int cpu = cpu_of(rq);

	/*
	 * Iterates the task_group tree in a bottom up fashion, see
	 * list_add_leaf_cfs_rq() for details.
	 */
	for_each_leaf_cfs_rq_safe(rq, cfs_rq, pos) {
		struct sched_entity *se;

		if (update_cfs_rq_load_avg(cfs_rq_clock_pelt(cfs_rq), cfs_rq)) {
			update_tg_load_avg(cfs_rq);

			if (cfs_rq->nr_running == 0)
				update_idle_cfs_rq_clock_pelt(cfs_rq);

			if (cfs_rq == &rq->cfs)
				decayed = true;
		}

		/* Propagate pending load changes to the parent, if any: */
		se = cfs_rq->tg->se[cpu];
		if (se && !skip_blocked_update(se))
			update_load_avg(cfs_rq_of(se), se, UPDATE_TG);

		/*
		 * There can be a lot of idle CPU cgroups.  Don't let fully
		 * decayed cfs_rqs linger on the list.
		 */
		if (cfs_rq_is_decayed(cfs_rq))
			list_del_leaf_cfs_rq(cfs_rq);

		/* Don't need periodic decay once load/util_avg are null */
		if (cfs_rq_has_blocked(cfs_rq))
			*done = false;
	}

	return decayed;
}
```

这个函数更新运行队列`rq`之中的所有`leaf cfs_rq`的平均负载等统计指标，`leaf cfs_rq`的含义在`enqueue_task_cfs.md`文件中的`list_add_leaf_cfs_rq`函数流程记录之中可以找到，通过`leaf cfs_rq`可以找到运行在当前cpu上的所有包含任务的CFS运行队列。使用`for_each_leaf_cfs_rq_safe`来遍历运行队列`rq`之中每个`leaf cfs_rq`，对于遍历到的每个CFS运行队列执行：

1).更新CFS运行队列的平均负载等统计指标，更新过程在`update_cfs_rq_load_avg`函数中实现，这个函数的详细流程记录见`enqueue_task_cfs.md`；

2).若更新CFS运行队列平均负载等统计指标的过程中运行过指数衰减计算：

2.1).`cfs_rq`为任务组在当前cpu上的CFS运行队列，将`cfs_rq`中平均负载等统计指标传播到任务组之中，更新过程在`update_tg_load_avg`函数中实现，这个函数的详细流程记录见`enqueue_task_cfs.md`；

2.2).若`cfs_rq`为当前运行队列`rq`中的CFS运行队列，将`delay`设置为`true`标记运行队列出现过指数衰减计算；

3).任务组的平均负载等统计指标已经更新，需要向调度层级中的上级传递统计指标的变化，代码之中`se`之中保存了最新的指标值、`cfs_rq_of(se)`为`se`的上级，调用`update_load_avg`函数将`se`之中保存的统计指标传播到上级之中，这个函数详细流程记录见`enqueue_task_cfs.md`；

4).若CFS运行队列`cfs_rq`之中没有可运行的任务，从`leaf cfs_rq`之中将此CFS运行队列移除，`cfs_rq_is_decayed`函数用于判断是否CFS运行队列之中是否有可运行的任务、`list_del_leaf_cfs_rq`将`cfs_rq`从`leaf cfs_rq`之中移除，这两个函数的内容忽略；

5).若`cfs_rq`之中依然还有阻塞的任务，通过`done`通过调用者这个情况，这个判断是在`cfs_rq_has_blocked`函数之中完成的，这个函数通过判断`load_avg`、`util_avg`两个指标是否为0来确定是否有阻塞的任务。考虑到这个函数是在未找到接下来运行的任务时调用的，这两个指标为0时意味着CFS运行队列中没有可运行任务，若这两个指标不为0则意味着CFS运行队列中有被阻塞的任务。

### `update_newidle_cost`函数

```c
static inline bool update_newidle_cost(struct sched_domain *sd, u64 cost)
{
	if (cost > sd->max_newidle_lb_cost) {
		/*
		 * Track max cost of a domain to make sure to not delay the
		 * next wakeup on the CPU.
		 */
		sd->max_newidle_lb_cost = cost;
		sd->last_decay_max_lb_cost = jiffies;
	} else if (time_after(jiffies, sd->last_decay_max_lb_cost + HZ)) {
		/*
		 * Decay the newidle max times by ~1% per second to ensure that
		 * it is not outdated and the current max cost is actually
		 * shorter.
		 */
		sd->max_newidle_lb_cost = (sd->max_newidle_lb_cost * 253) / 256;
		sd->last_decay_max_lb_cost = jiffies;

		return true;
	}

	return false;
}
```

这个函数更新调度域`sd`在运行队列空闲时运行负载均衡消耗的最长时间，要考虑本次运行负载均衡消耗的时间和`sd`之中保存的最长时间相对大小：若前者大于后者直接将`sd`之中保存的历史最长时间更新为前者；若前者小于后者并且现在与上次衰减时间差距超过1s，则将调度域`sd`的最长时间衰减1%，防止因为最长时间过大抑制负载均衡的执行。

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

这个函数基于调度域默认的负载均衡时间间隔(`balance_interval`)，结合当前cpu是否处于忙碌状态，计算出一个负载均衡时间间隔。若当前cpu处于忙碌状态，将调度域默认的负载均衡时间间隔放大，减少负载均衡的运行次数。在当前cpu处于忙碌状态的时候，为了防止不同层次的调度域执行负载均衡的时间成为倍数关系造成的不同层次调度域连续运行负载均衡操作，递减调度域`sd`的时间间隔。最后为调度域`sd`运行负载均衡的时间间隔加一个上限、下限限制为最终的结果并返回。

### `nohz_newidle_balance`函数

```c
static void nohz_newidle_balance(struct rq *this_rq)
{
	int this_cpu = this_rq->cpu;

	/*
	 * This CPU doesn't want to be disturbed by scheduler
	 * housekeeping
	 */
	if (!housekeeping_cpu(this_cpu, HK_TYPE_SCHED))
		return;

	/* Will wake up very soon. No time for doing anything else*/
	if (this_rq->avg_idle < sysctl_sched_migration_cost)
		return;

	/* Don't need to update blocked load of idle CPUs*/
	if (!READ_ONCE(nohz.has_blocked) ||
	    time_before(jiffies, READ_ONCE(nohz.next_blocked)))
		return;

	/*
	 * Set the need to trigger ILB in order to update blocked load
	 * before entering idle state.
	 */
	atomic_or(NOHZ_NEWILB_KICK, nohz_flags(this_cpu));
}
```

这个函数为当前的cpu设置`NOHZ_NEWILB_KICK`标志，这个标志的含义是当cpu进入到空闲状态时更新运行队列之中`blocked_load`，`blocked load`的含义见`newidle_balance`函数详细流程记录。不是所有的情况都需要都需要设置这个标记，当运行队列`this_rq`空闲的时间过短以至于小于任务从其他cpu之中迁移过来的时间、现在的时间小于下一次`blocked load`更新的时间时，不需要添加这个标记。第一种情况可以理解为现在当前cpu的空闲时间还很短，可能很快就会有任务从其他的cpu之中迁移过来，如果有任务迁移过来了当前cpu没有时间进行其他的计算了，所以需要继续等。

### `update_idle_rq_clock_pelt`函数

```c
/*
 * When rq becomes idle, we have to check if it has lost idle time
 * because it was fully busy. A rq is fully used when the /Sum util_sum
 * is greater or equal to:
 * (LOAD_AVG_MAX - 1024 + rq->cfs.avg.period_contrib) << SCHED_CAPACITY_SHIFT;
 * For optimization and computing rounding purpose, we don't take into account
 * the position in the current window (period_contrib) and we use the higher
 * bound of util_sum to decide.
 */
static inline void update_idle_rq_clock_pelt(struct rq *rq)
{
	u32 divider = ((LOAD_AVG_MAX - 1024) << SCHED_CAPACITY_SHIFT) - LOAD_AVG_MAX;
	u32 util_sum = rq->cfs.avg.util_sum;
	util_sum += rq->avg_rt.util_sum;
	util_sum += rq->avg_dl.util_sum;

	/*
	 * Reflecting stolen time makes sense only if the idle
	 * phase would be present at max capacity. As soon as the
	 * utilization of a rq has reached the maximum value, it is
	 * considered as an always running rq without idle time to
	 * steal. This potential idle time is considered as lost in
	 * this case. We keep track of this lost idle time compare to
	 * rq's clock_task.
	 */
	if (util_sum >= divider)
		rq->lost_idle_time += rq_clock_task(rq) - rq->clock_pelt;

	_update_idle_rq_clock_pelt(rq);
}
```

这个函数中`divider`为`util_sum`的几何级数之和的极限值，`util_sum`为运行队列`rq`之中正在运行任务的几何级数之和，若`util_sum`大于其理论上的最大值则意味着运行队列`rq`在超负荷运行，这就意味着理论上应该有一段时间运行队列`rq`应该处于空闲状态，`struct rq`之中的`lost_idle_time`字段保存理论上应该处于空闲状态但实际上却没有的时间累计值。这个时间由`rq`之中的当前时间与`PELT`统计计算使用的当前时间(`clock_pelt`)之差得到，`PELT`统计计算使用的当前时间不包含运行队列处于空闲状态的时间，所以把二者的差值当作`lost_idle_time`的累加值。`_update_idle_rq_clock_pelt`函数更新`rq`之中其他的时间字段，接下来详细记录这个函数的流程。

### `_update_idle_rq_clock_pelt`函数

```c
/* The rq is idle, we can sync to clock_task */
static inline void _update_idle_rq_clock_pelt(struct rq *rq)
{
	rq->clock_pelt  = rq_clock_task(rq);

	u64_u32_store(rq->clock_idle, rq_clock(rq));
	/* Paired with smp_rmb in migrate_se_pelt_lag() */
	smp_wmb();
	u64_u32_store(rq->clock_pelt_idle, rq_clock_pelt(rq));
}
```

这个函数将`PELT`统计计算使用的当前时间(`clock_pelt`)设置为运行队列之中的追踪任务时间使用的当前时间(`rq_clock_task(rq)`，这个时间忽略系统中断等占用的时间)、记录运行队列进入空闲状态的时间(`clock_idle`)、记录`PELT`统计计算使用的进入空闲状态的时间。

