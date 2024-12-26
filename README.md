这个仓库中记录我学习Linux 6.1.116版本内核源码的点点滴滴，有的内容是函数流程记录、有的流程是机制记录。

1.`task_wake_up.md`中的内容是`try_to_wake_up`函数的流程记录；

2.`schedule.md`中的内容是任务调度相关的过程记录，主要是`schedule`函数流程记录，这个函数是内核调度机制中进行任务切换的主要函数；

3.`task_fork_cfs.md`中的内容是`task_fork_fair`函数的流程记录，这个函数是内核调度机制中的`task_fork`方法的实现；

4.`enqueue_task_fair.md`中的内容是`enqueue_task_fair`函数的流程记录，这个函数是内核调度机制中`enqueue_task`方法的CFS实现；

5.`check_preempt_cfs.md`中的内容是`check_preempt_wakeup`函数的流程记录，这个函数是内核调度机制中`check_preempt_curr`方法的CFS实现；

6.`dequeue_task_cfs.md`中的内容是`dequeue_task_fair`函数的流程记录，这个函数是内核调度机制中`dequeue_task`方法的CFS实现；

7.`task_tick_cfs.md`中的内容是`task_tick_fair`函数的流程记录，这个函数是内核调度机制中`task_tick_fair`方法的CFS实现；

8.`pick_next_task_cfs.md`中的内容是`__pick_next_task_fair`函数的流程记录，这个函数是内核调度机制中`pick_next_task`方法的CFS实现；
