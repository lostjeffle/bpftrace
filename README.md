[![Build Status](https://travis-ci.org/ajor/bpftrace.svg?branch=master)](https://travis-ci.org/ajor/bpftrace)

# BPFtrace

BPFtrace is a high-level tracing language for Linux enhanced Berkeley Packet Filter (eBPF) available in recent Linux kernels (4.x). BPFtrace uses LLVM as a backend to compile scripts to BPF-bytecode and makes use of [BCC](https://github.com/iovisor/bcc) for interacting with the Linux BPF system, as well as existing Linux tracing capabilities: kernel dynamic tracing (kprobes), user-level dynamic tracing (uprobes), and tracepoints. The BPFtrace language is inspired by awk and C, and predecessor tracers such as DTrace and SystemTap. BPFtrace was created by [Alastair Robertson](https://github.com/ajor).

For instructions on building BPFtrace, see [INSTALL.md](INSTALL.md). There is also a [Reference Guide](docs/reference_guide.md) and [One-Liner Tutorial](docs/tutorial_one_liners.md).

## Examples

Count system calls:
```
tracepoint:syscalls:sys_enter_*
{
  @[name] = count();
}
```
```
Attaching 320 probes...
^C

...
@[tracepoint:syscalls:sys_enter_futex]: 50
@[tracepoint:syscalls:sys_enter_newfstat]: 52
@[tracepoint:syscalls:sys_enter_clock_gettime]: 56
@[tracepoint:syscalls:sys_enter_perf_event_open]: 148
@[tracepoint:syscalls:sys_enter_select]: 156
@[tracepoint:syscalls:sys_enter_dup]: 291
@[tracepoint:syscalls:sys_enter_read]: 308
@[tracepoint:syscalls:sys_enter_bpf]: 310
@[tracepoint:syscalls:sys_enter_open]: 363
@[tracepoint:syscalls:sys_enter_ioctl]: 571
@[tracepoint:syscalls:sys_enter_dup2]: 580
@[tracepoint:syscalls:sys_enter_close]: 998
```

Produce a histogram of amount of time (in nanoseconds) spent in the `read()` system call:
```
tracepoint:syscalls:sys_enter_read
{
  @start[tid] = nsecs;
}

tracepoint:syscalls:sys_exit_read / @start[tid] /
{
  @times = hist(nsecs - @start[tid]);
  delete(@start[tid]);
}
```
```
Attaching 2 probes...
^C

@start[9134]: 6465933686812

@times:
[0, 1]                 0 |                                                    |
[2, 4)                 0 |                                                    |
[4, 8)                 0 |                                                    |
[8, 16)                0 |                                                    |
[16, 32)               0 |                                                    |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)           326 |@                                                   |
[512, 1k)           7715 |@@@@@@@@@@@@@@@@@@@@@@@@@@                          |
[1k, 2k)           15306 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[2k, 4k)             609 |@@                                                  |
[4k, 8k)             611 |@@                                                  |
[8k, 16k)            438 |@                                                   |
[16k, 32k)            59 |                                                    |
[32k, 64k)            36 |                                                    |
[64k, 128k)            5 |                                                    |
```

Print paths of any files opened along with the name of process which opened them:
```
kprobe:sys_open
{
  printf("%s: %s\n", comm, str(arg0))
}
```
```
Attaching 1 probe...
git: .git/objects/70
git: .git/objects/pack
git: .git/objects/da
git: .git/objects/pack
git: /etc/localtime
systemd-journal: /var/log/journal/72d0774c88dc4943ae3d34ac356125dd
DNS Res~ver #15: /etc/hosts
DNS Res~ver #16: /etc/hosts
DNS Res~ver #15: /etc/hosts
^C
```

Whole system profiling (TODO make example check if kernel is on-cpu before recording):
```
profile:hz:99
{
  @[stack] = count()
}
```
```
Attaching 1 probe...
^C

...
@[
_raw_spin_unlock_irq+23
finish_task_switch+117
__schedule+574
schedule_idle+44
do_idle+333
cpu_startup_entry+113
start_secondary+344
verify_cpu+0
]: 83
@[
queue_work_on+41
tty_flip_buffer_push+43
pty_write+83
n_tty_write+434
tty_write+444
__vfs_write+55
vfs_write+177
sys_write+85
entry_SYSCALL_64_fastpath+26
]: 97
@[
cpuidle_enter_state+299
cpuidle_enter+23
call_cpuidle+35
do_idle+394
cpu_startup_entry+113
rest_init+132
start_kernel+1083
x86_64_start_reservations+41
x86_64_start_kernel+323
verify_cpu+0
]: 150
```

## Tools

bpftrace contains various tools, which also serve as examples of programming in the bpftrace language.

- tools/[bashreadline.bt](tools/bashreadline.bt): Print entered bash commands system wide. [Examples](tools/bashreadline_example.txt).
- tools/[biosnoop.bt](tools/biosnoop.bt): Block I/O tracing tool, showing per I/O latency. [Examples](tools/biosnoop_example.txt).
- tools/[capable.bt](tools/capable.bt): Trace security capabilitiy checks. [Examples](tools/capable_example.txt).
- tools/[cpuwalk.bt](tools/cpuwalk.bt): Sample which CPUs are executing processes. [Examples](tools/cpuwalk_example.txt).
- tools/[execsnoop.bt](tools/execsnoop.bt): Trace new processes via exec() syscalls. [Examples](tools/execsnoop_example.txt).
- tools/[gethostlatency.bt](tools/gethostlatency.bt): Show latency for getaddrinfo/gethostbyname[2] calls. [Examples](tools/gethostlatency_example.txt).
- tools/[loads.bt](tools/loads.bt): Print load averages. [Examples](tools/loads_example.txt).
- tools/[pidpersec.bt](tools/pidpersec.bt): Count new procesess (via fork). [Examples](tools/pidpersec_example.txt).
- tools/[vfscount.bt](tools/vfscount.bt): Count VFS calls. [Examples](tools/vfscount_example.txt).
- tools/[vfsstat.bt](tools/vfsstat.bt): Count some VFS calls, with per-second summaries. [Examples](tools/vfsstat_example.txt).
- tools/[xfsdist.bt](tools/xfsdist.bt): Summarize XFS operation latency distribution as a histogram. [Examples](tools/xfsdist_example.txt).

For more eBPF observability tools, see [bcc tools](https://github.com/iovisor/bcc#tools).

## Probe types
<center><a href="images/bpftrace_probes_2018.png"><img src="images/bpftrace_probes_2018.png" border=0 width=700></a></center>

### kprobes
Attach a BPFtrace script to a kernel function, to be executed when that function is called:

`kprobe:vfs_read { ... }`

### uprobes
Attach script to a userland function:

`uprobe:/bin/bash:readline { ... }`

### tracepoints
Attach script to a statically defined tracepoint in the kernel:

`tracepoint:sched:sched_switch { ... }`

Tracepoints are guaranteed to be stable between kernel versions, unlike kprobes.

### software
Attach script to kernel software events, executing once every provided count or use a default:

`software:faults:100`
`software:faults:`

### hardware
Attach script to hardware events (PMCs), executing once every provided count or use a default:

`hardware:cache-references:1000000`
`hardware:cache-references:`

### profile
Run the script on all CPUs at specified time intervals:

`profile:hz:99 { ... }`

`profile:s:1 { ... }`

`profile:ms:20 { ... }`

`profile:us:1500 { ... }`

### interval
Run the script once per interval, for printing interval output:

`interval:s:1 { ... }`

`interval:ms:20 { ... }`

### Multiple attachment points
A single probe can be attached to multiple events:

`kprobe:vfs_read,kprobe:vfs_write { ... }`

### Wildcards
Some probe types allow wildcards to be used when attaching a probe:

`kprobe:vfs_* { ... }`

### Predicates
Define conditions for which a probe should be executed:

`kprobe:sys_open / uid == 0 / { ... }`

## Builtins
The following variables and functions are available for use in bpftrace scripts:

Variables:
- `pid` - Process ID (kernel tgid)
- `tid` - Thread ID (kernel pid)
- `uid` - User ID
- `gid` - Group ID
- `nsecs` - Nanosecond timestamp
- `cpu` - Processor ID
- `comm` - Process name
- `stack` - Kernel stack trace
- `ustack` - User stack trace
- `arg0`, `arg1`, ... etc. - Arguments to the function being traced
- `retval` - Return value from function being traced
- `func` - Name of the function currently being traced
- `name` - Full name of the probe
- `curtask` - Current task_struct as a u64.
- `rand` - Random number of type u32.

Functions:
- `hist(int n)` - Produce a log2 histogram of values of `n`
- `lhist(int n, int min, int max, int step)` - Produce a linear histogram of values of `n`
- `count()` - Count the number of times this function is called
- `sum(int n)` - Sum this value
- `min(int n)` - Record the minimum value seen
- `max(int n)` - Record the maximum value seen
- `avg(int n)` - Average this value
- `stats(int n)` - Return the count, average, and total for this value
- `delete(@x)` - Delete the map element passed in as an argument
- `str(char *s)` - Returns the string pointed to by `s`
- `printf(char *fmt, ...)` - Print formatted to stdout
- `print(@x[, int top [, int div]])` - Print a map, with optional top entry count and divisor
- `clear(@x)` - Delet all key/values from a map
- `sym(void *p)` - Resolve kernel address
- `usym(void *p)` - Resolve user space address (incomplete)
- `kaddr(char *name)` - Resolve kernel symbol name
- `reg(char *name)` - Returns the value stored in the named register
- `join(char *arr[])` - Prints the string array
- `time(char *fmt)` - Print the current time
- `exit()` - Quit bpftrace

See the [Reference Guide](docs/reference_guide.md) for more detail.

## Internals

<center><a href="images/bpftrace_internals_2018.png"><img src="images/bpftrace_internals_2018.png" border=0 width=700></a></center>

bpftrace employes various techniques for efficiency, minimizing the instrumentation overhead. Summary statistics are stored in kernel BPF maps, which are asynchronously copied from kernel to user-space, only when needed. Other data, and asynchronous actions, are passed from kernel to user-space via the perf output buffer.
