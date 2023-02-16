---
layout: post
title: tracing in linux
date: '2018-10-04T09:39:10-0500'
author: arges
tags:
- linux
- kernel
- tracing
modified_time: '2018-10-04T09:39:14-0500'
---

Tracing in Linux is robust, flexible and has many options. So what mechanisms
exist both in kernel and user space for instrumenting the kernel? In addition
what is available in which kernel version, and what requirements does each
technology require? Here we'll mostly focus on syscall tracing.

Overview
--------

A great overview on Linux tracing systems can be found [here][5]. Overall
thinking about tracing systems as data sources (in kernel), and frontends help
explain why there are so many tools. Some data sources are 'tracepoints' while
other are 'probes'. Tracepoints are low-overhead and can be activated or
deactivated, they are compiled into the kernel so they cannot be dynamically
instrumented. Probes are very flexible, but require larger overhead as you are
replacing a 'nop' instruction with a branch in order to redirect execution of
probed function. In addition you must know the function name and can only
instrument 'real' (non-inlined) functions. Some good slides describing this are
[here][6]. Other overviews of tracing technologies can be found [here][7].

Tracepoint
----------

Linux kernel [tracepoints][3] allow for a _tracepoint_ to be embedded in
various critical kernel functions. A _tracepoint_ can even be instrumented by a
new kernel module to execute custom code on hitting a _tracepoint_. This point
makes it very extensible.

An advantage of tracepoints is they are defined in the code rather than hooking
into a function name. This is important because function names change between
kernel versions as well as parameters.

Tracepoints are defined in the Linux kernel like this:
```c
   TRACE_EVENT(sched_switch,

	TP_PROTO(struct rq *rq, struct task_struct *prev,
		 struct task_struct *next),

	TP_ARGS(rq, prev, next),

	TP_STRUCT__entry(
		__array(	char,	prev_comm,	TASK_COMM_LEN	)
		__field(	pid_t,	prev_pid			)
		__field(	int,	prev_prio			)
		__field(	long,	prev_state			)
		__array(	char,	next_comm,	TASK_COMM_LEN	)
		__field(	pid_t,	next_pid			)
		__field(	int,	next_prio			)
	),

	TP_fast_assign(
		memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
		__entry->prev_pid	= prev->pid;
		__entry->prev_prio	= prev->prio;
		__entry->prev_state	= prev->state;
		memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
		__entry->next_pid	= next->pid;
		__entry->next_prio	= next->prio;
	),

	TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s ==> next_comm=%s next_pid=%d next_prio=%d",
		__entry->prev_comm, __entry->prev_pid, __entry->prev_prio,
		__entry->prev_state ?
		  __print_flags(__entry->prev_state, "|",
				{ 1, "S"} , { 2, "D" }, { 4, "T" }, { 8, "t" },
				{ 16, "Z" }, { 32, "X" }, { 64, "x" },
				{ 128, "W" }) : "R",
		__entry->next_comm, __entry->next_pid, __entry->next_prio)
   );
```

The name `sched_switch` is the name of the tracepoint, and this macro creates a
function called `trace_sched_switch` which needs to be added into the desired
Linux function to be traced. More info can be found [here][4].

This interface has been in the kernel a long time and new tracepoints are being
added with every release. More fundamental tracepoints have been added in much
older kernels. For example, the `trace_sched_switch` function was added the
kernel by `0a16b60758433` which was released in v2.6.28. The
`trace_syscall_enter` and `trace_syscall_exit` functions were added by
`a871bd33a6c` which was released in v2.6.32.

These tracepoints can be accessed via `ftrace` tracefs filesystem, for example:
```bash
# echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_open
# cat /sys/kernel/debug/tracing/trace_pipe
 systemd-journal-365   [000] .... 88393.862127: sys_open(filename: 55f45105d8a0, flags: 80042, mode: 1a0)
 systemd-journal-365   [000] .... 88393.862597: sys_open(filename: 7fffdddb2cf0, flags: 90800, mode: 7fffdddb2d21)
 systemd-journal-365   [000] .... 88393.862665: sys_open(filename: 7fffdddb2f40, flags: 90800, mode: 0)
            bash-18684 [000] .... 88393.864513: sys_open(filename: 1158488, flags: 90800, mode: 118d4c0)
            bash-18199 [000] .... 88393.864996: sys_open(filename: 119b008, flags: 2c1, mode: 180)
            bash-18199 [000] .... 88393.865052: sys_open(filename: 119b008, flags: 0, mode: 180)
            bash-18685 [000] .... 88393.866117: sys_open(filename: 1158488, flags: 90800, mode: 118d4c0)
```

If you prefer a front-end tool you can use perf to capture specific syscalls for a command:
```
# perf trace --no-syscalls --event 'syscalls:sys_enter_connect' ping 8.8.8.8 -c 1
     1.704 syscalls:sys_enter_connect:fd: 0x00000004, uservaddr: 0x7ffd0bf83e50, addrlen: 0x00000010)
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=120 time=19.6 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 19.616/19.616/19.616/0.000 ms
```

To trace all syscalls on a system use the following:
```
# cd /sys/kernel/debug/tracing/
# echo 1 > events/syscalls/enable
# cat trace_pipe
bash-18199 [000] .... 84627.885591: sys_rt_sigaction -> 0x0
bash-18199 [000] .... 84627.885595: sys_rt_sigaction(sig: 2, act: 7ffd9fa6d780, oact: 7ffd9fa6d820, sigsetsize: 8)
bash-18199 [000] .... 84627.885596: sys_rt_sigaction -> 0x0
bash-18199 [000] .... 84627.885682: sys_open(filename: 1125148, flags: 241, mode: 1b6)
bash-18199 [000] .... 84627.885705: sys_open -> 0x3
bash-18199 [000] .... 84627.885709: sys_fcntl(fd: 1, cmd: 1, arg: 0)
bash-18199 [000] .... 84627.885720: sys_fcntl -> 0x0
```

