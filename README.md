# 4190.307 Operating Systems (Spring 2020)
# Project #4: Simplified Linux 2.4 Scheduler
### Due: 11:59PM (Sunday), May 3

## Introduction

Currently, the CPU scheduler of ``xv6`` uses a simple round-robin policy. The goal of this project is to understand the scheduling subsystem of the ``xv6`` by implementing a simplified version of the Linux 2.4 scheduler algorithm.


## Background

### O(n) scheduler in Linux 2.4

The scheduling algorithm used in the Linux 2.4 kernel is a quite simple and straightforward priority scheduler based on dynamic priority. It was called as the _O(n)_ scheduler because the scheduler needs to iterate over every task during a scheduling event. The complexity of the scheduler has been improved in the Linux 2.6 kernel resulting in the _O(1)_ scheduler, which is eventually replaced by the CFS(Completely Fair Scheduler) in Linux 2.6.23.

The main characteristics of the _O(n)_ scheduler is to divide processor time into __epochs__. Within each epoch, every task can execute up to its time slice. The amount of time slice is given based on the __nice__ value which represents the task's static priority. If the currently running task exhausts all the time slice, then the kernel schedules the next task that has the maximum __goodness__. The goodness of a task is calculated considering its remaining time slice, its nice value, etc. When all the runnable processes used up their time slices, a new epoch begins and the scheduler redistributes the time slice to *all* tasks. Note that the scheduler adds half of the remaining time slice for blocked tasks so that they can be scheduled quicker and executed longer when they are woken up. 

## Problem specification

In order to implement and test a simplified Linux 2.4 scheduler, we introduce two new system calls, ``nice()`` and ``getticks()``. Also we have added three new fields, ``p->nice``, ``p->counter`` and ``p->ticks`` in the ``proc`` structure of ``xv6`` as shown below.

<pre>
// Per-process state (in kernel/proc.h)
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  <b>// The following three fields are added for PA4</b>
  <b>int nice;                 // Nice value (-20 <= nice <= 19)</b>
  <b>int counter;              // Time slice</b>
  <b>int ticks;                // Total number of ticks used</b>

  ...
}
</pre>

### 1. Implement the ``nice()`` system call

First, you need to implement the ``nice()`` system call that is used to set the ``nice`` value of a process.

__SYNOPSIS__
```
    int nice(int pid, int inc);
```

__DESCRIPTION__

The ``nice()`` system call adds ``inc`` to the current nice value of the process ``pid``. The range of the nice value is from -20 to 19, where a higher nice value means a lower priority. When a process is created, its nice value is inherited from the parent by default. The nice value of the ``init`` process should be set to zero. 

* If ``pid`` is positive, then the nice value of the process with the specified ``pid`` is changed. 
* If ``pid`` is zero, then the nice value of the calling process is changed.

__RETURN VALUE__

* On success, zero is returned. 
* On error, -1 is returned. The possible error conditions are as follows.
  - ``pid`` is negative.
  - There is no valid process that has ``pid``.
  - The resulting nice value exceeds the range [-20, 19].


### 2. Implement a Simplified Linux 2.4 scheduler

Our simplified Linux 2.4 scheduler works as follows:

1. On every timer tick, the ``p->counter`` value of the currently running process ``p`` is decremented by 1. If ``p->counter`` value becomes zero, it means that the process used up all its time slice and the scheduler is invoked. 

2. For the next process to run, the scheduler picks the process that has the maximum __goodness__ value. The goodness of a process is calculated based on the following formula. If there are more than one process that have the same goodness value, you can pick any of them.

```
    goodness(p) = 0                            , if p->counter is zero
                = p->counter + (20 - p->nice)  , Otherwise
```


3. Eventually, the ``p->counter`` values of all the runnable processes will become zero. Now the scheduler moves on to a new __epoch__ and redistributes the time slice for __*all*__ processes. Basically, the time slice (in ticks) of a process is determined by the following formula:
```
    p->counter = ((20-(p->nice)) >> 2) + 1                       // for runnable processes
```
For example, for the default process that has the nice value of 0, its time slice is 6 ticks. The highest-priority process with the nice value of -20 will receive 11 ticks, while the lowest-priority process (with the nice value of 19) will get only 1 tick per epoch.

The blocked processes (e.g., processes waiting for I/O) will have non-zero ``p->counter`` values at the beginning of a new epoch. If a task did not use all of its time slice, then half of the remaining time slice is added to the new time slice to allow it to execute quickly and longer in the next epoch. Hence, for blocked processes, the time slice is given by:
```
    p->counter = p->counter >> 1 + ((20-(p->nice)) >> 2) + 1      // for blocked processes
```

Note that because only half of the remaining time slice is added to the base time slice, the ``p->counter`` value never becomes larger than twice its base time slice. 

4. When forking a new child process, the parent process' remaining time slice is split between the parent and the child; i.e., ``(p->counter + 1) >> 1`` is given to the parent, and ``p->counter >> 1`` is given to the child. This prevents users from forking new children to get unlimited time slice.

5. Once scheduled, the running process is not preempted until the end of its time slice even if a new process with the higher goodness value wakes up.


### 3. Revise the ``getticks()`` system call

The skeleton code already includes an implementation of the ``getticks()`` system call for the default round-robin scheduler. You need to make sure the ``getticks()`` system call returns the correct number of ticks used by each process under the new scheduling algorithm.

__SYNOPSIS__
```
    int getticks(int pid);
```

__DESCRIPTION__

The ``getticks()`` system call returns the number of ticks used by the process ``pid``. The kernel maintains per-process value, ``p->ticks``, for each process. The ``p->ticks`` value is initialized to zero when the corresponding process is created. Afterwards, the ``p->ticks`` value of the currently running process is incremented by one on every timer interrupt.

