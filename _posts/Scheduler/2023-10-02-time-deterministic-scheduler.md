<font size='5'><center>Time-deterministic scheduler</center></font>

The time determination scheduler is the specific implementation of the "deterministic scheduling time" in the figure, including the establishment of the "deterministic execution environment" of the "preconditions" and the statistics of the user task WCET. Time-based deterministic scheduling can be broken down into:

![deterministic execution](/commons/images/3141911-20230609154213117-326442955.png)

1. Determine Point in Time: Start the execution of the user task at a point in time.

2. Determine the end of time: The execution of the user task is completed at a certain point in time.

3. Determine the amount of time allocation: Give users a fixed time quota for tasks in each scheduling cycle.

The time-determining scheduler in DataFLow belongs to the scenario of "starting at a fixed point in time" (that is, ensuring that the task can be executed at a predetermined start time), which is only applicable to the Linux environment.

![time-determining scheduler](/commons/images/3141911-20230609161204802-1514915239.png)

Procs that can currently be pushed to the time-determined scheduler are MSG type and TIMER type (see Dataflow-Module Introduction chapter)

1. In principle, it is not recommended to assign MSG-type Proc to the time-determined scheduler, because the scheduler is executed frame by frame, which is equivalent to the tasks in the orchestration table are periodic.

2. If it is a TIMER type Proc and has Input, then the Input of Proc is satisfied, bound to Task, and pushed into the lock-free queue. The execution thread takes the Task from the lockless queue and executes it; If the lockless queue is empty, the TaskNull event is skipped and reported.

3. If it is a TIMER type Proc, but there is no Input, you can register a Task with RegisterTask() before starting the scheduler, it does not enter the lock-free queue and can be used repeatedly.

4. wcet sampling, used in DEBUG mode, counts the execution time of user functions.

5. The default scheduling policy for execution threads is SCHED_FIFO, priority 98, and can be configured.

![time-deterministic scheduler](/commons/images/3141911-20230609161326301-1686036557.png)


The files involved in the demo are as follows:

+ The project startup tool mainboard2 and related configuration files

  + mainboard2

  + process.json

  + runtime_context.json

  + schedule.json

+ Time to qualitatively scheduler related files

  + demo1_dag_debug.json: DAG description file (no WCET information)

+ Task orchestration tools

  + task_scheduler

DAGs Design

![DAG description](/commons/images/3141911-20230609161636322-1317638452.png)

<font size='5'><center>Timing scheduler</center></font>

# Function description

The scheduled scheduler is mainly used to execute periodic tasks on a regular basis. Users can configure the following options:

+ Task execution cycle (in ms)

+ The function that the user expects to be executed

+ Set the number of periodic task executions (-1 means until the timer is canceled)

+ Configure the timed period interval

+ Specifies the timing task clock type, including steady-state clock/system clock/analog clock.

+ Task trigger types, including DefaultTrigger/SelfTrigger, when the execution time of a periodic task exceeds the cycle interval, the next execution trigger timing is as follows:

  + DefaultTrigger: Triggers execution based on the task cycle interval.
  + SelfTrigger: Repeat immediately.

## Core Design

1. The timing scheduler is implemented by the time library and Dataflow. Built-in timing checking thread and multiple task execution threads.

2. The Timer library provides an executor base class, and DataFlow inherits the interface from this base class and implements related interface logic, mainly including adding/canceling tasks. The overall logic is controlled by the timer. 
3. Timer library: implements the logic of checking threads, periodically detects whether the task times out, blocks when it is judged that the current task has not timed out, and waits until the next timeout; When the timeout timer is obtained, the task is added to the DataFlow check thread for execution. Dataflow: To implement the task execution logic, DataFlow has 4 built-in execution threads by default, and the timer library generates the corresponding timer_id for each timer, modulifies the id, and adds the same timer_id tasks to the corresponding execution thread queue.

![Timer thread scheduler](/commons/images/3141911-20230609163529084-764899901.png)
