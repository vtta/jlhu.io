+++
+++
看看目前Firecracker主线 (v1.10.1) 对snapshot的支持. 根据之前cloudhypervisor的经验, 除了命令行我们主要和hypervisor打交道的途径其实API. 那么先搞清楚snapshot从API会是一个不错的切入点, 因为他有一个比较直观的[定义](https://github.com/firecracker-microvm/firecracker/blob/v1.10.1/src/firecracker/swagger/firecracker.yaml)和[文档](https://github.com/firecracker-microvm/firecracker/blob/v1.10.1/docs/snapshotting/snapshot-support.md).

快照相关的接口大概包括: 创建 `PUT /snapshot/create`; 和恢复 `PUT /snapshot/load`. 最终调用的是`vmm`的`create_snapshot/restore_from_snapshot`

其中创建的快照类型可以是全量也可以是增量快照. [^diff] 全量快照创建时会返回: 系统状态`snapshot_path`; 内存状态`mem_file_path`; 不会包括磁盘文件, 这个需要用户手动复制. 同时为了支持增量快照, 脏页记录会被全部清空. 增量快照同样会返回系统状态和内存状态, 只不过内存状态只包括脏页. 脏页信息是由底层的KVM的脏页记录提供的[^pml]. 

FC的快照恢复后, 除开正常由内核处理内存文件缺页, 原生还支持`uffd`. 恢复时`backend_path`给出`uffd`的`socket`路径. 正常情况则直接给出内存状态文件. FC也给出了`uffd`的[文档](https://github.com/firecracker-microvm/firecracker/blob/v1.10.1/docs/snapshotting/handling-page-faults-on-snapshot-resume.md);

快照创建后或者恢复后VM都处于暂停状态, 需要一个继续API`PATCH /vm`来继续执行. 一些快照的workflow例子如下:

- `Boot from a fresh microVM` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> ... ;
- `Boot from a fresh microVM` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` -> `Resume` -> ... -> `Pause` -> `Create snapshot` -> ... ;
- `Load snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> ... ;
- `Load snapshot` -> `Resume` -> `Pause` -> `Create snapshot` -> `Resume` -> `Pause` -> `Resume` -> ... -> `Pause` -> `Create snapshot`-> ... ;

[^diff]: 这其实暗示FC原生就有detect working set的能力.
[^pml]: 不知道KVM的脏页记录是否是基于PML.

---

实现中关于系统状态的部份我们并不需要修改, 着重关注内存状态. `vmm`中的对应应该是`GuestMemoryState`, 其由遍历`GuestMemoryMmap`中的每个区间创建.

`GuestMemoryMmap`的结构其实很简单, 就是一堆`AtomicBitmap`. 每个`AtomicBitmap`负责一个region. 目前需要搞清楚这个region到底指的是什么? 难道VM的内存不是一整块连续的mmap mapping么? 还是说在处理缺页时会创建很多不连续的region? 

关于这个`AtomicBitmap`和KVM的`dirty log` (FC中叫`DirtyBitmap`) 并不是一个东西, 但是推测这个结构是拿来暂存`DirtyBitmap`的结果. 因为`deepseek`在对`KVM_GET_DIRTY_LOG`的介绍中提到, 默认情况内核在返回位图后会**自动清除脏页标记**[^clear]. 所以FC需要保存一下结果, 用来做错误处理等, 需要重试的逻辑. FC中保存增量备份过程中调用的函数`dump_dirty()`也会同时检查FC和KVM的两个`Bitmap`结果.

另外`dump_dirty()`的逻辑也相当简单: 内存状态文件中全部的区间背靠背存放, 写入过程直接将脏页写到对应地址, 干净页直接跳过.

[^clear]: [内核文档](https://www.kernel.org/doc/Documentation/virt/kvm/api.txt)中也应证了该说法: “The bits in the dirty bitmap are cleared before the ioctl returns, unless `KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2` is enabled.”

---

接下来要考虑的是如何记录只读页. 其实也比较obvious, FaaSnap记录的是present page, 而FC记录的是写入了的页, 二者相减即是read-only页. 那么接下来则需要看看FaaSnap是如何修改FC实现的present tracking. 同时需要注意的是如何实现layer多个内存状态文件.

FaaSnap文章中介绍是使用了`mincore`检查页表A-bit, 简单rg发现FaaSnap的FC中并没有相关代码, 实际上他是在测试框架中实现的. 实现方法也很简单: 只需要定期扫描内存状态文件, 看看有哪些页被map了. 原文是按照RSS增大触发扫描. 

我们可以考虑直接实现在FC中, 新引入一个API触发`mincore`扫描, 记录到一个新的Bitmap结构中. 最后在保存内存状态时额外存储一个文件. 这样就没必要想FaaSnap再实现一个API中间件了, 也不用考虑访问其他进程打开的文件的问题. 缺点可能是复杂度稍微高点, 但是考虑到我们对rust更为熟悉, 对go更不熟悉, 而且对超大代码库中下的开发有经验, 此方法对我们更友好点. 

---

要实现mincore和layering, 顺序应该是先做layering. 因为在layering情况下, mincore需要对每个layer都调用. 可以参考FaaSnap的layering实现.

先看看FC恢复内存镜像的方法: 跟上面保存内存镜像的方法和FC代码可以看到其大致流程是先读取系统状态`MicrovmState`; 然后根据其中的`GuestMemoryState`元数据记录的所有内存区间依次将内存状态文件中的相应区间`mmap`到内存. (要注意的是, 其中每个内存区间的存储方式是背靠背存储, 结果就是这同一个文件会被`mmap`多次.) 

为了兼容这种存储方式, 我们可以在文件最后加上一个header, 其中记录layer的metadata. 在FC原本恢复逻辑结束后, 读取上面的layer信息, 然后再kj重新`mmap`这些区间.

当然这种方式比较hacky, 被选的方案可以是out-of-band patching. 即在恢复之前, 我们先对`GuestMemoryState`元数据以及内存状态文件进行修改, 预处理得到layer之后对所有小区间, 再写回并依赖FC原逻辑处理.



---

关于测试. 目前FC的test是通过pytest以及python的FC API client实现的, 对我们很友好. 另外snapshot相关的test也可以作为我们的参考.

~~目前使用devtool+rootless docker总会出现devtool的docker会把文件用户组改成100999的情况. 可以参考[这里](https://joeeey.com/blog/rootless-docker-avoiding-common-caveats/)使用`sudo setfacl -Rm d:u:jlhu:rwX,u:jlhu:rwX build test_results resources/$(uname -m)`保证我们一直拥有对devtool碰过的目录的权限. 另外想要撤回修改可以使用`sudo setfacl -b -R -d`.~~ 目前已经放弃使用rootless mode, 徒增烦恼, 可以直接将当前用户加入docker用户组`sudo usermod -aG docker $USER`.

目前可以成功在本地运行FC自带的pytest测试. 所需要的patch如[`fc-local-tests.diff`](fc-local-tests.diff). 最终的测试输出如[`fc-local-tests.log`](fc-local-tests.log), 结果如[`fc-local-tests.json`](fc-local-tests.json).

---

接下来就是仿照现有的test来跑实验. 目前发现现有的一个snapshot相关测试流程大概是先启动VM, 然后从host中将一些helper scp到guest, 最后通过vsock与guest做交互. 我们也可以仿照这个逻辑, 开机后setup所有需要的Fn, 然后通过vsock调用Fn.




