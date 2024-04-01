---
type: post
title: bpf for linux tracing
date: '2019-03-21T11:54:37-0500'
author: arges
tags:
- linux
- kernel
- bpf
- tracing
modified_time: '2019-03-21T11:54:41-0500'
---

Overview
--------

Originally called eBPF for extended Berkeley Packet Filter, BPF is an in-kernel
bytecode interpreter that is very multi-purpose. In this post, I'm going to
focus on its tracing capabilities.

To use BPF one can use the bpf [syscall][1]. This provides a way to both load
BPF bytecode programs and also create memory maps for sharing data between
kernel and user space.

Because of the complexity, using bpf in a DIY method becomes a bit complex.
There are many ready to use tools such as [BCC][3] and libbpf that make usage
way easier.

Ready to Use BPF via BCC
========================

The `bcc` project makes it easy to use BPF via a python program.

To use, first install dependencies:
```
sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
```

Next, create a python file named trace.py:
```
#!/usr/bin/python
from bcc import BPF

b = BPF(text="""
TRACEPOINT_PROBE(raw_syscalls, sys_exit) {
    u64 pid_tgid = bpf_get_current_pid_tgid();
    u64 uid_gid = bpf_get_current_uid_gid();
    bpf_trace_printk("SYS_EXIT tgid:%ld pid:%ld uid:%ld\\n", pid_tgid >> 32 & 0xffff, pid_tgid & 0xffff, uid_gid >> 32 & 0xf
fff);
}
""")

print("%-18s %-16s %-6s %s" % ("TIME(s)", "COMM", "PID", "EVENT"))
while 1:
    try:
        (task, pid, cpu, flags, ts, msg) = b.trace_fields()
    except ValueError:
        continue
    print("%-18.9f %-16s %-6d %s" % (ts, task, pid, msg))
```

Then run it:
```
sudo python trace.py
```

And magically it spits out text like:
```
32508.991461000   python           21303  00000001: SYS_EXIT tgid:21303 pid:21303 uid:0
32508.991468000   python           21303  00000001: SYS_EXIT tgid:21303 pid:21303 uid:0
32508.991475000   python           21303  00000001: SYS_EXIT tgid:21303 pid:21303 uid:0
32508.991482000   python           21303  00000001: SYS_EXIT tgid:21303 pid:21303 uid:0
32508.991490000   python           21303  00000001: SYS_EXIT tgid:21303 pid:21303 uid:0
```

All this is attaching to a raw tracepoint and printing out some data related to
that process. The python library is taking the text and compiling it to BPF
bytecode and handling allocation of maps. Then its calling the `bpf` syscall to
create maps and insert the program. And in addition it is taking the data from
the map and printing it to the screen.

An strace of the python program reveals the following:
```
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_TRACEPOINT, insn_cnt=32, insns=0x7f259141b7d0, license="GPL", log_level=0, log_size=0, log_buf=0, kern_version=266002, prog_flags=0, ...}, 64) = 3
```

DIY BPF
=======

There are many ways to create a BPF program. One method as described above uses
C code with many helper functions and then compiles to bytecode and is
inserted. Another method would be writing bytecode directly using assembler
like macros:
```
BPF_MOV64_REG(BPF_REG_2, BPF_REG_10),       /* r2 = fp */
BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4),      /* r2 = r2 - 4 */
BPF_LD_MAP_FD(BPF_REG_1, map_fd),           /* r1 = map_fd */
BPF_CALL_FUNC(BPF_FUNC_map_lookup_elem),
```

Using C directly and having a compiler like `clang` with a BPF target makes
writing your own BPF program slightly easier. However the loading part becomes
a bit more complex. Once you've created a bpf object file you'll have to use
`libbpf` or write your own loader that can convert the ELF object into the
correct structures for the BPF syscall. These structures can be a program or
maps and each has different types depending on your BPF program.

