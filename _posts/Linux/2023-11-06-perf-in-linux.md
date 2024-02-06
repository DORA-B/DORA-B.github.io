Perf Commands:
> perf 

```shell

root@f5c6cb9f45b1:/# perf 

 usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]

 The most commonly used perf commands are:
   annotate        Read perf.data (created by perf record) and display annotated code
   archive         Create archive with object files with build-ids found in perf.data file
   bench           General framework for benchmark suites
   buildid-cache   Manage build-id cache.
   buildid-list    List the buildids in a perf.data file
   c2c             Shared Data C2C/HITM Analyzer.
   config          Get and set variables in a configuration file.
   daemon          Run record sessions on background
   data            Data file related processing
   diff            Read perf.data files and display the differential profile
   evlist          List the event names in a perf.data file
   ftrace          simple wrapper for kernel's ftrace functionality
   inject          Filter to augment the events stream with additional information
   iostat          Show I/O performance metrics
   kallsyms        Searches running kernel for symbols
   kmem            Tool to trace/measure kernel memory properties
   kvm             Tool to trace/measure kvm guest os
   list            List all symbolic event types
   lock            Analyze lock events
   mem             Profile memory accesses
   record          Run a command and record its profile into perf.data
   report          Read perf.data (created by perf record) and display the profile
   sched           Tool to trace/measure scheduler properties (latencies)
   script          Read perf.data (created by perf record) and display trace output
   stat            Run a command and gather performance counter statistics
   test            Runs sanity tests.
   timechart       Tool to visualize total system behavior during a workload
   top             System profiling tool.
   version         display the version of perf binary
   probe           Define new dynamic tracepoints
   trace           strace inspired tool

 See 'perf help COMMAND' for more information on a specific command.
```
> perf stat -h
```shell
root@f5c6cb9f45b1:/# perf stat -h 

 Usage: perf stat [<options>] [<command>]

    -a, --all-cpus        system-wide collection from all CPUs
    -A, --no-aggr         disable CPU count aggregation
    -B, --big-num         print large numbers with thousands' separators
    -C, --cpu <cpu>       list of cpus to monitor in system-wide
    -D, --delay <n>       ms to wait before starting measurement after program start (-1: start with events disabled)
    -d, --detailed        detailed run - start a lot of events
    -e, --event <event>   event selector. use 'perf list' to list available events
    -G, --cgroup <name>   monitor event in cgroup name only
    -g, --group           put the counters into a counter group
    -I, --interval-print <n>
                          print counts at regular interval in ms (overhead is possible for values <= 100ms)
    -i, --no-inherit      child tasks do not inherit counters
    -M, --metrics <metric/metric group list>
                          monitor specified metrics or metric groups (separated by ,)
    -n, --null            null run - dont start any counters
    -o, --output <file>   output file name
    -p, --pid <pid>       stat events on existing process id
    -r, --repeat <n>      repeat command and print average + stddev (max: 100, forever: 0)
    -S, --sync            call sync() before starting a run
    -t, --tid <tid>       stat events on existing thread id
    -T, --transaction     hardware transaction statistics
    -v, --verbose         be more verbose (show counter open errors, etc)
    -x, --field-separator <separator>
                          print counts with custom separator
        --all-kernel      Configure all used events to run in kernel space.
        --all-user        Configure all used events to run in user space.
        --append          append to the output file
        --control <fd:ctl-fd[,ack-fd] or fifo:ctl-fifo[,ack-fifo]>
                          Listen on ctl-fd descriptor for command to control measurement ('enable': enable events, 'disable': disable events).
                          Optionally send control command completion ('ack\n') to ack-fd descriptor.
                          Alternatively, ctl-fifo / ack-fifo will be opened and used as ctl-fd / ack-fd.
        --filter <filter>
                          event filter
        --for-each-cgroup <name>
                          expand events for each cgroup
        --interval-clear  clear screen in between new interval
        --interval-count <n>
                          print counts for fixed number of times
        --iostat[=<default>]
                          measure I/O performance metrics provided by arch/platform
        --log-fd <n>      log output to fd, instead of stderr
        --metric-no-group
                          don't group metric events, impacts multiplexing
        --metric-no-merge
                          don't try to share events between metrics in a group
        --metric-only     Only print computed metrics. No raw values
        --no-csv-summary  don't print 'summary' for CSV summary output
        --no-merge        Do not merge identical named events
        --per-core        aggregate counts per physical processor core
        --per-die         aggregate counts per processor die
        --per-node        aggregate counts per numa node
        --per-socket      aggregate counts per processor socket
        --per-thread      aggregate counts per thread
        --percore-show-thread
                          Use with 'percore' event qualifier to show the event counts of one hardware thread by sum up total hardware threads of same physical core
        --post <command>  command to run after to the measured command
        --pre <command>   command to run prior to the measured command
        --quiet           don't print output (useful with record)
        --scale           Use --no-scale to disable counter scaling for multiplexing
        --smi-cost        measure SMI cost
        --summary         print summary for interval mode
        --table           display details about each run (only with -r option)
        --td-level <n>    Set the metrics level for the top-down statistics (0: max level)
        --timeout <n>     stop workload and print counts after a timeout period in ms (>= 10ms)
        --topdown         measure top-down statistics
```

