+++
+++
## Memtis 测试

### 代码分析

[Memtis会为每个memory node启动一个kmigrated](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L1125). [根据node的层级会分别执行demotion和promotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L1080). [demotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L996)/[promotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L1062)是以polling形式等待是否有页需要迁移. [demotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L998)/[promotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L1062)采用可调参数形式决定每次polling iteration之间的等待时间.

#### demotion

[demotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L989)时检测[是否需要为promotion让出空间](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L165), [只要在下层有active anonymous页的情况下就会尝试](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L121C19-L121C34), demotion的页数量等于需要为promotion让出的空间加上达到`max_watermark`所需释放的空间. 即memtis会保持上层空闲内存水位在`max_watermark`, 同时也会为防止promotion无法申请内存而保留额外的空闲页. 这个额外的空闲页为promotion需要的页或者是100MiB(如果处于direct demotion mode, 即allocator或promotion需要立即取得上层内存时).

demotion会[优先demote文件页和inactive anonymous页](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L594), 只有[在direct demotion mode中, 需要紧急demote来满足分配或promotion请求时, 这些还不能达到需要demote的内存量下才会demote active anonymous页](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L602).

[demotion页数量计算公式](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L174C20-L174C91):

````
nr_exceeded
= nr_lru_pages + nr_need_promoted + fasttier_max_watermark - max_nr_pages
= nr_need_promoted + fasttier_max_watermark - (max_nr_pages - nr_lru_pages)
  (为promotion让出的空间)   +    (达到max_watermark所需释放的空间)
````

### promotion

[promotion](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L1058)时[会检测是否在下层node中有的active anonymous页](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L121)且[下层空闲内存超过`max_watermark`](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L642), 如果有则进行promotion.  [promotion的页数量](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L650C2-L650C70)为[下层active anonymous内存的页数](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L645)或[下层空闲内存超过`max_watermark`的页数](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L227), 二者取小值. 即尽量把下层active anonymous页全部迁移, 但是尽量不要让空闲内存加上被换下来的页低于`max_watermark`.

promotion页数量计算公式:

```
nr_to_promote
= min(
 max_nr_pages - fasttier_max_watermark - cur_nr_pages - nr_isolated, // 空闲内存超过max_watermark的页数
 lruvec_lru_size(lruvec, LRU_ACTIVE_ANON, MAX_NR_ZONES) // active anonymous的大小
  )
```

### TODO

- [ ] memtis的migration真的是按照distribution决定迁移优先级的么? migrate的page 平均访问次数是多少? ~~代码看memtis只是简单的从lru中顺序拿取, 并没有检查access counter.~~ memtis的[demotion中会检查`warm_threshold`, 如果比他还低则可以demote](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/htmm_migrater.c#L399).
- [ ] memtis的histogram中warm的页到底被放在了active还是inactive list中? promotion的时候是怎么处理的?

## Migration Profiling

对于一次migration需要收集的数据:

- 方向: promotion/demotion
- 当前内存状态:
  - 各个watermark以及可用内存数量
