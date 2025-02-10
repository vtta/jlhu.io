+++
+++
测试perf不同时间内能够采样多少sample.

测试方法: perf + timeout + gups

```shell
set -l period 16
for time in 1 2 4 8 16 32 64 128
 set -l label $period-$time
 printf "=== %s ===" $lable
    sudo perf record -z -vv \
        --count $period \
        --phys-data --data --weight \
        -e $event:Pu \
        --output $label.perf.data \
        timeout $time \
        gups --thread=4 --update=800000000 --len=13950255104 --granularity=8 --report=1000 hotset --hot=1065353216 --weight=9 --reverse \
        2> $label.stderr | tee $label.stdout
 echo
end

sudo perf script --force \
    --input $label.perf.data \
    --fields tid,pid,time,phys_addr,addr,event \
    > $label.perf.data.csv 2> $label.perf.data.err
    
```

```python
#!/usr/bin/env python3
import math
import re
from io import StringIO
from pathlib import Path

import pandas as pd
from fire import Fire

HEX = r"0x[0-9a-fA-F]+"
PAGE_SIZE = 4096


def count_hot(samples, start, length):
    count = samples.query(f"{start + length - (1<<30)} <= va < {start} + {length}").shape[0]
    return count


def main(run):
    run = Path(run)
    event, period = run.stem.split("-")
    start, length = list(
        map(
            lambda x: int(x, 0),
            re.search(
                rf"memory (?P<start>{HEX}) length (?P<length>\d+)",
                run.with_suffix(".stdout").read_text(),
            ).groups()))
    samples = pd.read_csv(
        StringIO(run.with_suffix(".perf.data.csv").read_text().replace("/", " ").replace(":", " ")),
        names=["pid", "tid", "ts", "event", "precision", "va", "pa"],
        sep=r"\s+",
    )
    # print(samples)
    samples["va"] = samples["va"].apply(int, base=16)
    samples["pa"] = samples["pa"].apply(int, base=16)
    samples["vpfn"] = samples["va"] / PAGE_SIZE
    samples["ppfn"] = samples["pa"] / PAGE_SIZE
    # query the number of samples whose va falls between the last gib of the memory range
    start_ts, last_ts = samples.ts.min(), samples.ts.max()
    upages = length / PAGE_SIZE
    # print("event,period,start,length,time,total_sample,hot_sample,hot_sample_ratio,hot_page,hot_page_ratio,hot_page_coverage")
    for time in range(1, math.ceil(last_ts - start_ts)):
        sub = samples.query(f"ts < {start_ts + time}")
        total = sub.shape[0]
        hot = count_hot(sub, start, length)
        # unique hot page
        uhot = count_hot(sub.drop_duplicates(subset=["vpfn"]), start, length)
        print(f"{event},{period},{start:#x},{length:#x},{time},{total},{hot},{hot/total:.6f},{uhot},{uhot/total:.6f},{uhot/upages:.6f}")
    total = samples.shape[0]
    hot = count_hot(samples, start, length)
    uhot = count_hot(samples.drop_duplicates(subset=["vpfn"]), start, length)
    print(f"{event},{period},{start:#x},{length:#x},overall,{total},{hot},{hot/total:.6f},{uhot},{uhot/total:.6f},{uhot/upages:.6f}")


if __name__ == "__main__":
    Fire(main)
```

```shell
echo cloud-hyperviso pcm-memory virtiofsd gdb | xargs -n1 sudo pkill -9
sudo swapoff --all
sudo sysctl -w vm.overcommit_memory=1
echo 3 | sudo tee /proc/sys/vm/drop_caches
ulimit -n 65535
echo 3000000 | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_{min,max}_freq
```
