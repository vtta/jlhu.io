+++
title = "Access Distribution of Common Benchmarks"
date = "2023-11-13"

+++



| Benchmark   | Usable | [SOSP21]HeMem | [SOSP23]Memtis |
| ----------- | ------ | ------------- | -------------- |
| GUPS        | √      | +             |                |
| Silo        | √      | +             | +              |
| FlexKVS     |        | +             |                |
| GAP-BC      | √      | +             |                |
| GAP-PR      | √      |               |                |
| SPECCPU-603 |        |               | +              |
| SPECCPU-654 |        |               | +              |
| Liblinear   | √      |               | +              |
| BTree       |        |               | +              |
| XSBench     |        |               | +              |
| PageRank    |        |               | +              |
| Graph500    |        |               | +              |
|             |        |               |                |
|             |        |               |                |



## Silo (dbtest --verbose --bench ycsb --slow-exit )

| --threads | --scale-factor | --ops-per-worker= | Memory delta (M) | DB size (M) | RSS max (K) | Load time (ms) | Runtime(s) | Elapsed |
| --------- | -------------- | ----------------- | ---------------- | ----------- | ----------- | -------------- | ---------- | ------- |
| 4         | 4000           | 1000              | 39.0312          | 602.027     | 774416      | 927.257        | 0.002667   | 0:01.14 |
| 4         | 4000           | 1000000           | 145.355          | 601.281     | 883768      | 933.095        | 0.683553   | 0:01.88 |
| 4         | 4000           | 100000000         | 209.863          | 603.77      | 991816      | 922.709        | 70.4167    | 1:11.63 |
| 4         | 4000           | 100000000         | 176.359          | 5954.52     | 7113108     | 9550.01        | 92.7303    | 1:44.92 |



## Silo access coverage test

```bash
sudo perf record --count <period> --phys-data --data --weight -z -vv -e MEM_TRANS_RETIRED.LOAD_LATENCY_GT_64:Pu -- /bin/time --verbose -- out-perf.masstree/benchmarks/dbtest --verbose --bench ycsb --num-threads 4 --scale-factor 40000 --ops-per-worker=100000000 --slow-exit
sudo perf --no-pager script -F addr,phys_addr | rg -v fffff | rg 7f > output
```

How much memory is touched by a given percentile of accesses?

| period | samples  | estimated accesses | P90 coverage        | P95 coverage         | P99 coverage         | P999 coverage        | Full coverage        |
| ------ | -------- | ------------------ | ------------------- | -------------------- | -------------------- | -------------------- | -------------------- |
| 1      | 88610803 | 88610803           | 0.6130977277652074  | 0.7730033431768285   | 0.9240028579311527   | 0.970641612997781    | 1                    |
| 17     | 9763005  | 165971085          | 0.46458638317904577 | 0.624430319266878    | 0.8232700235756748   | 0.880813551855566    | 0.8872078604760153   |
| 107    | 1616069  | 172919383          | 0.25276777435383163 | 0.30568535334956     | 0.3480190228483312   | 0.3575445413955926   | 0.3586029323452883   |
| 1007   | 174920   | 176144440          | 0.04371423649081078 | 0.0494359780167541   | 0.05401337123750875  | 0.055042891014367136 | 0.05515771954269742  |
| 10007  | 17781    | 177934467          | 0.00528867393338339 | 0.005871346694282206 | 0.006337222437793646 | 0.006442208520838478 | 0.006454019455181021 |

The percentage of pages that have been accessed less or equal to C times when the period is 1:

| C     | pages   | percentage           |
| ----- | ------- | -------------------- |
| 1     | 21061   | 0.013816643607219214 |
|       |         |                      |
| 2     | 32228   | 0.02114252837820905  |
| 4     | 39439   | 0.025873159262386335 |
| 8     | 52622   | 0.03452160010916336  |
| 10    | 62844   | 0.04122753671962795  |
| 12    | 95182   | 0.06244222837578174  |
| 14    | 171745  | 0.11266983791471744  |
| 16    | 309120  | 0.20279193162070194  |
|       |         |                      |
| 17    | 400147  | 0.2625083561795711   |
| 53    | 1358498 | 0.8912151705579074   |
| 107   | 1512725 | 0.9923926784450257   |
| 503   | 1514365 | 0.9934685673162018   |
| 1007  | 1523273 | 0.9993124807701265   |
| 5003  | 1523752 | 0.9934685673162018   |
| 10007 | 1524233 | 0.9999422693776442   |