For reference here are the program types (as of Linux v5.0):
```
enum bpf_prog_type {
	BPF_PROG_TYPE_UNSPEC,
	BPF_PROG_TYPE_SOCKET_FILTER,
	BPF_PROG_TYPE_KPROBE,
	BPF_PROG_TYPE_SCHED_CLS,
	BPF_PROG_TYPE_SCHED_ACT,
	BPF_PROG_TYPE_TRACEPOINT,
	BPF_PROG_TYPE_XDP,
	BPF_PROG_TYPE_PERF_EVENT,
	BPF_PROG_TYPE_CGROUP_SKB,
	BPF_PROG_TYPE_CGROUP_SOCK,
	BPF_PROG_TYPE_LWT_IN,
	BPF_PROG_TYPE_LWT_OUT,
	BPF_PROG_TYPE_LWT_XMIT,
	BPF_PROG_TYPE_SOCK_OPS,
	BPF_PROG_TYPE_SK_SKB,
	BPF_PROG_TYPE_CGROUP_DEVICE,
	BPF_PROG_TYPE_SK_MSG,
	BPF_PROG_TYPE_RAW_TRACEPOINT,
	BPF_PROG_TYPE_CGROUP_SOCK_ADDR,
	BPF_PROG_TYPE_LWT_SEG6LOCAL,
	BPF_PROG_TYPE_LIRC_MODE2,
	BPF_PROG_TYPE_SK_REUSEPORT,
	BPF_PROG_TYPE_FLOW_DISSECTOR,
};
```
And here are the map types:
```
enum bpf_map_type {
	BPF_MAP_TYPE_UNSPEC,
	BPF_MAP_TYPE_HASH,
	BPF_MAP_TYPE_ARRAY,
	BPF_MAP_TYPE_PROG_ARRAY,
	BPF_MAP_TYPE_PERF_EVENT_ARRAY,
	BPF_MAP_TYPE_PERCPU_HASH,
	BPF_MAP_TYPE_PERCPU_ARRAY,
	BPF_MAP_TYPE_STACK_TRACE,
	BPF_MAP_TYPE_CGROUP_ARRAY,
	BPF_MAP_TYPE_LRU_HASH,
	BPF_MAP_TYPE_LRU_PERCPU_HASH,
	BPF_MAP_TYPE_LPM_TRIE,
	BPF_MAP_TYPE_ARRAY_OF_MAPS,
	BPF_MAP_TYPE_HASH_OF_MAPS,
	BPF_MAP_TYPE_DEVMAP,
	BPF_MAP_TYPE_SOCKMAP,
	BPF_MAP_TYPE_CPUMAP,
	BPF_MAP_TYPE_XSKMAP,
	BPF_MAP_TYPE_SOCKHASH,
	BPF_MAP_TYPE_CGROUP_STORAGE,
	BPF_MAP_TYPE_REUSEPORT_SOCKARRAY,
	BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE,
	BPF_MAP_TYPE_QUEUE,
	BPF_MAP_TYPE_STACK,
};
```

As you can see there are many types of maps and programs. For tracing, KPROBE
and TRACEPOINT program types are most useful. For map types, the array and hash
types are useful depending on how you want to manage memory between user and
kernel space.

For a high-performance buffer one could use an array type. For even more
performance that array could have a single element and data could be read from
within offsets of that single element.

To share this map with userspace one can also use `bpf_obj_pin` and
`bpf_obj_get` to allow a userspace program to access the memory through a file
descriptor.


Features Supported in Linux Kernel Versions
-------------------------------------------

A very comprehensive document can be found [here][2] outlining the various
features and which kernel versions they were supported in. Below are a few
highlights.
 - BPF JIT compilation `3.16`
 - BPF syscall `3.18`
 - BPF attach to kprobes `4.1`
- `BPF_OBJ_PIN and BPF_OBJ_GET` `4.4`
- `BPF_MAP_TYPE_PERCPU_ARRAY` `4.6`
 - BPF attach to tracepoints `4.7`
 - BPF attach to raw tracepoints `4.17`

Limitations
-----------

Here I'll highlight some limitations versus using a kernel module to trace
system calls.

