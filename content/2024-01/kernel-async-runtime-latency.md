+++
title = "Kernel Async Runtime Latency"
date = "2024-01-20"
[extra]
add_toc = true
+++

Linux kernel has many ways to asynchronously execute some code,
e.g. `workqueue`, `irq_work`, `task_work`, `kthread`


## Linux's interrupt and scheduling
Like any other OS, the scheduler is driven mainly by timer interrupt
(other interrupt might also trigger scheduling).
After the interrupt handler finishes and before returning to userspace,
the scheduler will check if it is a good time to switch to a different task.
This check is performed in `irqentry_exit()`.
And finally `switch_to()` will be called to run another task if the scheduler decides so.

Why we need to know this is that
this is exactally when many kernel asynchronous runtimes choose to run the queued iterms,
including `irq_work` and `task_work`.


<details><summary>code</summary>

```c
// DEFINE_IDTENTRY_*() will include irqentry_exit() to all interrupt handler
// e.g. DEFINE_IDTENTRY_SYSVEC(sysvec_apic_timer_interrupt)
=> irqentry_exit()
=> irqentry_exit_to_user_mode()
=> exit_to_user_mode_prepare()
=> exit_to_user_mode_loop()
=> schedule()
=> __schedule()
=> context_switch()
=> switch_to()
```
</details>


## irq_work
As the name suggests, `irq_work` is used for run things in the interrupt context.
Items on the list are executed in the timer interrupt handler.
They are run with interrupt disabled (see `tick_sched_timer()`).

`irq_work`s are maintained in a per-CPU single linked list.

<details><summary>call stack</summary>

```c
=> hrtimer_interrupt()
=> __hrtimer_run_queues()
=> __run_hrtimer() // setup in tick_setup_sched_timer()
=> tick_sched_timer()
=> tick_sched_handle()
=> update_process_times()
=> irq_work_tick()
=> irq_work_run_list()
```

```sh
$ sudo bpftrace -e 'kfunc:update_process_times { @[kstack] = count(); }'
// ...
@[
    bpf_prog_6deef7357e7b4530_sd_fw_ingress+26123
    bpf_prog_6deef7357e7b4530_sd_fw_ingress+26123
    bpf_trampoline_6442556272+71
    update_process_times+9
    tick_sched_timer+191
    __hrtimer_run_queues+347
    hrtimer_interrupt+244
    __sysvec_apic_timer_interrupt+103
    sysvec_apic_timer_interrupt+54
    asm_sysvec_apic_timer_interrupt+26
]: 2515
```
</details>

<details><summary>Useful APIs</summary>

```c
#include <linux/irq_work.h>
/*
 * An entry can be in one of four states:
 *
 * free	     NULL, 0 -> {claimed}       : free to be used
 * claimed   NULL, 3 -> {pending}       : claimed to be enqueued
 * pending   next, 3 -> {busy}          : queued, pending callback
 * busy      NULL, 2 -> {free, claimed} : callback in progress, can be claimed
 */
struct irq_work {
	void (*func)(struct irq_work *);
    // ...
};
static inline
void init_irq_work(struct irq_work *work, void (*func)(struct irq_work *));
bool irq_work_queue(struct irq_work *work);
bool irq_work_queue_on(struct irq_work *work, int cpu);
static inline bool irq_work_is_pending(struct irq_work *work);
static inline bool irq_work_is_busy(struct irq_work *work);
```
</details>


## task_work
`task_work` is very similar, but they are mainly used for managing per-thread states.
For an example, NUMA migration (see `task_numa_work()`) is executed using `task_work`.
Queued `task_work`s are run via `task_work_run()` right before returning to userspace
from syscalls or interrupt handlers.
They are run with irq enabled.

Just like `irq_work`, `task_work`s are also maintained in a single linked list.
However, they are contained in every `task_struct`
via an ad-hoc `cmpxcgh()` based linked list
instead of the generic `llist` used by `irq_work`.


<details><summary>call stack</summary>

```c
// interrupt handler return in DEFINE_IDTENTRY_SYSVEC(func)
=> irqentry_exit()
=> irqentry_exit_to_user_mode()
=> exit_to_user_mode_prepare()
=> exit_to_user_mode_loop()
=> resume_user_mode_work()
=> task_work_run()
```

```c
=> do_syscall_64()
=> syscall_exit_to_user_mode()
=> __syscall_exit_to_user_mode_work()
=> exit_to_user_mode_prepare()
=> exit_to_user_mode_loop()
=> resume_user_mode_work()
=> task_work_run()
```
</details>

<details><summary>Useful APIs</summary>

```c
#include <linux/task_work.h>

struct callback_head {
	struct callback_head *next;
	void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));

typedef void (*task_work_func_t)(struct callback_head *);
static inline void
init_task_work(struct callback_head *twork, task_work_func_t func);
int task_work_add(struct task_struct *task, struct callback_head *twork,
			enum task_work_notify_mode mode);
```
</details>

## kthread
`kthread`s are very similar to userspace threads.
They are useful for running long running things.
Thread creation, running and termination are all controlled by the users themselves.

<details><summary>Useful APIs</summary>