If we compare sample period with the access count, we might find using larger period to estimate hot-set has larger error.

| C or period | coverage             | tail percentage      | 1 - tail         | ratio          |
| ----------- | -------------------- | -------------------- | ---------------- | -------------- |
| 1           | 1                    | 0.013816643607219214 | 0.9861833564     | 1.01401021779  |
| 17          | 0.8872078604760153   | 0.2625083561795711   | 0.7374916438     | 1.2030073397   |
| 107         | 0.3586029323452883   | 0.9923926784450257   | 0.007607321555   | 47.1391842388  |
| 1007        | 0.05515771954269742  | 0.9993124807701265   | 0.0006875192299  | 80.2271662288  |
| 10007       | 0.006454019455181021 | 0.9999422693776442   | 0.00005773062236 | 111.7954248775 |



```python
#!/usr/bin/env python
# coding: utf-8

# In[446]:

from functools import *
import numpy as np
import matplotlib.pyplot as plt
import pandas as pd
from pathlib import Path

# In[447]:

raw = Path("output").open("r").read()

# In[448]:

pfns = np.fromiter(map(lambda x: int(x, 16) >> 12, raw.split()),
                   dtype=np.uint64).reshape((-1, 2)).T

# In[449]:

vpfns = pd.Series(pfns[0])

# In[450]:

plt.boxplot(x=vpfns, showmeans=True)

# In[451]:

df = pd.DataFrame(dict(pfn=vpfns))
Q1 = df.quantile(0.25)
Q3 = df.quantile(0.75)
IQR = Q3 - Q1

df = df[~((df < (Q1 - 3 * IQR)) | (df > (Q3 + 3 * IQR))).any(axis=1)]

# In[452]:

plt.boxplot(df, showmeans=True)

# In[453]:

(Q1, Q3, IQR, len(df) - len(vpfns))

# In[ ]:

# In[454]:

df.hist(bins=100)

# In[455]:

df['count'] = df['pfn'].map(df['pfn'].value_counts())

# In[456]:

sorted = df.sort_values('count', ascending=False).reset_index(drop=True)

# In[457]:

sorted

# In[458]:

df2 = df['pfn'].value_counts()
df2 = pd.DataFrame(dict(pfn=df2.index, count=df2.values))

# In[459]:

df2

# In[460]:

df2['cumsum'] = df2['count'].cumsum()

# In[461]:

df2

# In[462]:

df2['cdf'] = df2['cumsum'] / df2['count'].sum()

# In[463]:

df2

# In[476]:

p95 = df2[df2['cdf'] < 0.95]

# In[477]:

p95

# In[478]:

len(p95), len(df2), (len(df2) - len(p95)) / len(df2)

# In[479]:

len(p95) * 4096 / (5953.17 * 1024 * 1024)

# In[ ]:

# In[ ]:
```





## GAP

