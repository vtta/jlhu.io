+++
title = "Balloon OOM"
date = "2023-11-29"
+++

Virtio-balloon has a feature called `DEFLATE_ON_OOM`,
which allows the balloon driver to deflate and use pages it previously returned to the host when out of memory.

This behaviour is controlled additionally by `MUST_TELL_HOST`.
When this is enabled, the driver must wait for the acknowledgment of the deflated pages before using.
Otherwise, the driver has the freedom to directly using pages even the deflation request is not completed.

<details><summary>virtio specification</summary>

> If `VIRTIO_BALLOON_F_DEFLATE_ON_OOM` has been negotiated,
> the driver MAY use pages from the balloon when num_pages is less than or equal to
> the actual number of pages in the balloon if this is required for system stability
> (e.g. if memory is required by applications running within the guest).
>
> The driver MUST use the deflateq to inform the device of pages that it wants to use from the balloon.
>
> If the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is negotiated,
> the driver MUST NOT use pages from the balloon until the device has acknowledged the deflate request.
>
> Otherwise, if the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is not negotiated,
> the driver MAY begin to re-use pages previously given to the balloon
> before the device has acknowledged the deflate request.
</details>

## Implementation
The current linux balloon driver has support for `DEFLATE_ON_OOM` but not for `MUST_TELL_HOST`.
This is dangerous.
When multiple guests OOM at the same time,
the host system might not have enough free pages to satisfy all the deflation requests
and might kill some (very likely innocent) guests.

The balloon drivers register a callback to be called firstly when entering the OOM killer.
This callback will try to deflate a set number of 4K pages (256).
If some pages are freed, the OOM killer will return and will not proceed to kill any processes.
This callback will not wait for the acknowledgment of deflation requests and
will allow the use of deflated pages instantly.

### `DEFLATE_ON_OOM` implementation details
<details><summary>source code</summary>

```c
// drivers/virtio/virtio_balloon.c
if (virtio_has_feature(vb->vdev, VIRTIO_BALLOON_F_DEFLATE_ON_OOM)) {
		vb->oom_nb.notifier_call = virtio_balloon_oom_notify;
		vb->oom_nb.priority = VIRTIO_BALLOON_OOM_NOTIFY_PRIORITY;
		err = register_oom_notifier(&vb->oom_nb);
		if (err < 0)
			goto out_unregister_shrinker;
}

// drivers/virtio/virtio_balloon.c
static int virtio_balloon_oom_notify(struct notifier_block *nb,
				     unsigned long dummy, void *parm)
{
	struct virtio_balloon *vb = container_of(nb,
						 struct virtio_balloon, oom_nb);
	unsigned long *freed = parm;

	*freed += leak_balloon(vb, VIRTIO_BALLOON_OOM_NR_PAGES) /
		  VIRTIO_BALLOON_PAGES_PER_PAGE;
	update_balloon_size(vb);

	return NOTIFY_OK;
}

// drivers/virtio/virtio_balloon.c
#define VIRTIO_BALLOON_OOM_NR_PAGES 256

// mm/oom_kill.c
int register_oom_notifier(struct notifier_block *nb)
{
	return blocking_notifier_chain_register(&oom_notify_list, nb);
}
EXPORT_SYMBOL_GPL(register_oom_notifier);

// mm/oom_kill.c
/**
 * out_of_memory - kill the "best" process when we run out of memory
 * @oc: pointer to struct oom_control
 *
 * If we run out of memory, we have the choice between either
 * killing a random task (bad), letting the system crash (worse)
 * OR try to be smart about which process to kill. Note that we
 * don't have to be perfect here, we just have to be good.
 */
bool out_of_memory(struct oom_control *oc)
{
	unsigned long freed = 0;

	if (oom_killer_disabled)
		return false;

	if (!is_memcg_oom(oc)) {
		blocking_notifier_call_chain(&oom_notify_list, 0, &freed);
		if (freed > 0 && !is_sysrq_oom(oc))
			/* Got some memory back in the last second. */
			return true;
	}
    // ...
}
```
</details>

<details><summary>ChatGPT's explaination on notifier_block</summary>

**Q: Can you explain the related concepts and APIs to `struct notifier_block` in the linux kernel?**

Certainly! In the Linux kernel, the `struct notifier_block` is a fundamental data structure used for implementing the notification mechanism. This mechanism allows different parts of the kernel to register interest in specific events and be notified when those events occur. This is commonly used for implementing callback mechanisms.

Here's an overview of the related concepts and the `struct notifier_block` API:

### `struct notifier_block`:

The `struct notifier_block` is defined in the `linux/notifier.h` header file. Its structure looks like this:

```c
struct notifier_block {
    int (*notifier_call)(struct notifier_block *nb, unsigned long action, void *data);
    struct notifier_block *next;
    int priority;
};
```

- **`notifier_call`**: This is a function pointer to the callback function that will be invoked when a notification occurs. The callback function takes three parameters:
  - `struct notifier_block *nb`: A pointer to the `notifier_block` structure itself.
  - `unsigned long action`: An identifier for the specific event or action that has occurred.
  - `void *data`: A pointer to additional data related to the event.

- **`next`**: A pointer to the next notifier block in the chain. Multiple notifier blocks can be linked together to form a chain.

- **`priority`**: The priority of the notifier block. When multiple notifier blocks are registered for the same event, the kernel invokes them in order of decreasing priority.

### Notifier Chains:

Notifier blocks are often organized into chains to form a linked list of callbacks. The Linux kernel maintains these chains for different events. When an event occurs, the kernel iterates through the corresponding notifier chain, invoking each callback in turn.

### Notifier Call Chain Functions:

The Linux kernel provides several functions to manipulate notifier chains:

- **`int register_notifier(struct notifier_block *nb)`**: Registers a notifier block with the corresponding event.

- **`int unregister_notifier(struct notifier_block *nb)`**: Unregisters a notifier block.

- **`int blocking_notifier_call_chain(struct notifier_block **chain, unsigned long action, void *data)`**: Invokes the notifier callbacks in a blocking manner. The kernel waits for each callback to complete before moving on to the next one.

- **`int notifier_call_chain(struct notifier_block **chain, unsigned long action, void *data)`**: Invokes the notifier callbacks in a non-blocking manner. The callbacks are called one after the other without waiting for them to complete.

### Example Usage:

Let's say you want to register interest in a specific event and perform some action when that event occurs. You would:

1. Create a `struct notifier_block` and initialize its `notifier_call` function pointer with your callback function.
2. Register the notifier block using `register_notifier`.
3. When the event occurs, the kernel will call your callback function, and you can perform the desired action.
4. Unregister the notifier block when it is no longer needed using `unregister_notifier`.

This mechanism is extensively used in the Linux kernel for various subsystems to communicate and coordinate activities.

Keep in mind that specific events and their identifiers (`action` values) vary depending on the context and subsystem within the kernel where you are working. Always refer to the relevant documentation for the specific subsystem you are dealing with.</details>