```c
#include <linux/kthread.h>

// kthread_create - create a kthread on the current node
#define kthread_create(threadfn, data, namefmt, arg...) // ...
struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
					   void *data,
					   int node,
					   const char namefmt[], ...);
struct task_struct *kthread_create_on_cpu(int (*threadfn)(void *data),
					  void *data,
					  unsigned int cpu,
					  const char *namefmt);

// wake_up_process - Wake up a specific process
int wake_up_process(struct task_struct *p)

// kthread_run - create and wake a thread.
#define kthread_run(threadfn, data, namefmt, ...) // ...
```
</details>

## workqueue
The current `workqueue` implementation is called Concurrency Managed Workqueue (cmwq).
`cmwq` maintains a set of unified per-CPU worker pools.
The work items on all the `workqueue`s are all executed by those pools in the end.
Workers are just kthreads,
so running things using a `workqueue` work is just like running them in a kthread.

`workqueue`'s queuing process involves several `spin_lock` with interrupt disabled.
The work is first added to a worker pool's.
And then a worker thread in the pool if available is woken up to execute the work item.
`queue_work()` cannot be used in critical portions of the kernel, like in the NMI context.

<details><summary>Useful APIs</summary>

```c
#include <linux/workqueue.h>

struct work_struct {
	work_func_t func;
    // ...
};
struct delayed_work {
	struct work_struct work;
	struct timer_list timer;
    // ...
};
#define INIT_WORK(_work, _func)	// ...
#define INIT_DELAYED_WORK(_work, _func) // ...

// work_pending - Find out whether a work item is currently pending
#define work_pending(work) // ...
// delayed_work_pending - Find out whether a delayable work item is currently
#define delayed_work_pending(w) // ...
extern unsigned int work_busy(struct work_struct *work);

// queue_work - queue work on a workqueue
static inline bool queue_work(struct workqueue_struct *wq,
			      struct work_struct *work);
extern bool queue_work_on(int cpu, struct workqueue_struct *wq,
			struct work_struct *work);
extern bool queue_work_node(int node, struct workqueue_struct *wq,
			    struct work_struct *work);
extern bool queue_delayed_work_on(int cpu, struct workqueue_struct *wq,
			struct delayed_work *work, unsigned long delay);

/*
 * System-wide workqueues which are always present.
 *
 * system_wq is the one used by schedule[_delayed]_work[_on]().
 * Multi-CPU multi-threaded.  There are users which expect relatively
 * short queue flush time.  Don't queue works which can run for too
 * long.
 *
 * system_highpri_wq is similar to system_wq but for work items which
 * require WQ_HIGHPRI.
 *
 * system_long_wq is similar to system_wq but may host long running
 * works.  Queue flushing might take relatively long.
 *
 * system_unbound_wq is unbound workqueue.  Workers are not bound to
 * any specific CPU, not concurrency managed, and all queued works are
 * executed immediately as long as max_active limit is not reached and
 * resources are available.
 *
 * system_freezable_wq is equivalent to system_wq except that it's
 * freezable.
 *
 * *_power_efficient_wq are inclined towards saving power and converted
 * into WQ_UNBOUND variants if 'wq_power_efficient' is enabled; otherwise,
 * they are same as their non-power-efficient counterparts - e.g.
 * system_power_efficient_wq is identical to system_wq if
 * 'wq_power_efficient' is disabled.  See WQ_POWER_EFFICIENT for more info.
 */
extern struct workqueue_struct *system_wq;
extern struct workqueue_struct *system_highpri_wq;
extern struct workqueue_struct *system_long_wq;
extern struct workqueue_struct *system_unbound_wq;
extern struct workqueue_struct *system_freezable_wq;
extern struct workqueue_struct *system_power_efficient_wq;
extern struct workqueue_struct *system_freezable_power_efficient_wq;
```
</details>

### delayed_work
Things that need to be executed as soon as possible (`struct work_struct`) or
are only needed after some delay (`struct delayed_work`),
can both be executed via `workqueue`.

When a `work_struct` is enqueued,
a idle worker if found will be woken up via a `try_to_wake_up()`
and execute the work immediately.
However, `try_to_wake_up()` isn't safe to be called in any context.
When called from `perf_event->overflow_handler` a spinlock recursion bug will occur.
To solve this, we can use a `delayed_work`.
When queuing a `delayed_work`, a timer will be added.
The timer will only be triggered after the specified delay.
In the timer callback, the `workqueue` facility will try to find an idle worker,
and starting executing the work.

Basically, the timer interrupt will check for the timers associated with `delayed_work`.
If there is something to do,
`ksoftirqd` kernel thread will be woken up to queue the underlying `work`
and wake up a `workqueue` worker to execute the actual `work`.



<details><summary>delayed_work invocation stack</summary>

```c
=> hrtimer_interrupt()
=> raise_softirq_irqoff()
=> wakeup_softirqd()
=> wake_up_process(__this_cpu_read(ksoftirqd))
-> run_ksoftirqd()
=> __do_softirq()
=> h->action(h)
=> run_timer_softirq() // setup by open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
=> __run_timers()
=> expire_timers()
=> call_timer_fn()
=> timer->function()
=> delayed_work_timer_fn() // setup by __queue_delayed_work()
=> __queue_work()
=> kick_pool()
=> wake_up_process(first_idle_worker(pool)->task)
-> kthread_worker_fn() // created by kthread_create_worker()
=> work->func(work)
```
</details>



