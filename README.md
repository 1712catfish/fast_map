## List of contents
* [Introduction](#introduction)    
* [Characteristics of fast\_map function](#characteristics-of-fast-map-function)  
* [Usage](#usage)  
* [Installation](#installation)  
* [Performance comparison](#performance-comparison) (against multithreading/multiprocessing on their own)   
* [Troubleshooting and issues](#troubleshooting-and-issues)  
* [Considerations](#considerations)  

## Introduction
**What is a map?**  
[map](https://www.w3schools.com/python/ref_func_map.asp) is a python function which allows to repetitively execute the same function without the need to use loops. It executes each task sequentially, meaning that it doesn't start executing a new task before completing the previous one.  

This library allows to execute multiple tasks in parallel using multiple processor cores, and multiple threads to maximise performance even when function is blocking (e.g. it's delayed by `time.sleep()`).  

## Characteristics of fast\_map function
* provides parallelism and concurrency for blocking functions    
* returns a [generator](https://stackoverflow.com/a/70233705/4620679) (meaning that individual returned values are returned immediately after being computed, before the whole collection is returned as a whole)  
* return is ordered (accordingly to supplied arguments)  
* evenly distributes tasks within processes  
* uses the number of threads equal to the number of supplied tasks (unless threads\_limit argument is provided)  
* uses the number of processes equal to 2\*CPU cores (which for some reason worked better during my tests than using the exact CPU cores number) unless the number of tasks is smaller than it  


## Usage
```python
from fast_map import fast_map
import time

def io_and_cpu_expensive_function(x):
    time.sleep(1)
    for i in range(10 ** 5):
        pass
    return x*x

for i in fast_map(io_and_cpu_expensive_function, range(8), threads_limit=None):
    print(i)
```

See [basic\_usage.py](https://github.com/michalmonday/fast_map/tree/master/examples/basic_usage.py) for a more elaborated demonstration.  

## Installation

`python3 -m pip install fast_map`


## Performance comparison
I compared it against using muliprocessing/multithreading on their own. [test\_fast\_map.py](https://github.com/michalmonday/fast_map/tree/master/test/test_fast_map.py ) is the script I used. It was tested with:  
  
Python3.7  
Ubuntu 18.04.6  
Intel i5-3320M   
8GB DDR3 memory

Results show that for IO+CPU expensive tasks fast\_map performs better than multithreading-only and multiprocessing-only approaches. For strictly CPU expensive tasks it performs better than multithreading-only but slightly worse than multiprocessing-only approach.  

In both cases, IO+CPU and strictly CPU expensive tasks, it performs better than the standard map.  

#### IO and CPU expensive task
Standard map is not shown because it would take minutes (as it executes tasks sequentially).  

"-1" result means that ProcessPoolExecutor failed due to "too many files open" (which on my system happens when around 1000 processes are created by the python script). It shows why creating large number of processes to achieve concurrency may be a bad idea. A better idea would be to either:  
* rely on multi-threading itself (which unfortunately utilizes only a single cpu-core)  
* use asyncio (assumming that the blocking code can be turned into coroutines), possibly combined with multiprocessing as shown in [asyncioeval](https://github.com/nbasker/tools/tree/master/asyncioeval)  
* combine multiprocessing with multi-threading just like fast\_map does  

![error - image didn't show](https://github.com/michalmonday/fast_map/blob/master/images/io_and_cpu.png?raw=true)

The following blocking function was used to produce the graph above:  

```python
def io_and_cpu_expensive_blocking_function(x):
    time.sleep(1)
    for i in range(10 ** 6):
        pass
    return x
```

#### Strictly CPU expensive task

It can be noticed that using larger number of threads tends to compute results faster even in CPU expensive tasks, however I would risk a statement that using such large number of threads (e.g. 1 per each task) for a stricly CPU expensive tasks may bring negligible speed improvement of the fast\_map but may possibly slow down the whole system. Because python processes may "fight" with other process over CPU time (that's just my hypothesis).  

![error - image didn't show](https://github.com/michalmonday/fast_map/blob/master/images/cpu_only.png?raw=true)  

The following blocking function was used to produce the graph above:  

```python
def cpu_expensive_blocking_function(x):
    for i in range(10 ** 6):
        pass
    return x
```


## Troubleshooting and issues 
Accessing thread-safe objects (created externally, and using locks under the hood) within the function supplied to fast\_map will probably result in a deadlock.

By default the fast\_map `threads_limit` parameter is `None`, meaning that a separate thread is spawned for **each** of supplied tasks (attempting to provide full concurrency). It is strongly encouraged to set threads\_limit to some reasonable value for 2 reasons:  
* large number of threads will slow down the CPU-expensive part of the blocking function  
* fast\_map will result in unhandled exception when too many threads try to be created   

(btw if threads\_limit is higher than the number of supplied tasks, then the number of created threads equals the number of supplied tasks, so threads\_limit doesn't force the number of created threads, it only limits them)  

## Implementation details
fast\_map uses multiprocessing module and its default process start method (which I believe is `fork` on Unix). It spawns the number of processes equal to the number of CPU cores. For each spawned process it uses a separate task supplying `multiprocessing.Queue` (each has its own for the sake of even task distribution). It uses a singl common results queue for collecting results. It uses `concurrent.futures.ThreadPoolExecutor` to implement multi-threading. It uses a single `threading.Thread` to enqueue all the tasks (this allows to start computation on multiple processes without the need to enqueue all the tasks first).   

It was inspired by a similar project which combined multiprocessing with asyncio:  
[asyncioeval](https://github.com/nbasker/tools/tree/master/asyncioeval) by Nicholas Basker


## Considerations
#### Why not use threading or multiprocessing on their own?  
Multithreading in Python uses a single core on multi-core processors. Multiprocessing isn't well suited to provide concurrency for large number of tasks (on my laptop it fails at around 1000 forked processes). Both of these combined appear to work well with functions expensive in terms or CPU work (e.g. `for i in range(10**6)`) and IO waiting time (e.g. `time.sleep(1)`).  

#### Why not use asyncio for concurrency instead of threading?  
I think asyncio is a good choice over multi-threading when we can modify a blocking function into an awaitable coroutine. If we want/must use a blocking function (e.g. we can't modify it into asyncio coroutine because it's from some library we can't modify) and we want to make it concurrent, asyncio provides `loop.run_in_executor` which relies on multi-threading anyway.   



