A note for reading super big file in Python
=================================

*I am very welcome for the hardware expert to ask some mysterious things in my experimental
setup. It will be a good direction for making more robust and consisent experiment.*
## Spec check. Environment (Disk performance)

* RAID5
* 3 4TB HDD
* Logicaly, total 8TB

Check the peak performance of HDD.
```
sudo hdparm -Tt /dev/md0
```

### Result
```
/dev/md0:
 Timing cached reads:   18804 MB in  1.99 seconds = 9436.72 MB/sec
 Timing buffered disk reads: 1124 MB in  3.00 seconds = 374.53 MB/sec
```

## Experiment 1. Compressed file (.gz) vs Uncompressed file
Before the experiment, I made two assumption. First, we are stuck in the I/O bound, so 
reducing the size of file read (by reading the compressed file) will bring us performance
improvement. Second, we are stuck in the CPU bound, so direct use of uncompressed file
give us performance improvement.  
The test was conducted by using 90GB file. (Compressed version: 22GB, Uncompressed version: 90GB)

### Script for compressed file
```python
import gzip

filepath = 'somewhere/in/the/disk.gz'
with gzip.open(filepath) as f:
    s = time.time()
    for line in f:
        pass
    e = time.time()
print("TIME: %f" % (e - s))
```

### Script for uncompressed file
```python
filepath = 'somewhere/in/the/disk'
with open(filepath) as f:
    s = time.time()
    for line in f:
        pass
    e = time.time()
print("TIME: %f" % (e - s))
```

### Result

Compressed file gives about 9 min. However, uncompressed file gives about 4 min.  

Q. Is 4min the peak read performance?  
A. 374 (MB/s) * 240 (s) = 89760 (MB) = 87.6562 (GB). I think it is peak performance.
In retrospect, I think it could be CPU bound because I assume that all cache miss which
is unnatural although it is sequential access. The other clue is the performance of `wl`
command in Linux. It just takes 5 seconds! There are some magical implementation to count
the number of line without reading, but it seems not (if it is, then the execution time might
be ~50ms).

Q. Possability of using `mmap`?  
A. For now, I'm not using mmap. Maybe, it could improve the performance, but I hesitate to use it
because I am not confident about the technique. memory-mapped could raise an issue about consistency
which is the aspect I want to avoid the most. If I am ready to use mmap using python, I will add it
in the following experiment.

Q. Why the compressed version takes so long?  
A. Maybe, since there is no operation in the forloop, uncompressed processing
could not hide by other works (e.g. pipeline). If there are some operations in the forloop,
It may be different, but I just decide not to do that because **the reason why I use the compressed
file is to avoid the I/O bound**, but the incurred CPU bound (uncompressed data) is more harsh than the I/O bound.

| Strategy               | Pros                           | Cons                         | Performance in my environment |
|------------------------|:------------------------------:|:----------------------------:|:-----------------------------:|
| Read compressed file   | Small required disk space      | Uncompressed for every read  | 9min  |
| Read uncompressed file | Direct read without processing | Large required disk space    | 3min  |

## Experiment 2. Read one file vs Read multiple files with multi-thread
Actually, in the Experiment 1, reading single file gives the peak performance. The reason for this experiment
is to the direction for using multi-core in Python. Because of GIL in Python, we have to use `multiprocess` library
to make use of parallelism. In that situation, each process will read each file. This experiment is the preliminary
experiment for the reachability to the peak performance when we read multiple different files **throught multi-thread**
(might incur cache miss?). 

To make multiple file, I simply use `split` command in Linux. Each will be 11GB.

```shell
$ split -n l/8 -d my_file my_file-
$ ls
my_file my_file-00 my_file-01 my_file-02 my_file-03 my_file-04 my_file-05 my_file-06 my_file-07 
```

### Script for reading one file
```python
filepath = 'somewhere/in/the/disk'
with open(filepath) as f:
    s = time.time()
    for line in f:
        pass
    e = time.time()
print("TIME: %f" % (e - s))
```

### Script for reading multiple file
```python
import time
import threading

def fun(x):
    with open('somewhere/in/the/disk-%02d' % x) as f:
        for line in f:
            pass
            
if __name__ == '__main__':
    threads = []
    for i in range(8):
        threads.append(threading.Thread(target=fun, args=(i,)))
        
    s = time.time()
    for th in threads:
        th.start()
    for th in threads:
        th.join()
    e = time.time()
    print('TIME: %f' % (e - s))
```

### Result
Both takes about 4min. No problem that I worried occured.

## Experiment 3. Multi-thread vs Multi-process when do something with file content
For now, I do something with file content. The difference between the previous experiment is 
that the CPU have to do something. Therefore, if CPU power is low, then the CPU bound will be critical
and the way to solve it is to take benefit from the multi-core parallelism. This will give us what is 
better choice, which will be obviously multiprocessing :)

### Script for reading multiple file using multi-threading
```python
import time
import json
import threading

def fun(x):
    with open('somewhere/in/the/disk-%02d' % x) as f:
        for line in f:
            json.loads(line)
            
if __name__ == '__main__':
    threads = []
    for i in range(8):
        threads.append(threading.Thread(target=fun, args=(i,)))
        
    s = time.time()
    for th in threads:
        th.start()
    for th in threads:
        th.join()
    e = time.time()
    print('TIME: %f' % (e - s))
```

### Script for reading multiple file using multi-processing
```python
import time
import json
import multiprocessing

def fun(x):
    with open('somewhere/in/the/disk-%02d' % x) as f:
        for line in f:
            json.loads(line)
            
if __name__ == '__main__':
    processes = []
    for i in range(8):
        processes.append(multiprocessing.Process(target=fun, args=(i,)))
        
    s = time.time()
    for p in processes:
        p.start()
    for p in processes:
        p.join()
    e = time.time()
    print('TIME: %f' % (e - s))
```

### Result
Multi-thread version takes more than 20min, whereas multi-process version takes 4min.

Q. Why the multi-process version reach the peak read performance?  
A. A plausible explanation will be this one. There is a minimal peak read latency, let's say `l_min`.
If the CPU does something within `l_min`, then the CPU must be wait for new data. If the time CPU does
something is more than `l_min`, then the CPU time is critical and read performance doesn't need to be peak
because CPU can't afford to handle new data. In terms of multi-process version, assuming that each process take
the disk read performance equally, the minimal read latency will be `l_min * n_process`. Therefore, each CPU
have `l_min * n_process` to handle data. The fact that the multi-process version reaches the peak performance means
that we don't have to divide task to 8 process. **In practice, the data processing will be more complex so that the
time to process data will be larger than `l_min * n_process`, which is CPU bound.**

## Conclusion
We have explored the set of simple but slightly different experiments to check what is going on when we read large data.
By analyzing the experimental result, we can ensure that whether the disk is fully exploited and how to do it when it isn't.
