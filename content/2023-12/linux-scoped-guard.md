+++
title = "Linux Scoped Guard"
date = "2023-12-04"
+++

[Recently](https://lwn.net/Articles/941051/),
Linux adopts a more ergonomic locking API, i.e. `scoped_guard`.
This API leverages scope and the variable attribute `__cleanup__` to automatically release held locks
when a scope ends, just like what is taken for granted in most mordern languages such as `rust`.
This design can eliminate the messy goto error handling practice widely used across the entire code base today.

### A basic feel
```c
spinlock_t l;
int x;

scoped_guard (spinlock_irqsave, &l) {
    x = 123;
}

// it also works for locks without arguments
scoped_guard (rcu) {
    // we are now inside a rcu_read_lock() protected region
}
```

### The magic sauce
The magic behind all this is the [`__cleanup__` variable attribute](https://echorand.me/site/notes/articles/c_cleanup/cleanup_attribute_c.html).
This variable attribute enables C programer to [define a destructor to be called when the variable of interest goes out of scope.](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#index-cleanup-variable-attribute)
[This example](https://clang.llvm.org/docs/AttributeReference.html#cleanup) offered by clang shows the basic syntax:
```c
static void foo (int *) { ... }
static void bar (int *) { ... }
void baz (void) {
    int x __attribute__((cleanup(foo)));
    {
        int y __attribute__((cleanup(bar)));
    }
}
```
The dtor used should accept a pointer to the variable.
A more meaningful example would be automatically freeing allocated memory, e.g.:
```c
void buf_dtor(char **buf) { free(*buf); }
void baz (void) {
    char *buf __attribute__ ((__cleanup__(buf_dtor))) = malloc(20);
}
```

### In action
This `scoped_guard` API has already been used when handling `execv()`.
<details><summary>code</summary>

```c
// kernel/sched/core.c
/*
 * sched_exec - execve() is a valuable balancing opportunity, because at
 * this point the task has the smallest effective memory and cache footprint.
 */
void sched_exec(void)
{
	struct task_struct *p = current;
	struct migration_arg arg;
	int dest_cpu;

	scoped_guard (raw_spinlock_irqsave, &p->pi_lock) {
		dest_cpu = p->sched_class->select_task_rq(p, task_cpu(p), WF_EXEC);
		if (dest_cpu == smp_processor_id())
			return;

		if (unlikely(!cpu_active(dest_cpu)))
			return;

		arg = (struct migration_arg){ p, dest_cpu };
	}
	stop_one_cpu(task_cpu(p), migration_cpu_stop, &arg);
}
```

Comparing to the old version from v6.1, we can remove the messy goto label `unlock`:
```c
void sched_exec(void)
{
	struct task_struct *p = current;
	unsigned long flags;
	int dest_cpu;

	raw_spin_lock_irqsave(&p->pi_lock, flags);
	dest_cpu = p->sched_class->select_task_rq(p, task_cpu(p), WF_EXEC);
	if (dest_cpu == smp_processor_id())
		goto unlock;

	if (likely(cpu_active(dest_cpu))) {
		struct migration_arg arg = { p, dest_cpu };

		raw_spin_unlock_irqrestore(&p->pi_lock, flags);
		stop_one_cpu(task_cpu(p), migration_cpu_stop, &arg);
		return;
	}
unlock:
	raw_spin_unlock_irqrestore(&p->pi_lock, flags);
}
```
</details>

### Nitty-gritty details
The core implementation of this API is the `scoped_guard` macro:

<details><summary>code</summary>

```c
// include/linux/cleanup.h
#define scoped_guard(_name, args...)					\
	for (CLASS(_name, scope)(args),					\
	     *done = NULL; !done; done = (void *)1)

#define CLASS(_name, var)						\
	class_##_name##_t var __cleanup(class_##_name##_destructor) =	\
		class_##_name##_constructor

/*
 * Additional helper macros for generating lock guards with types, either for
 * locks that don't have a native type (eg. RCU, preempt) or those that need a
 * 'fat' pointer (eg. spin_lock_irqsave).
 *
 * DEFINE_LOCK_GUARD_0(name, lock, unlock, ...)
 * DEFINE_LOCK_GUARD_1(name, type, lock, unlock, ...)
 *
 * will result in the following type:
 *
 *   typedef struct {
 *	type *lock;		// 'type := void' for the _0 variant
 *	__VA_ARGS__;
 *   } class_##name##_t;
 *
 * As above, both _lock and _unlock are statements, except this time '_T' will
 * be a pointer to the above struct.
 */
#define DEFINE_LOCK_GUARD_1(_name, _type, _lock, _unlock, ...)		\
__DEFINE_UNLOCK_GUARD(_name, _type, _unlock, __VA_ARGS__)		\
__DEFINE_LOCK_GUARD_1(_name, _type, _lock)

#define __DEFINE_UNLOCK_GUARD(_name, _type, _unlock, ...)		\
typedef struct {							\
	_type *lock;							\
	__VA_ARGS__;							\
} class_##_name##_t;							\
									\
static inline void class_##_name##_destructor(class_##_name##_t *_T)	\
{									\
	if (_T->lock) { _unlock; }					\
}

#define __DEFINE_LOCK_GUARD_1(_name, _type, _lock)			\
static inline class_##_name##_t class_##_name##_constructor(_type *l)	\
{									\
	class_##_name##_t _t = { .lock = l }, *_T = &_t;		\
	_lock;								\
	return _t;							\
}


// include/linux/spinlock.h
DEFINE_LOCK_GUARD_1(raw_spinlock_irqsave, raw_spinlock_t,
		    raw_spin_lock_irqsave(_T->lock, _T->flags),
		    raw_spin_unlock_irqrestore(_T->lock, _T->flags),
		    unsigned long flags)

```
</details>

It first wrap the logic in the scope into a body for a for loop with only 1 iteration.
The declaration part of the `for` declares a variable called `scope`.
The `scope` variable has a ctor calling the locking funciton and dtor calling the unlock function.
<details><summary>code</summary>

```c
// expansion of the scoped_guard macro:
for (class_raw_spinlock_irqsave_t scope __attribute__((__unused__))
     __attribute__((__cleanup__(class_raw_spinlock_irqsave_destructor))) =
	     class_raw_spinlock_irqsave_constructor(&p->pi_lock),
     *done = ((void *)0);
     !done; done = (void *)1)

// expansion of the macro declaring the ctor and dtor:
typedef struct {
	raw_spinlock_t *lock;
	unsigned long flags;
} class_raw_spinlock_irqsave_t;
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((no_instrument_function)) void
class_raw_spinlock_irqsave_destructor(class_raw_spinlock_irqsave_t *_T)
{
	if (_T->lock) {
		do {
			({
				unsigned long __dummy;
				typeof(_T->flags) __dummy2;
				(void)(&__dummy == &__dummy2);
				1;
			});
			_raw_spin_unlock_irqrestore(_T->lock, _T->flags);
		} while (0);
	}
}
static inline __attribute__((__gnu_inline__)) __attribute__((__unused__))
__attribute__((no_instrument_function)) class_raw_spinlock_irqsave_t
class_raw_spinlock_irqsave_constructor(raw_spinlock_t *l)
{
	class_raw_spinlock_irqsave_t _t = { .lock = l }, *_T = &_t;
	do {
		({
			unsigned long __dummy;
			typeof(_T->flags) __dummy2;
			(void)(&__dummy == &__dummy2);
			1;
		});
		_T->flags = _raw_spin_lock_irqsave(_T->lock);
	} while (0);
	return _t;
}

// declaration for raw_spinlock_irqsave and raw_spin_unlock_irqrestore
#define raw_spin_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		flags = _raw_spin_lock_irqsave(lock);	\
	} while (0)

#define raw_spin_unlock_irqrestore(lock, flags)		\
	do {							\
		typecheck(unsigned long, flags);		\
		_raw_spin_unlock_irqrestore(lock, flags);	\
	} while (0)
```
</details>


### Takeaway
Lastly, untile v6.6, scoped_guard can be used with these locking mechanisms:

| File       | Lock                 |
| ---------- | -------------------- |
| rcupdate.h | rcu                  |
| srcu.h     | srcu                 |
| irqflags.h | irq                  |
|            | irqsave              |
| spinlock.h | raw_spinlock         |
|            | raw_spinlock_nested  |
|            | raw_spinlock_irq     |
|            | raw_spinlock_irqsave |
|            | spinlock             |
|            | spinlock_irq         |
|            | spinlock_irqsave     |
| preempt.h  | preempt              |
|            | preempt_notrace      |
| mutex.h    | mutex                |
| rwsem.h    | rwsem_read           |
|            | rwsem_write          |


<details><summary>code</summary>

```c
kernel/sched/sched.h:DEFINE_LOCK_GUARD_1(rq_lock, struct rq, // ...
kernel/sched/sched.h:DEFINE_LOCK_GUARD_1(rq_lock_irq, struct rq, // ...
kernel/sched/sched.h:DEFINE_LOCK_GUARD_1(rq_lock_irqsave, struct rq, // ...
kernel/sched/sched.h:DEFINE_LOCK_GUARD_2(double_raw_spinlock, raw_spinlock_t, // ...
kernel/sched/sched.h:DEFINE_LOCK_GUARD_2(double_rq_lock, struct rq, // ...
include/linux/srcu.h:DEFINE_LOCK_GUARD_1(srcu, struct srcu_struct, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(raw_spinlock, raw_spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(raw_spinlock_nested, raw_spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(raw_spinlock_irq, raw_spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(raw_spinlock_irqsave, raw_spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(spinlock, spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(spinlock_irq, spinlock_t, // ...
include/linux/spinlock.h:DEFINE_LOCK_GUARD_1(spinlock_irqsave, spinlock_t, // ...
include/linux/preempt.h:DEFINE_LOCK_GUARD_0(preempt, preempt_disable(), preempt_enable())
include/linux/preempt.h:DEFINE_LOCK_GUARD_0(preempt_notrace, preempt_disable_notrace(), preempt_enable_notrace())
include/linux/preempt.h:DEFINE_LOCK_GUARD_0(migrate, migrate_disable(), migrate_enable())
include/linux/rcupdate.h:DEFINE_LOCK_GUARD_0(rcu, rcu_read_lock(), rcu_read_unlock())
include/linux/irqflags.h:DEFINE_LOCK_GUARD_0(irq, local_irq_disable(), local_irq_enable())
include/linux/irqflags.h:DEFINE_LOCK_GUARD_0(irqsave, // ...

include/linux/rwsem.h:DEFINE_GUARD(rwsem_read, struct rw_semaphore *, down_read(_T), up_read(_T))
include/linux/rwsem.h:DEFINE_GUARD(rwsem_write, struct rw_semaphore *, down_write(_T), up_write(_T))
include/linux/mutex.h:DEFINE_GUARD(mutex, struct mutex *, mutex_lock(_T), mutex_unlock(_T))
```
</details>

### Implement your own guard
To enable one class of lock to be used with `scoped_guard`,
a set of convient macros `DEFINE_LOCK_GUARD_x` have been provided,
where `x` means the number of arguments need.

```c
// spinlock.h
DEFINE_LOCK_GUARD_1(spinlock_irqsave, spinlock_t,
		    spin_lock_irqsave(_T->lock, _T->flags),
		    spin_unlock_irqrestore(_T->lock, _T->flags),
		    unsigned long flags)

// rcupdate.h
DEFINE_LOCK_GUARD_0(rcu, rcu_read_lock(), rcu_read_unlock())
```
What this macro does is creating a new class.
The ctor for this class contains the locking routine,
and the dtor contians unlocking routine. 

This class contains at leat one field (named `lock`),
which is used for storing the locked object.
For example, we can make `scoped_guard` work with a `struct` created by ourself:
```c
struct mutex_protected { struct mutex m; void *protected; };
DEFINE_LOCK_GUARD_1(mutex_protected, struct mutex_protected,
		    mutex_lock(&_T->lock->m), mutex_unlock(&_T->lock->m))

struct mutex_protected y;
scoped_guard (mutex_protected, &y) {
    y.protected; // protected by mutex_lock
}
```

This class can also have more fields,
e.g. `spinlock_irqsave` requires an additional flags field to save interrupt status.