| workload | scale (-g) | degree (-k) | trials (-n) | iteration (-i) | threads ([OMP_NUM_THREADS](https://www.openmp.org/spec-html/5.0/openmpse50.html)=) | file size (B) | RSS (K)  | ratio         | average trial time (s) | elapsed  |
| -------- | ---------- | ----------- | ----------- | -------------- | ------------------------------------------------------------ | ------------- | -------- | ------------- | ---------------------- | -------- |
| pr       | 27         | 24          | 16          | 20             | 144                                                          | 26277071857   | 27177336 | 1.05908269443 | 86.5043                | 23:21.56 |
| pr       | 24         | 24          | 16          | 20             | 4                                                            | 3229895617    | 3352420  | 1.06284489874 | 5.36867                | 1:27.99  |
| bc       | 24         | 24          | 16          | 20             | 4                                                            | 3229895617    | 3546136  | 1.1242602531  | 16.53271               | 4:30.79  |
| tc       | 24         | 24          | 16          | 20             | 4                                                            | 3229895617    | 6834832  | 2.1669022154  | 409.76528              | 1:49:17  |
| bfs      | 24         | 24          | 16          | 20             | 4                                                            | 3229895617    | 3288108  | 1.04245554385 | 0.22021                | 0:04.77  |
|          |            |             |             |                |                                                              |               |          |               |                        |          |
| pr       | 26         | 12          | 16          | 20             | 4                                                            | 6872100289    | 7498788  |               | 23.93817               | 6:30.49  |
| bc       | 26         | 12          | 16          | 20             | 4                                                            | 6872100289    | 8055708  |               | 58.54718               | 15:52.94 |
| bfs      | 26         | 12          | 16          | 20             | 4                                                            | 6872100289    | 7236612  |               | 3.19704                | 1:12.96  |
| tc       | 26         | 12          | 16          | 20             | 4                                                            | 6872100289    |          |               |                        |          |





## XSBench

| preset (-s) | threads (-t) | gridpoints (-g) (\~RSS) | particles (-p) (\~runtime) | lookup (-l) | RSS (K) (=512*g) | runtime |
| ----------- | ------------ | ----------------------- | -------------------------- | ----------- | ---------------- | ------- |
| large       | 4            | 11303                   | 500000                     | 34          | 5786212          | 7.517   |
| -           | 4            | 14500                   | 500,000                    | 34          | 7420504          | 7.759   |
| -           | 4            | 14500                   | 7500,000                   | 34          | 7420504          | 117.094 |



## liblinear

| solver (-s) | thread (-m) | dataset                 | training size | file size  | RSS      | ~~ratio~~        | elapsed |
| ----------- | ----------- | ----------------------- | ------------- | ---------- | -------- | ---------------- | ------- |
| 6           | 4           | kdda                    | 8,407,752     | 2670162192 | 11935828 | ~~4.5773578506~~ | 2:23.42 |
| 6           | 4           | url_combined_normalized | 2,396,130     | 4119894512 | 9163848  | ~~2.2776749076~~ | 1:18.73 |
| 2           | 4           | kdda                    | 8,407,752     | 2670162192 | 7395476  |                  | 2:49.90 |
|             |             |                         |               |            |          |                  |         |





## Overall access coverage

| benchmark | workload       | period | samples   | estimated accesses | max rss | P90 coverage         | P95 coverage        | P99 coverage        | P999 coverage      | Full coverage      |
| --------- | -------------- | ------ | --------- | ------------------ | ------- | -------------------- | ------------------- | ------------------- | ------------------ | ------------------ |
| silo      | ycsb           | 1      | 88693164  | 88693164           | 7092704 | 0.5263701967542985   | 0.6640621122776307  | 0.7940841743853966  | 0.8342324732570258 | 0.8596642408875373 |
| gap-bfs   | s24d24         | 1      |           |                    |         |                      |                     |                     |                    |                    |
| gap-pr    | s24d24         | 1      | 454701798 | 454701798          | 3352420 | 0.01818027588810594  | 0.14482213986518616 | 0.6862388083239547  | 0.8987223508553105 | 0.9924719775007587 |
| gap-bc    | s24d24         | 1      | 129004732 | 129004732          | 3546136 | 0.24341536816410878  | 0.485309080080403   | 0.8061100871483778  | 1.0021172340823927 | 1.0021172340823927 |
| gap-bfs   | s26d12         | 1      | 10484493  | 10484493           | 7236612 | 0.35327084000081804  | 0.4441166667495784  | 0.5626312423548478  | 0.5954858433753254 | 0.5991361703515402 |
| gap-pr    | s26d12         | 1      | 923585021 | 923585021          | 7498788 | 0.031941161691729385 | 0.12205119013899313 | 0.6788569032755694  | 0.8828674713833755 | 1.0156953363663568 |
| gap-bc    | s26d12         | 1      | 257121963 | 257121963          | 8055708 | 0.22855495755307914  | 0.43515479955331055 | 0.7668877769651035  | 0.954252065740218  | 1.002242385151001  |
| liblinear | url-s6         | 1      | 33557005  | 33557005           | 9163848 | 0.40133430654454744  | 0.5187311125583134  | 0.7633103246058696  | 0.8563197865967719 | 0.8666538236160458 |
| liblinear | kdda-s2        | 1      | 530268843 | 530268843          | 7392536 | 0.6460359476098595   | 0.7353043664582763  | 0.8427889969017398  | 0.925332254046514  | 0.9810111171592536 |
| xsbench   | g14500p1500000 | 1      | 76622569  | 76622569           | 7421420 | 0.031092701935748146 | 0.03523476639241547 | 0.35162165731086503 | 0.6590490768613014 | 0.6998153992093158 |
| xsbench   | g14500p7500000 | 1      | 383691407 | 383691407          | 7420220 | 0.031193684284293456 | 0.03538331747576218 | 0.45940147327168196 | 0.8479748578883106 | 0.9754589486565088 |

```python
#!/usr/bin/env python3

from functools import partial
import matplotlib.pyplot as plt
import pandas as pd

from pathlib import Path
import sys

def eprint(*args, **kwargs):
    print(*args, file=sys.stderr, **kwargs)

def remove_outliers(df):
    Q1 = df.quantile(0.25)
    Q3 = df.quantile(0.75)
    IQR = Q3 - Q1
    oldlen = len(df)
    df = df[~((df < (Q1 - 3 * IQR)) | (df > (Q3 + 3 * IQR))).any(axis=1)]
    newlen = len(df)
    removed = oldlen - newlen
    eprint(f"{Q1=}\n{Q3=}\n{IQR=}\n{oldlen=} {newlen=} {removed=}")
    return df

def cdf(df, column):
    df = df[column].value_counts().reset_index().sort_values('count', ascending=False).reset_index(drop=True)
    df['cumsum'] = df['count'].cumsum()
    df['cdf'] = df['cumsum'] / df['count'].sum()
    return df

def percentile_coverage(df, p, total_pages):
    df2 = df[df['cdf'] < p]
    position = len(df2)
    total = len(df)
    tail = (len(df) - len(df2)) / len(df)
    pcoverage = len(df2) / total_pages
    coverage = len(df) / total_pages
    eprint(f"{p=} {position=} {total=} {tail=} {pcoverage=} {coverage=}")
    return df2

page_size = 4096
mmap_end = (1<<47) - page_size
heap_begin = mmap_end // 3 * 2
heap_end = mmap_begin = heap_begin + (mmap_end - heap_begin) // 2

total_pages = int(sys.argv[1], 0) // page_size
data = pd.read_csv(
        sys.stdin, 
        delim_whitespace=True, 
        header=None, 
        names=['va', 'pa'], 
        converters=dict(
            va=partial(int, base=16), 
            pa=partial(int, base=16)
        ),
)
data['vpfn'] = data['va'] // page_size
data['ppfn'] = data['pa'] // page_size

heap = data[(heap_begin < data['va']) & (data['va'] < heap_end)]
mmap = data[(mmap_begin < data['va']) & (data['va'] < mmap_end)]

for (name, data) in [('heap', heap), ('mmap', mmap)]:
    eprint(f"{name=}\n{data.head()=}")
    vpfn = pd.DataFrame(dict(vpfn=data.vpfn))
    fig, ((box0, box1), (hist0, hist1)) = plt.subplots(nrows=2, ncols=2, figsize=(8.5, 11))
    box0.boxplot(vpfn, showmeans=True)
    vpfn = remove_outliers(vpfn)
    box1.boxplot(vpfn, showmeans=True)

    hist0.hist(vpfn.vpfn, bins=1000)

    df = cdf(vpfn, 'vpfn')
    hist1.plot(df.cdf)

    percentile_coverage(df, .90, total_pages)
    percentile_coverage(df, .95, total_pages)
    percentile_coverage(df, .99, total_pages)
    percentile_coverage(df, .999, total_pages)
    percentile_coverage(df, 1, total_pages)

    # fig.savefig('plot.pdf')
    # fig.savefig('plot.svg')
    fig.savefig(f'{name}.png', dpi=600)
    eprint()

```