* If ``pid`` is positive, then return the number of ticks used by the process with the specified ``pid``.

* If ``pid`` is zero, then return the number of ticks used by the calling process.


__RETURN VALUE__

* On success, the number of ticks is returned.
* On error (e.g., no process found), -1 is returned.


## Skeleton code

Since no code from the previous projects are needed, we recommend you to download the fresh ``xv6`` source code from Github to avoid any possible conflict. The skeleton code for this project (PA4) is available as a branch named ``pa4``. 

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa4
```
After downloading, you have to set your STUDENTID again in the ``Makefile``. 

The ``pa4`` branch includes two user-level programs called ``schedtest1`` and ``schedtest2`` whose source code is available in ``./user/schedtest1.c`` and ``./user/schedtest2.c``, respectively. ``schedtest1`` performs a stress test on the scheduler; it first creates 30 processes and then each process repeats some computation and sleep. At the same time, one of the existing processes is killed and then created again every 3 ticks. The second program, ``schedtest2``, creates three CPU-intensive child processes and then measures the number of ticks used by those processes, while changing their nice values every 300 ticks. 

The following shows a sample output of the ``schedtest2`` program when it is run under the default round-robin scheduler. The first column shows the elapsed time in seconds, and the rest of the columns represent the number of ticks used by process P1, P2, and P3 during the 5-second period. You can see that each process uses exactly the same number of ticks. 

```
xv6 kernel is booting

init: starting sh
$ schedtest2
0, 1, 1, 1
5, 17, 17, 17
10, 17, 17, 17
15, 17, 17, 17
20, 17, 17, 17
25, 17, 17, 17
30, 17, 17, 17
35, 17, 17, 17
40, 17, 17, 17
45, 17, 17, 17
50, 17, 17, 17
55, 17, 17, 17
60, 17, 17, 17
65, 17, 17, 17
70, 17, 17, 17
75, 17, 17, 17
80, 17, 17, 17
85, 17, 17, 17
90, 17, 17, 17
95, 17, 17, 17
100, 17, 17, 17
105, 17, 17, 17
110, 17, 17, 17
115, 17, 17, 17
120, 17, 17, 17
125, 17, 17, 17
130, 17, 17, 17
135, 17, 17, 17
140, 17, 17, 17
145, 17, 17, 17
150, 17, 17, 17
155, 17, 17, 17
$
```

Once you implement the simplified Linux 2.4 scheduler successfully, you should be able to get the result something similar to the following:

```
xv6 kernel is booting

init: starting sh
$ schedtest2
0, 7, 7, 6
5, 18, 18, 18
10, 18, 18, 18
15, 18, 18, 18
20, 18, 18, 18
25, 18, 18, 18
30, 28, 18, 7
35, 33, 18, 3
40, 33, 18, 3
45, 33, 18, 3
50, 33, 18, 3
55, 33, 18, 3
60, 27, 18, 5
65, 27, 18, 6
70, 27, 18, 6
75, 27, 18, 6
80, 27, 18, 6
85, 27, 18, 6
90, 24, 18, 8
95, 24, 18, 9
100, 24, 18, 9
105, 24, 18, 9
110, 24, 18, 9
115, 24, 18, 9
120, 21, 18, 11
125, 21, 18, 12
130, 21, 18, 12
135, 21, 18, 12
140, 21, 18, 12
145, 21, 18, 12
150, 21, 18, 12
$
```

In the above example, you can see that the number of ticks used by each process varies depending on its nice value which is changed every 5 seconds. We provide you with a Python script called ``graph.py`` in the ``./xv6-riscv-snu`` directory. You can use the Python script to convert the above ``xv6`` output into a graph as follows:

```
$ make qemu-log
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 3G -smp 1 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 | tee xv6.log

xv6 kernel is booting

init: starting sh
$ schedtest2                <---- run the schedtest2 program
0, 7, 7, 6
5, 18, 18, 18
10, 18, 18, 18
...
$ QEMU: Terminated          <---- quit the qemu using ^a-x. xv6 output is available in the xv6.log file

$ make png                  <---- Generate the graph. (this should be done on Ubuntu, not on xv6)
./graph.py xv6.log graph.png
$
```

If everything goes fine, you will get the following graph:

![PNG image here](https://github.com/snu-csl/os-pa4/blob/master/graph.png)


## Restrictions

* To make the problem easier, we assume a single-processor machine in this project. The ``CPUS`` variable that represents the number of CPUs in the target QEMU machine emulator is already set to 1 in the ``Makefile``.

* Your implementation should pass the following test programs available on ``xv6``:
  * ``usertests``
  * ``schedtest1``
  * ``schedtest2``

* Do not add any system calls other than ``nice()`` and ``getticks()``.

* You only need to modify those files in the ``./kernel`` directory. Changes to other source code will be ignored during grading.

## Hand in instructions

* Please remove all the debugging outputs before you submit. 
* To submit your code, please run ``make submit`` in the ``xv6-riscv-snu`` directory. It will create a file named ``xv6-PA4-STUDENTID.tar.gz`` file in the parent directory. Please upload the file to the server.
* By default, the ``sys.snu.ac.kr`` server is only accessible from the SNU campus network. If you want to access the server outside of the SNU campus, please send a mail to the TA.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delay.
* __You can use up to 5 _slip days_ during this semester__. If your submission is delayed by 1 day and if you decided to use 1 slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server after each submission. Saving the slip days for later projects is highly recommended!
* Any attempt to copy others' work will result in heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
