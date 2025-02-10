+++
title = "Kernel Trace Point"
date = "2024-01-15"
# [extra]
# add_toc = true
+++

While checking Memtis' source code,
I found that it leverages trace points to do profiling and debugging.
I have long wanted to try this.

Linux's [`ftrace`](https://docs.kernel.org/trace/ftrace.html) subsystem enables users to
dynamically configure what kernel functionality to inspect at runtime.
It's realized using trace points.

Let's see a classical example:

<details><summary>Trace point definition for task switches:</summary>

```c
// include/trace/events/sched.h
/*
 * Tracepoint for task switches, performed by the scheduler:
 */
TRACE_EVENT(sched_switch,

	TP_PROTO(bool preempt,
		 struct task_struct *prev,
		 struct task_struct *next),

	TP_ARGS(preempt, prev, next),

	TP_STRUCT__entry(
		__array(	char,	prev_comm,	TASK_COMM_LEN	)
		__field(	pid_t,	prev_pid			)
		__field(	int,	prev_prio			)
		__field(	long,	prev_state			)
		__array(	char,	next_comm,	TASK_COMM_LEN	)
		__field(	pid_t,	next_pid			)
		__field(	int,	next_prio			)
	),

	TP_fast_assign(
		memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
		__entry->prev_pid	= prev->pid;
		__entry->prev_prio	= prev->prio;
		__entry->prev_state	= __trace_sched_switch_state(preempt, prev);
		memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
		__entry->next_pid	= next->pid;
		__entry->next_prio	= next->prio;
		/* XXX SCHED_DEADLINE */
	),

	TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> next_comm=%s next_pid=%d next_prio=%d",
		__entry->prev_comm, __entry->prev_pid, __entry->prev_prio,

		(__entry->prev_state & (TASK_REPORT_MAX - 1)) ?
		  __print_flags(__entry->prev_state & (TASK_REPORT_MAX - 1), "|",
				{ TASK_INTERRUPTIBLE, "S" },
				{ TASK_UNINTERRUPTIBLE, "D" },
				{ __TASK_STOPPED, "T" },
				{ __TASK_TRACED, "t" },
				{ EXIT_DEAD, "X" },
				{ EXIT_ZOMBIE, "Z" },
				{ TASK_PARKED, "P" },
				{ TASK_DEAD, "I" }) :
		  "R",

		__entry->prev_state & TASK_REPORT_MAX ? "+" : "",
		__entry->next_comm, __entry->next_pid, __entry->next_prio)
);
```
</details>

```c
TRACE_EVENT(sched_switch, // ...
```

<details><summary>Which is called in the main scheduler function:</summary>

```c
// kernel/sched/core.c
/*
 * __schedule() is the main scheduler function.
 */
static void __sched notrace __schedule(unsigned int sched_mode)
{
	struct task_struct *prev, *next;
	// ...
	rq = cpu_rq(cpu);
	prev = rq->curr;
	// ...
	next = pick_next_task(rq, prev, &rf);
    // ...
	if (likely(prev != next)) {
		// ...
		RCU_INIT_POINTER(rq->curr, next);
		// ...
        trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next);
		// ...
	} else {
        // ...
	}
}
```
</details>

```c
trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next);
```

A trace point contains 6 parts, i.e.:
- name
- trace point function prototype
- variable names of arguments
- trace data format
- trace data assignment
- printing

The name, format and printing can be checked from userspace via `tracefs`,
which is typically mounted under `/sys/kernel/tracing`.
For a example, the `sched_switch` event's format can be checked from:
`/sys/kernel/tracing/events/sched/sched_switch/format`.

Other than the native linux `ftrace` interface,
trace points could be used with a more convient tool `bpftrace`.
Because trace points are statically defiend,
they are guaranteed to exists,
which is not like other raw kernel functions (`kfunc`).


