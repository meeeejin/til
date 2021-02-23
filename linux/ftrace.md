# How to use ftrace

## Setting up

Typically, `ftrace` is mounted at `/sys/kernel/debug`. When `ftrace` is configured, it will create its own directory called *tracing* within the Debugfs file system:

```bash
$ cd /sys/kernel/debug/tracing/
$ ls
available_events            current_tracer         free_buffer               max_graph_depth  saved_cmdlines_size  set_ftrace_pid      stack_trace_filter  trace_marker_raw     tracing_on
available_filter_functions  dynamic_events         function_profile_enabled  options          saved_tgids          set_graph_function  synthetic_events    trace_options        tracing_thresh
available_tracers           dyn_ftrace_total_info  hwlat_detector            per_cpu          set_event            set_graph_notrace   timestamp_mode      trace_pipe           uprobe_events
buffer_percent              enabled_functions      instances                 printk_formats   set_event_pid        snapshot            trace               trace_stat           uprobe_profile
buffer_size_kb              error_log              kprobe_events             README           set_ftrace_filter    stack_max_size      trace_clock         tracing_cpumask
buffer_total_size_kb        events                 kprobe_profile            saved_cmdlines   set_ftrace_notrace   stack_trace         trace_marker        tracing_max_latency
```

## Function tracing

1. To find out which tracers are available, simply cat the `available_tracers` file in the `tracing` directory:

```bash
$ cat available_tracers
hwlat blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function nop
```

2. To enable the function graph tracer, echo *function_graph* into the `current_tracer` file:

```bash
$ cat current_tracer
nop
$ echo function_graph > current_tracer
$ cat current_tracer
function_graph
```

3. To extract traces based on specific functions, use the `available_filter_functions` file:

```bash
$ cat available_filter_functions
...
$ echo "*pwrite*" > set_ftrace_filter
$ echo "*pread*" > set_ftrace_filter
$ cat set_ftrace_filter
__ia32_compat_sys_x86_pread
__x32_compat_sys_x86_pread
check_spread.isra.83.part.84
cpuset_mem_spread_node
cpuset_update_task_spread_flag
cpuset_slab_spread_node
do_compat_preadv64
__ia32_compat_sys_preadv64
__x32_compat_sys_preadv64
__ia32_compat_sys_preadv
__x32_compat_sys_preadv
__ia32_compat_sys_preadv64v2
__ia32_compat_sys_preadv2
__x32_compat_sys_preadv64v2
__x32_compat_sys_preadv2
ksys_pread64
__x64_sys_pread64
__ia32_sys_pread64
do_preadv
__x64_sys_preadv
__ia32_sys_preadv
__ia32_sys_preadv2
__x64_sys_preadv2
zpodd_zpready
$ cat trace
```

If you want to see all the results, initialize the `available_filter_functions` file:

```bash
$ echo " " > set_ftrace_filter
```

Conversely, you may not want to trace a specific function. In that case, you can exclude the function from tracing by adding the function name to the `set_ftrace_notrace` file:

```bash
$ echo "*abc*" > set_ftrace_notrace
```

## PID filtering

To trace only functions called in a specific process, add PID to the `set_ftrace_pid` file:

```bash
$ echo 23321 > set_ftrace_pid
```

## Starting and stopping the trace

The file `tracing_on` is used to disable/enable the ring buffer from recording data:

```bash
# Disable the ftrace ring buffer
$ echo 0 > tracing_on

# Enable the ftrace ring buffer 
$ echo 1 > tracing_on
```

## Reference
 - [Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)