# Command Examples

+ The first step is collecting data. We need to collect sched_stat and sched_switch events. Sched_stat events are not enough, because they are generated in the context of a task, which wakes up a target task (e.g. releases a lock). We need the same event but with a call-chain of the target task. This call-chain can be extracted from a previous sched_switch event.
	+ `-e sched:sched_stat_sleep -e sched:sched_switch -e sched:sched_process_exit`: These are options that specify which events perf should record:
		+ `sched:sched_stat_sleep`: This event is recorded when a task goes to sleep, or in other words, when it is not runnable, waiting for some resource or event.
		+ `sched:sched_switch`: This records context switch information, which occurs when the kernel switches the CPU from running one task to another.
		+ `sched:sched_process_exit`: This captures information about process termination/exiting.
	+ `-g`: This option enables call graph (stack frame) recording. With call graphs, you can see the path of function calls that lead to the event, which is helpful for understanding the runtime behavior of the system and identifying performance bottlenecks.
	+ `-o ~/perf.data.raw`: This directs perf to output the raw profiling data to a file named `perf.data.raw` in the user's home directory (~). By default, perf writes to a file named perf.data in the current directory.
	+ `~/foo`: This part of the command indicates the actual workload or command to be profiled is an executable named `foo` located in the user's home directory. The `perf record` command will run this executable as the workload to profile.
+ The second step is merging sched_start and sched_switch events. It can be done with help of "perf inject -s".
	+  post-process a perf.data file that was previously recorded with perf record. 
	+  It allows additional data to be injected into an existing perf.data file or it can filter or transform trace data in various ways.
	+  `-v`: This is the verbose flag. When you add -v to perf commands, it runs in verbose mode, giving more detailed output about what the command is doing.
	+ `-s`: This option tells perf inject to build synthesized events such as mmap events to resolve symbolic names for addresses within the recorded data. It is particularly useful when the data file is to be analyzed on a different machine or at a different time when the mapped memory addresses may differ.
	+ `-i ~/perf.data.raw`: Specifies the input file from which to read perf data. This would be the raw perf data file you've named `perf.data.raw` in your home directory.
	+ `-o ~/perf.data`: Defines the output file to which the resulting perf data should be written. In this case, it's a file named `perf.data` 
	+  It's a way to enrich or transform your profiling data post-capture without having to recapture it, which can save time and effort, especially if you forgot to record some types of data initially.
+ The third step is used for generating a performance report from the data collected by `perf record`. Here is a breakdown of what each part of the command does:
	+ `--stdio`: This option tells perf report to use standard I/O (input/output) for displaying the report. It is the textual output mode that prints the report to the terminal (or to a file if redirected). Without this, perf report might use a TUI (Text User Interface) which provides an interactive text-based graphical interface.
	+ `-i ~/perf.data`:  The `-i` flag specifies the input file for the perf report command, which in this case is `~/perf.data`. It tells perf report to generate the report based on the `perf.data` file located in the home directory.
	+ `--show-total-period`: This option includes the "total period" in the report output, which is essentially the total time (or count) that each function (or sample point) was recorded as running during the profiling. It helps you understand how long or how often a particular function was active in relation to the total time of the collected data.
