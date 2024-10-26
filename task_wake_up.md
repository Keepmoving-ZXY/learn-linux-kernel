## 简介

task唤醒涉及到的函数是`try_to_wake_up`，函数的声明为`statc int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)`，函数的作用是当指定的task `p`的状态包含`state`指定状态时尝试将task唤醒，注意这里的task可能是一个进程也可能是一个线程（即轻量级进程）。`wake_up_process`函数中会调用这个函数，`wake_up_process`函数在内核中许多地方会使用，在GPU驱动中就可以看到这个函数的身影。

## 函数流程

这个函数首先禁止任务抢占：

```
    unsigned long flags;
    int cpu, success = 0;

    preempt_disable();
```

### 情况1：被唤醒的task与当前正在执行的task为同一个task

若需要被唤醒的task `p`与当前正在执行的task是同一个task，则判断任务状态是否匹配，当任务状态匹配时将task `p`的状态配置为正在运行中，随后代码跳转到`out`标识符，代码如下：

```
    if (p == current) {
        /*
         * We're waking current, this means 'p->on_rq' and 'task_cpu(p)
         * == smp_processor_id()'. Together this means we can special
         * case the whole 'p->on_rq && ttwu_runnable()' case below
         * without taking any locks.
         *
         * In particular:
         *  - we rely on Program-Order guarantees for all the ordering,
         *  - we're serialized against set_special_state() by virtue of
         *    it disabling IRQs (this allows not taking ->pi_lock).
         */
        if (!ttwu_state_match(p, state, &success))
            goto out;

        trace_sched_waking(p);
        WRITE_ONCE(p->__state, TASK_RUNNING);
        trace_sched_wakeup(p);
        goto out;
    }
```

上述代码中trace_sched_xxx函数实现暂时忽略，将重点放在`ttwu_state_match`的函数实现中。

#### `ttwu_state_match`函数

任务状态的判断实现在这个函数中，实现如下：

```
static __always_inline
bool ttwu_state_match(struct task_struct *p, unsigned int state, int *success)
{
    if (IS_ENABLED(CONFIG_DEBUG_PREEMPT)) {
        WARN_ON_ONCE((state & TASK_RTLOCK_WAIT) &&
                 state != TASK_RTLOCK_WAIT);
    }

    if (READ_ONCE(p->__state) & state) {
        *success = 1;
        return true;
    }

#ifdef CONFIG_PREEMPT_RT
    /*
     * Saved state preserves the task state across blocking on
     * an RT lock.  If the state matches, set p::saved_state to
     * TASK_RUNNING, but do not wake the task because it waits
     * for a lock wakeup. Also indicate success because from
     * the regular waker's point of view this has succeeded.
     *
     * After acquiring the lock the task will restore p::__state
     * from p::saved_state which ensures that the regular
     * wakeup is not lost. The restore will also set
     * p::saved_state to TASK_RUNNING so any further tests will
     * not result in false positives vs. @success
     */
    if (p->saved_state & state) {
        p->saved_state = TASK_RUNNING;
        *success = 1;
    }
#endif
    return false;
```

 仅仅关注这个函数中第2个`if`代码中的内容，`READ_ONCE`防止编译器对从内存中读取task状态这个动作进行优化，读取到状态之后与指定的状态进行对比，若状态匹配则返回成功。
