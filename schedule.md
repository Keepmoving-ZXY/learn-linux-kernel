## 简介

在linux任务调度过程中，当为某个CPU中rq上正在运行的任务设置了`TIF_NEED_RESCHED`标志，意味着这个任务即将被其他任务抢占，即另外一个任务替代这个任务占用CPU资源，完成这个动作的函数是`schedule`函数。在`try_to_wake_up`中会将rq中正在执行的任务加上`TIF_NEED_RESCHED`标志，另外一个流程是定时器由驱动的rq的时间更新流程，在这个流程中可能会为rq中正在运行的任务添加`TIF_NEED_RESCHED`标记。在记录`schedule`函数之前，先记录高精度定时器如何影响任务调度过程的。

## 代码流程

### 1.高精度定时器

本节关注高精度定时器对调度流程的影响，大致的流程是使用`hrtick_rq_init`函数初始化rq中高精度定时器，然后调用`__hrtick_restart`启用rq中的高精度定时器，当定时器触发时调用`hrtick`函数更新rq的时间相关字段。

#### `hrtick_rq_init`函数

```c
static void hrtick_rq_init(struct rq *rq)
{
#ifdef CONFIG_SMP
    INIT_CSD(&rq->hrtick_csd, __hrtick_start, rq);
#endif
    hrtimer_init(&rq->hrtick_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL_HARD);
    rq->hrtick_timer.function = hrtick;
}
```

这个函数初始化rq中高精度定时器，这个定时器使用单调递增时钟、使用相对时间表示定时器触发时刻、在硬件中断上下文中运行，定时器触发是执行的函数是`hrtick`。同时还会初始化`call single data`相关的字段，`call single data`的用途在接下来的函数中说明。

#### `hrtick_start`函数

```c
/*
 * Called to set the hrtick timer state.
 *
 * called with rq->lock held and irqs disabled
 */
void hrtick_start(struct rq *rq, u64 delay)
{
    struct hrtimer *timer = &rq->hrtick_timer;
    s64 delta;

    /*
     * Don't schedule slices shorter than 10000ns, that just
     * doesn't make sense and can cause timer DoS.
     */
    delta = max_t(s64, delay, 10000LL);
    rq->hrtick_time = ktime_add_ns(timer->base->get_time(), delta);

    if (rq == this_rq())
        __hrtick_restart(rq);
    else
        smp_call_function_single_async(cpu_of(rq), &rq->hrtick_csd);
}
```

这个函数设置触发时间为当前时间加上至少10us的延迟时间，如果定时器所属的rq与当前运行CPU的rq相同，则调用`__krtick_restart`函数启动定时器，反之则通知rq所属cpu执行`__hrtick_start`函数启动定时器（`smp_call_function_single_async`）。

#### `__hrtick_restart`函数

```c
static void __hrtick_restart(struct rq *rq)
{
    struct hrtimer *timer = &rq->hrtick_timer;
    ktime_t time = rq->hrtick_time;

    hrtimer_start(timer, time, HRTIMER_MODE_ABS_PINNED_HARD);
}
```

这个函数调用`hrtimer_start`启动一个高精度定时器，`HRTIMER_MODE_ABS_PINNED_HARD`表示这个高精度定时器触发时间不是相对时间、这个定时器在指定CPU中运行、这个定时器在硬中断的上下文中执行。

#### `__hrtick_start`函数

```c
/*
 * called from hardirq (IPI) context
 */
static void __hrtick_start(void *arg)
{
    struct rq *rq = arg;
    struct rq_flags rf;

    rq_lock(rq, &rf);
    __hrtick_restart(rq);
    rq_unlock(rq, &rf);
}
```

这个函数在硬件中断上下文中执行（`IPI`中断）中执行，在持有rq锁的前提下在rq所在的CPU中调用`__hrtick_restart`函数启动高精度定时器。

#### `hrtick`函数

```c
/*
 * High-resolution timer tick.
 * Runs from hardirq context with interrupts disabled.
 */
static enum hrtimer_restart hrtick(struct hrtimer *timer)
{
    struct rq *rq = container_of(timer, struct rq, hrtick_timer);
    struct rq_flags rf;

    WARN_ON_ONCE(cpu_of(rq) != smp_processor_id());

    rq_lock(rq, &rf);
    update_rq_clock(rq);
    rq->curr->sched_class->task_tick(rq, rq->curr, 1);
    rq_unlock(rq, &rf);

    return HRTIMER_NORESTART;
}
```

这个函数在持有rq锁的前提下，更新rq的时间相关字段、调用rq所在CPU中正在运行的任务使用的调度类的`task_tick`方法。注意到这个函数返回的是`HRTIMER_NORESTART`，意味着这个定时器不会自动启用，在调度类中才会继续启动这个定时器，例如CFS的`enqueue_task_fair`函数。在执行调度类的`task_tick`函数过程中会为当前正在运行的任务设置`TIF_NEED_RESCHED`标记，然后内核其他部分代码执行过程中调用`schedule`函数触发一次进程切换。

### 2.任务切换

