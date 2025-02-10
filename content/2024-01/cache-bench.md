+++
title = "Cache Bench"
date = "2024-01-10"
# [extra]
# add_toc = true
+++

We want to know exactally how good our SDH is.
While [SOSP23]S3-FIFO provides a good example of benchmarking existing caching solutions,
it does not contains our SDH and it's implementation is unspeakable.

I decide to make a very simple cache benchmark library using Rust.
I plan to implement S3-FIFO, TinyLFU and our SDH.

## How should we abstract the cache algorithms
Normal systems that require a cache also have a backed storage.
The basic operations a that cache support include read (`get(key)`) and write (`upsert(key, value)`).

<details><summary>For a read, the cache could choose to support read-through or not, as shown in the pseudo code:</summary>

```python
# w/o read-through
def read(key, cache, storage):
    if cache.contains(key):
        return cache.get(key)
    if not storage.contains(key):
        return None
    value = storage.get(key)
    while not cache.hasspace(value.len()):
        k, v = cache.evit()
        if storage.contains(k):
            storage.update(k, v)
        else
            storage.insert(k, v)
    cache.insert(key, value)
    return value

# w/ read-through
def read(key, cache, storage):
    if cache.contains(key):
        return cache.get(key)
    if not storage.contains(key):
        return None
    value = storage.get(key)
    return value
```
</details>

<details><summary>For a write, there is also a choice between write-back and write-through:</summary>

```python
# write-back
def write(key, value, cache, storage):
    if cache.contains(key):
        cache.update(key, value)
        return
    while not cache.hasspace(value.len()):
        k, v = cache.evit()
        if storage.contains(k):
            storage.update(k, v)
        else
            storage.insert(k, v)
    cache.insert(key, value)

# write-through:
def write(key, value, cache, storage):
    # this can still happen without read-through
    if cache.contains(key):
        cache.remove(key)
    if storage.contains(key):
        storage.update(key)
    else
        storage.insert(key, value)
```
</details>

As shown in the pseudo code,
a cache needs to have the following interface:

| Fn                       | Meaning                                                      |
| ------------------------ | ------------------------------------------------------------ |
| `contains(key) -> bool`  | Check if the given `key` already cached.                     |
| `get(key) -> value`      | Read the `value` associated with the given `key`.            |
| `hasspace(size) -> bool` | Check if the cache can fit a new value with the given `size`. (Assume that `value` always can fit in the cache after remove some evictions.) |
| `evit() -> (key, value)` | Evict any value from the cache.                              |
| `insert(key, value)`     | Insert the `key` and `value` into the cache.                 |
| `remove(key)`            | Evict the `value` associated with the given `key`.           |

## Off-critical-path cache
These interfaces should be enough for implementing a arbitrary synchronous cache policy.
The most typical one would be: write-back cache without read-through.

However, it's not enough for off-critical-path cache (or known as tiering [FAST21]NHC):
read/write-through cache with asynchronous migration.
<details><summary>The pseudo code for such cache would be:</summary>

```python
def read(key, cache, storage):
    send(key)
    if cache.contains(key):
        return cache.get(key)
    if storage.contains(key):
        return storage.get(key)
    return None

def write(key, value, cache, storage):
    send(key)
    if cache.contains(key):
        cache.update(key)
    if storage.contains(key):
        storage.update(key)

async def migrate(cache, storage):
    for key in recv():
        if cache.contains(key):
            continue
        value = storage.get(key)
        while not cache.hasspace(value.len()):
            k, v = cache.evit()
            if storage.contains(k):
                storage.update(k, v)
            else
                storage.insert(k, v)
        cache.insert(key, value)
```
</details>

As shown in the pseudo code above,
unlike the on-critial-path case,
in which the subsequent accesses are guaranteed to see the results of cache decision
the hit rate largely depends on the latency between access discovery and final migration.

If the burst of accesses have a smaller duration than the latency,
the cache will be doing wasted work because
nothing will be accessed again in a near future.

## Metrics for evaluation
Reuse time distribution.