Ftrace
------

To see tracepoints we used the tracefs filesystem which is also called
`ftrace`. See the [ftrace][1] document for more exhaustive use of ftrace in
userspace.

Ftrace itself can use various data sources such as tracepoints and even
function trampolines. The function trampolines are instrumented and configured
via the kernel and utilize gcc's `-pg` flag for function profiling. This is
very powerful and the basis of kernel livepatching.

Kernel requirements depend on what information you want to get, for syscalls
`HAVE_SYSCALL_TRACEPOINTS` (formerly `HAVE_FTRACE_SYSCALLS`) needs to be
implemented for tracing to work with ftrace. This is architecture
dependent, and for x86 was implemented in v2.6.30. More details can be found in
the [design][2] document. In addition much of the code for handling syscall
tracing for ftrace is in `kernel/trace/trace_syscalls.c`.

Kprobes
-------

Another interface that can be used via tracefs.

Example usage:
```
# echo 1 > /sys/kernel/debug/tracing/tracing_on
# echo 'p:myprobe do_sys_open dfd=%ax filename=%dx flags=%cx mode=+4($stack)' >> /sys/kernel/debug/tracing/kprobe_events
# echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
# cat /sys/kernel/debug/tracing/trace
# tracer: nop
#
# entries-in-buffer/entries-written: 4/951007   #P:1
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
             cat-18934 [000] d... 92669.062797: myprobe: (do_sys_open+0x0/0x2a0) dfd=0xffffffff81216050 filename=0x88000 flags=0x1 m
ode=0xffffffffffffffff
             cat-18934 [000] d... 92669.062846: myprobe: (do_sys_open+0x0/0x2a0) dfd=0xffffffff81216050 filename=0x88000 flags=0x316
8 mode=0x18f0c660ffffffff
             cat-18934 [000] d... 92669.063316: myprobe: (do_sys_open+0x0/0x2a0) dfd=0xffffffff81216050 filename=0x88000 flags=0x0 m
ode=0x18f0cae0ffffffff
             cat-18934 [000] d... 92669.063423: myprobe: (do_sys_open+0x0/0x2a0) dfd=0xffffffff81216050 filename=0x8000 flags=0x0 mo
de=0x1000ffffffff
```

The nice thing here is you can print custom parameters, and potentially use
even more sophisticated parsing of said parameters.

eBPF
----

Extended Berkley Packet Filter has been usable for tracing since v4.1, but many
improvements are still ongoing. To use eBPF you write a C program that gets
loaded into the kernel via the BPF syscall. The program has to use specific
functions and cannot have loops. The kernel verifies this before allowing the
code to be executed in kernel. The advantage here is that writing a custom
kernel module is not required and one can attach to tracepoints and do advanced
filtering and instrumentation.

An example of tracing open syscalls is [here][8].

More information can be found [here][9].

Frontends
---------

SystemTap, LTTng, and sysdig are all other tracing tools that utilize kernel
functionality ranging from tracepoints to kprobes.

SystemTap makes it easy to write kprobes for various functions and has its own
high level syntax that can be compiled down into something usable by the
kernel.

LTTng ships its own kernel module which instruments tracepoints (among many
other things). The [webpage][10] has very extensive documentation. Very cool
discussion on their ring buffer design.

```
# lttng create my-kernel-session --output=/tmp/my-kernel-trace
# lttng list --kernel
# lttng enable-event --kernel --syscall open,close
# lttng start
# lttng stop
# lttng destroy
# babeltrace /tmp/my-kernel-trace
[18:38:35.107903286] (+0.000003918) test syscall_exit_open: { cpu_id = 0 }, { ret = 3 }
[18:38:35.107920793] (+0.000017507) test syscall_entry_close: { cpu_id = 0 }, { fd = 3 }
[18:38:35.107921474] (+0.000000681) test syscall_exit_close: { cpu_id = 0 }, { ret = 0 }
[18:38:35.108280899] (+0.000359425) test syscall_entry_open: { cpu_id = 0 }, { filename = "/usr/lib/python3.5/__pycache__/struct.cpy
thon-35.pyc", flags = 524288, mode = 438 }
[18:38:35.108284573] (+0.000003674) test syscall_exit_open: { cpu_id = 0 }, { ret = 3 }
[18:38:35.108299173] (+0.000014600) test syscall_entry_close: { cpu_id = 0 }, { fd = 3 }
```

Sysdig is a commercial product that also has an open-source component which is
the kernel module and some user-space tooling. The module registers functions
on various kernel tracepoints, looks up additional information in the kernel,
and injects this into a ring buffer exposed into userspace.

[1]: https://www.kernel.org/doc/Documentation/trace/ftrace.txt
[2]: https://www.kernel.org/doc/Documentation/trace/ftrace-design.txt
[3]: https://www.kernel.org/doc/Documentation/trace/tracepoints.txt
[4]: https://lwn.net/Articles/379903/
[5]: https://jvns.ca/blog/2017/07/05/linux-tracing-systems/
[6]: https://blog.linuxplumbersconf.org/2014/ocw/system/presentations/1773/original/ftrace-kernel-hooks-2014.pdf
[7]: http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html
[8]: https://github.com/torvalds/linux/blob/master/samples/bpf/syscall_tp_kern.c
[9]: https://lwn.net/Articles/740157/
[10]: https://lttng.org/docs