#### `__switch_to_asm`函数

```c
/*
 * %rdi: prev task
 * %rsi: next task
 */
.pushsection .text, "ax"
SYM_FUNC_START(__switch_to_asm)
    /*
     * Save callee-saved registers
     * This must match the order in inactive_task_frame
     */
    pushq    %rbp
    pushq    %rbx
    pushq    %r12
    pushq    %r13
    pushq    %r14
    pushq    %r15

    /* switch stack */
    movq    %rsp, TASK_threadsp(%rdi)
    movq    TASK_threadsp(%rsi), %rsp

#ifdef CONFIG_STACKPROTECTOR
    movq    TASK_stack_canary(%rsi), %rbx
    movq    %rbx, PER_CPU_VAR(fixed_percpu_data) + stack_canary_offset
#endif

    /*
     * When switching from a shallower to a deeper call stack
     * the RSB may either underflow or use entries populated
     * with userspace addresses. On CPUs where those concerns
     * exist, overwrite the RSB with entries which capture
     * speculative execution to prevent attack.
     */
    FILL_RETURN_BUFFER %r12, RSB_CLEAR_LOOPS, X86_FEATURE_RSB_CTXSW

    /* restore callee-saved registers */
    popq    %r15
    popq    %r14
    popq    %r13
    popq    %r12
    popq    %rbx
    popq    %rbp

    jmp    __switch_to
SYM_FUNC_END(__switch_to_asm)
.popsection
```

这个函数执行任务抢占过程中非常关键的一个函数，这个函数中有一些微妙的细节，结合一个例子来理解这些微妙的细节。假设`switch_to`函数的`prev`为任务a、`next`为任务b、任务a此时运行在CPU 3之中、在任务c运行时抢占了任务b的CPU资源，那么在调用`__switch_to_asm`函数时是在CPU 3上执行任务a的过程中，`pushq`使用的栈空间为任务a的栈空间。

先关注任务c抢占任务b时的一些操作，这些操作有助于理解任务b抢占任务a过程中的一个细节，即执行`popq`指令的时候使用的栈究竟是谁的栈。此时这个函数执行的时候`prev`为任务b、`next`为任务c、在任务b所在的CPU中运行此函数。此函数首先将`rbp`、`rbx`、`r12-r15`寄存器内容放入到任务b的栈之中，然后使用`movq %rsp, TASK_THREADSP(%rdi)`将`rsp`寄存器的值放入到任务b的`task_struct->thread_struct->sp`字段之中、使用`movq TASK_threadsp(%rsi), %rsp`将任务c的栈地址写入到`rsp`寄存器中，后续代码开始使用c的栈空间而非b的栈空间。接下来的`popq`指令使用的栈是任务c的栈空间而非任务b的栈空间，恢复寄存器的值（这些值是在任务c被其他任务抢占的时候执行这个函数中前几个`pushq`指令的时候入栈的）。从这里看到这个函数中`pushq`使用的栈与`popq`使用的栈不同同一个栈，`pushq`使用的栈是被抢占任务的栈，而`popq`使用的栈是即将恢复执行任务的栈。接下来`__switch_to`函数返回之后任务b失去CPU资源，导致任务b处于暂停状态，任务c开始执行。

接下来关注任务b抢占任务a的情况，此时`__switch_to_asm`函数的`prev`为任务a、`next`为任务b，此时这个函数使用任务a的栈空间。

函数开始的四个`pushq`指令将6个寄存器的值保存到任务a的栈之中，然后`rsp`寄存器的值至任务a的`task_struct->thread_struct->sp`之中、将`rsp`寄存器的值设置为任务b的栈空间，后续的代码开始使用任务b的栈空间。

`CONFIG_STACKPROTECTOR`启用的代码与一种栈溢出检测技术相关，即在栈空间的结束后的位置放置一个叫做stack canary的值，通过检查栈空间结束后第一个位置的值是否与这个值相同来判断是否存在栈溢出现象，这个宏启用的代码使用任务b的stack canary替换任务a的stack canary。

`FILL_RETURN_BUFFER`指令涉及到现代处理器的RSB（Return stack buffer）特性，RSB中存放了进程执行函数调用过程中每次调用返回之后可能执行的下一条指令的地址，这个指令用于在任务切换时清空被抢占任务的地址信息（使用特定的值替代被抢占进程的地址信息），以防止RSB中存储其他进程地址而带来的潜在安全性问题。

接下来执行的6条`popq`指令从任务b的栈中弹出6个寄存器的值，这些值并不是在任务b抢占任务a时入栈的值，而是在任务c抢占任务b的时候入栈的值。在任务c抢占任务b时将这几个寄存器的值压入栈中，随后任务b失去CPU资源暂停执行，随着时间的流逝任务b得到在CPU 3资源可以继续运行，此时CPU 3使用的栈空间为任务b的栈空间，所以此时是从b的栈空间中出栈任务b被任务c抢占时压入的寄存器的值。

最后执行`__switch_to`函数，当函数返回之后任务a已经失去CPU资源暂停运行，任务b开始恢复运行。
