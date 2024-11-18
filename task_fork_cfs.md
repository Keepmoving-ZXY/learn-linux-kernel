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
