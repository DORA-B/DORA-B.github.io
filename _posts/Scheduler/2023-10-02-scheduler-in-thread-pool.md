---
title: Scheduler in thread-pool
date: 2023-10-02
categories: [Scheduler]
tags: [scheduler]     # TAG names should always be lowercase
---
# Classfication of the Scheduler in Thread pool

The capability of the thread pool scheduler is carried by SchedulerGroup, which is a general task scheduler.
(CPU-intensive tasks)

## Simle Scheduler

+ User can set thread properties, priority, and CPU affinity of a thread or all threads
+ Gets the number of threads in the current thread pool is supported
+ Support for adding normal and deferred tasks to the thread pool
+ Supports canceling tasks or canceling all tasks based on task producers
+ Support dynamic start and stop of the thread pool (adding tasks and canceling tasks are not supported after shutdown)

### Core design

1. if the task queue is not added, the task is not created, and the task is created when the task is added.
2. Automatically set the t hread name to the id of the first func passed in
3. Each thread correponds to a task queue, and the tasks are executed in the order in which they are queued.

Need to bind proc to run on a certain thread;

In dataflow, for message-triggered Procs, if the default scheduling policy is used, each Proc will run on a separate thread.

## Fair Thread Pool Scheduler

The fair scheduler mainly ensures the order of tasks and reduces the latency of the overall data link, and supports the following functions:

+ Support to get the number of threads in the current thread pool (maximum number of threads 512)

+ Support for adding tasks to the thread pool

+ Support for adding tasks to the thread pool and returning the thread index of the executed task (mainly used for the Join interface when the resident task exits)

+ Supports blocking waiting for resident tasks to exit

+ Support for canceling tasks related to that producer added to the thread pool through the task producer information

### Core design

For DAG, it is a typical pipeline scenario, which needs to ensure high fps and low delay. On the whole, the number of tasks in each node is not much different, but the frequency of tasks is not synchronized due to the different resources required by different tasks. Regular thread pool, for this scenario, resources will be tilted towards high-frequency tasks, and the overall delay becomes longer. To solve this problem, the fair scheduler uses the following data structure:

+ A group of worker threads

+ A main queue for tasks from which worker threads fetch task execution

+ A task queue with a set of keys for queue_name



## Priority Scheduler

Supports getting the number of threads in the current thread pool

Support for adding tasks to the thread pool

Support for adding tasks to the thread pool and returning the thread index of the executed task (mainly used for the Join interface when the resident task exits)

Supports blocking waiting for resident tasks to exit

Supports canceling tasks added to the thread pool or all tasks within the thread pool based on the task producer

### Core design

+ group of worker threads
+ A set of task queues with different priorities
+ If all tasks are set to the same priority, the priority scheduling type degenerates into the most basic thread pool
+ The thread pool supports a policy of priority escalation, where some tasks are moved to a higher priority queue if they are not executed for more than a specified amount of time
+ Each iteration to see if the task is queued for too long
+ Support for turning off priority runtime updates
+ The number of priority queues is fixed, and the framework abstracts the priority of different levels to prevent performance problems caused by excessive priority