```shell
$ ./perf record -e sched:sched_stat_sleep -e sched:sched_switch  -e sched:sched_process_exit -g -o ~/perf.data.raw ~/foo
$ ./perf inject -v -s -i ~/perf.data.raw -o ~/perf.data
$ ./perf report --stdio --show-total-period -i ~/perf.data
```

# Examples of the overhead calculation

The overhead can be shown in two columns as 'Children' and 'Self' when perf collects callchains. 
+ The 'self' overhead is simply calculated by adding all period values of the entry - usually a function (symbol). This is the value that perf shows traditionally and sum of all the 'self' overhead values should be 100%.
+ The 'children' overhead is calculated by adding all period values of the child functions so that it can show the total overhead of the higher level functions even if they don't directly execute much. 'Children' here means functions that are called from another (parent) function.

It might be confusing that the sum of all the 'children' overhead values exceeds 100% since each of them is already an accumulation of 'self' overhead of its child functions. But with this enabled, users can find which function has the most overhead even if samples are spread over the children.

Consider the following example; there are three functions like below.

```c
void foo(void) {
    /* do something */
}

void bar(void) {
    /* do something */
    foo();
}

int main(void) {
    bar()
    return 0;
}
```

+ In this case 'foo' is a child of 'bar', and 'bar' is an immediate child of 'main' so 'foo' also is a child of 'main'. In other words, 'main' is a parent of 'foo' and 'bar', and 'bar' is a parent of 'foo'.

+ Suppose all samples are recorded in 'foo' and 'bar' only. When it's recorded with callchains the output will show something like below in the usual (self-overhead-only) output of perf report:
	```shell
	Overhead  Symbol
	........  .....................
	  60.00%  foo
			  |
			  --- foo
				  bar
				  main
				  __libc_start_main

	  40.00%  bar
			  |
			  --- bar
				  main
				  __libc_start_main
	```
+ When the `--children` option is enabled, the 'self' overhead values of child functions (i.e. 'foo' and 'bar') are added to the parents to calculate the 'children' overhead. In this case the report could be displayed as:
	```shell
	Children      Self  Symbol
	........  ........  ....................
	 100.00%     0.00%  __libc_start_main
			  |
			  --- __libc_start_main

	 100.00%     0.00%  main
			  |
			  --- main
				  __libc_start_main

	 100.00%    40.00%  bar
			  |
			  --- bar
				  main
				  __libc_start_main

	  60.00%    60.00%  foo
			  |
			  --- foo
				  bar
				  main
				  __libc_start_main
	```
In the above output, the 'self' overhead of 'foo' (60%) was add to the 'children' overhead of 'bar', 'main' and '__libc_start_main'. Likewise, the 'self' overhead of 'bar' (40%) was added to the 'children' overhead of 'main' and '__libc_start_main'.

So '__libc_start_main' and 'main' are shown first since they have same (100%) 'children' overhead (even though they have zero 'self' overhead) and they are the parents of 'foo' and 'bar'.

Since v3.16 the 'children' overhead is shown by default and the output is sorted by its values. The 'children' overhead is disabled by specifying --no-children option on the command line or by adding 'report.children = false' or 'top.children = false' in the perf config file.

# Flame Graph

Flame Graphs can be produced from perf_events profiling data using the FlameGraph tools software. This visualizes the same data you see in perf report, and works with any perf.data file that was captured with stack traces (-g).



```shell
Log in as a root user. 
git clone https://github.com/brendangregg/FlameGraph  # or download it from GitHub
cd FlameGraph 
perf record -F 99 -a -g -- sleep 60
perf script | ./stackcollapse-perf.pl > out.perf-folded
./flamegraph.pl out.perf-folded > perf.svg
firefox perf.svg  # or any other web browser, etc.
```
```shell
perf script -i hellotest.data | ~/Github/FlameGraph/stackcollapse-perf.pl > hellotest.perf-folded
~/Github/FlameGraph/flamegraph.pl perfhellotest.perf-folded > perfhellotest.svg
```
![ResizeMemory overhead ratio](/commons/images/3141911-20231106134358289-1778534097.png)

# References

+ [perf.wiki.kernel.org](https://perf.wiki.kernel.org/index.php/Tutorial#Linux_sourcecode)
+ [brendangregg.com](https://www.brendangregg.com/perf.html)
+ [Github examples](https://github.com/brendangregg/perf-tools)