In general
==========
 - Cannot call exported kernel functions
 - Cannot access arbitrary memory (need to use helper functions)
 - Memory dereferencing requires `bpf_probe_read`
 - Performance overhead (although minimized with raw tracepoints)
 - Program must terminate and cannot have 'infinite' loops

FD to pathname mapping
======================

When tracing syscalls it is useful to determine the full path that a particular
syscall is operating on. For example of `openat`is called, on exit it returns
a file descriptor value which refers to a file. To get the fullpath we can
either construct it from 'dirfd', 'pathname', and 'cwd' depending on a few
rules or we can inquire the fullpath directly from the file descriptor. To
construct the fullpath we can use a function like `user_path_at`, however BPF
does not allow for calling of exported kernel functions. So this means
some duplication of lookup code here. [Others][7] have hit this point, and one
could use some shortcuts to get the name via dentry but it is far from
complete.

Peer Sockaddr Lookup
====================

Determining the peer socket during socket operations is critical for figuring
out the other side of a particular connection. When using BPF this isn't
impossible, but requires some traversal of various memory structures. It seems
that whenever you need to lookup a memory address one needs to call
`bpf_probe_read`. So something like ptr->-ptr->-ptr would need multiple calls
to deference.

BPF Tracing Performance
-----------------------

To understand how BPF can be used for tracing, it is important to review how
tracing can be done in the Linux kernel in the first place! Over the course
of the Linux kernel's development there have been various tracing subsystems
added to solve different problems.

More details can be found in a previous blog post [here][5].

Sysdig has both a kernel module ring-buffer and BPF tracing solution and has
done a comparison of performance based on various benchmarks. Here a benchmark
that runs many syscalls [shows][4] at worst 33% increase in instrumentation
overhead. The difficulty with this study is understanding this at a per syscall
level as some benchmarks may use more syscalls and thus have more overhead. In
addition where is the performance bottleneck? Is it in the copying of event
data into the ring buffer, or is it elsewhere? I believe this study was using
'normal' bpf tracepoints and not raw tracepoints.

In a patchset [here][6] there was a comparison of kprobes to tracepoints to
raw_tracepoints done which show that raw tracepoints have very close to base
performance (10% instrumentation overhead in a worst case).
```
tracepoint    base  kprobe+bpf tracepoint+bpf raw_tracepoint+bpf
task_rename   1.1M   769K        947K            1.0M
urandom_read  789K   697K        750K            755K
```

References
==========
- [http://man7.org/linux/man-pages/man2/bpf.2.html][1]
- [https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md][2]
- [https://github.com/iovisor/bcc][3]
- [https://sysdig.com/blog/sysdig-and-falco-now-powered-by-ebpf/][4]
- [http://chrisarges.net/2018/10/04/tracing-in-linux.html][5]
- [https://lwn.net/Articles/748352/][6]
- [https://github.com/iovisor/bcc/issues/237][7]
- [https://ferrisellis.com/posts/ebpf_past_present_future/][8]
- [https://ferrisellis.com/posts/ebpf_syscall_and_maps/][9]
- [https://github.com/zoidbergwill/awesome-ebpf][10]
- [http://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html][11]
- [https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/][12]
- [https://github.com/cilium/cilium/blob/master/Documentation/bpf.rst][13]


[1]: http://man7.org/linux/man-pages/man2/bpf.2.html
[2]: https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md
[3]: https://github.com/iovisor/bcc
[4]: https://sysdig.com/blog/sysdig-and-falco-now-powered-by-ebpf/
[5]: http://chrisarges.net/2018/10/04/tracing-in-linux.html
[6]: https://lwn.net/Articles/748352/
[7]: https://github.com/iovisor/bcc/issues/237
[8]: https://ferrisellis.com/posts/ebpf_past_present_future/
[9]: https://ferrisellis.com/posts/ebpf_syscall_and_maps/
[10]: https://github.com/zoidbergwill/awesome-ebpf
[11]: http://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html
[12]: https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/
[13]: https://github.com/cilium/cilium/blob/master/Documentation/bpf.rst
