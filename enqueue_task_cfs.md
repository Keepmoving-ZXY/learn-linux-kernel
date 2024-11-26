本文档记录`enqueue_task_fair`函数的流程，这个函数将调度实体`se`插入到cfs_rq的红黑树之中，这个过程中还会更新一些统计字段。

`enqueue_task_fair`函数

```c
/*
 * The enqueue_task method is called before nr_running is
 * increased. Here we update the fair scheduling stats and
 * then put the task into the rbtree:
 */
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
    struct cfs_rq *cfs_rq;
    struct sched_entity *se = &p->se;
    int idle_h_nr_running = task_has_idle_policy(p);
    int task_new = !(flags & ENQUEUE_WAKEUP);

    /*
     * The code below (indirectly) updates schedutil which looks at
     * the cfs_rq utilization to select a frequency.
     * Let's add the task's estimated utilization to the cfs_rq's
     * estimated utilization, before we update schedutil.
     */
    util_est_enqueue(&rq->cfs, p);

    /*
     * If in_iowait is set, the code below may not trigger any cpufreq
     * utilization updates, so do it here explicitly with the IOWAIT flag
     * passed.
     */
    if (p->in_iowait)
        cpufreq_update_util(rq, SCHED_CPUFREQ_IOWAIT);

    for_each_sched_entity(se) {
        if (se->on_rq)
            break;
        cfs_rq = cfs_rq_of(se);
        enqueue_entity(cfs_rq, se, flags);

        cfs_rq->h_nr_running++;
        cfs_rq->idle_h_nr_running += idle_h_nr_running;

        if (cfs_rq_is_idle(cfs_rq))
            idle_h_nr_running = 1;

        /* end evaluation on encountering a throttled cfs_rq */
        if (cfs_rq_throttled(cfs_rq))
            goto enqueue_throttle;

        flags = ENQUEUE_WAKEUP;
    }

    for_each_sched_entity(se) {
        cfs_rq = cfs_rq_of(se);

        update_load_avg(cfs_rq, se, UPDATE_TG);
        se_update_runnable(se);
        update_cfs_group(se);

        cfs_rq->h_nr_running++;
        cfs_rq->idle_h_nr_running += idle_h_nr_running;

        if (cfs_rq_is_idle(cfs_rq))
            idle_h_nr_running = 1;

        /* end evaluation on encountering a throttled cfs_rq */
        if (cfs_rq_throttled(cfs_rq))
            goto enqueue_throttle;
    }

    /* At this point se is NULL and we are at root level*/
    add_nr_running(rq, 1);

    if (!task_new)
        check_update_overutilized_status(rq);

enqueue_throttle:
    assert_list_leaf_cfs_rq(rq);

    hrtick_update(rq);
}
```

这里先关注两个循环（`for_each_sched_entity`）中的内容，这两个循环涉及到内核的任务组调度功能，关于这个功能的详细介绍见[task_group](https://blogs.oracle.com/linux/post/cfs-group-scheduling)，从介绍中可以看到在任务组调度功能之中，调度实体被划分到了多个层级，有的层次中调度实体对应一个任务组、有的层级中的调度实体对应一个任务，这两个`for`循环都是对于任务`p`对应的调度实体以及上级每个层级与之关联的调度实体而言的。第一个`for`循环用于将`p`对应的实体以及每个层级中与这个实体关联的实体放入到rq之中，第二个`for`循环用于更新平均负载、任务组的可运行任务相关统计字段、重新计算任务组实体的权重。两个循环之后调用`add_nr_running`更新cfs_rq中正在运行任务的计数并进行超载检查，最后调用`hrtick_update`更新高精度定时器。

这里仅仅是简单整理一下这个函数的整体流程，后边会具体记录这其中许多函数的流程，被忽略的函数除外。这个函数中还隐藏着一个有意思的处理方式，这种涉及到了两个字段的更新（`h_nr_running`、`idle_h_nr_running`），这两个字段为cfs_rq的统计字段，代表了在当前的调度层级以及下层之中正在运行的实体、优先级极低（任务的调度策略被指定为了`SCHED_IDLE`）的任务。先以`h_nr_running`为例，当任务`p`对应的实体以及调度层级中与之关联的实体都放入到cfs_rq之中时，每个层级之中新增的正在运行的实体为2，但是在第一个`for`循环之中只为每一层的cfs_rq中这个字段递增，这个看起来很奇怪，因为这里需要2次递增，直到看到第二个`for`循环之中的代码才知道第二次递增在这个循环之中。`idle_h_nr_running`的数值更新逻辑类似，也是分到了两个`for`循环之中完成数值递增，只不过在某个层级调度实体对应的任务组为`SCHED_IDLE`组时才开始更新。
