+++
title = "task_struct, Thread and Process"
date = "2023-10-06"
# [extra]
# add_toc = true
+++

Linux does not have a clear conceptual separation between thread and process,
although they are the most basic terms when disscussing scheduling related topics.

A process, by definition, is the basic unit of resource management.
A process can contail multiple threads.
A thread is the basic unit of scheduling.
In Linux's words, threads are `task_struct`s, and a process is a "task group".
When someone wants to identify all threads of one paticular process,
he can find out all the threads sharing the same tgid.

Below is a diagram from [stackoverflow](https://stackoverflow.com/a/9306150),
which shows the Linux's mess very clearly.
```
                         USER VIEW
                         vvvv vvvv
              |          
<-- PID 43 -->|<----------------- PID 42 ----------------->
              |                           |
              |      +---------+          |
              |      | process |          |
              |     _| pid=42  |_         |
         __(fork) _/ | tgid=42 | \_ (new thread) _
        /     |      +---------+          |       \
+---------+   |                           |    +---------+
| process |   |                           |    | process |
| pid=43  |   |                           |    | pid=44  |
| tgid=43 |   |                           |    | tgid=42 |
+---------+   |                           |    +---------+
              |                           |
<-- PID 43 -->|<--------- PID 42 -------->|<--- PID 44 --->
              |                           |
                        ^^^^^^ ^^^^
                        KERNEL VIEW
```


## Additional materials
- https://zhuanlan.zhihu.com/p/136404837
- https://cloud.tencent.com/developer/article/1339563
