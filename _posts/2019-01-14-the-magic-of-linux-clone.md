---
layout: post
title: the magic of linux clone
date: '2019-01-14T12:44:27-0600'
author: arges
tags:
- linux
- kernel
modified_time: '2019-01-14T12:44:34-0600'
---

Processes and threads are concepts that have been used across many operating
systems and programming languages. While we tend to think of 'process' as a
heavy thing and 'thread' as a light thing, how they are implemented and handled
depends on context. This article will focus on how the Linux operating system
handles these concepts and the mechanisms used.


Creating a new process
----------------------

Generally in Linux if you want to create a new process in C you can do the following:

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/syscall.h>
#include <unistd.h>

int main()
{
	pid_t pid;
	pid = fork();

	if (pid < 0) {
		exit(-1);
	}

	if (pid == 0) {
		pid_t tid = syscall(__NR_gettid);
		printf("child: pid:%ld tid:%ld\n", (long)getpid(), (long)tid);
	} else {
		pid_t tid = syscall(__NR_gettid);
		printf("parent: pid:%ld tid:%ld\n", (long)getpid(), (long)tid);
	}
	return 0;
}
```

Output from the program:
```
child: pid:9154 tid:9154
parent: pid:9153 tid:9153
```

Notice that process and thread ids have changed and that `pid == tid` in this model.
Now if you strace the program you will see the following:
```
execve("./a.out", ["./a.out"], 0x7fff319c16d0 /* 61 vars */) = 0
brk(NULL)                               = 0x557cdc5e3000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=99382, ...}) = 0
mmap(NULL, 99382, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f2719d5e000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\260\34\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2030544, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f2719d5c000
mmap(NULL, 4131552, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f271975f000
mprotect(0x7f2719946000, 2097152, PROT_NONE) = 0
mmap(0x7f2719b46000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e7000) = 0x7f2719b46000
mmap(0x7f2719b4c000, 15072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f2719b4c000
close(3)                                = 0
arch_prctl(ARCH_SET_FS, 0x7f2719d5d4c0) = 0
mprotect(0x7f2719b46000, 16384, PROT_READ) = 0
mprotect(0x557cdb1d6000, 4096, PROT_READ) = 0
mprotect(0x7f2719d77000, 4096, PROT_READ) = 0
munmap(0x7f2719d5e000, 99382)           = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f2719d5d790) = 9154
gettid()                                = 9153
getpid()                                = 9153
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 2), ...}) = 0
brk(NULL)                               = 0x557cdc5e3000
brk(0x557cdc604000)                     = 0x557cdc604000
write(1, "parent: pid:9153 tid:9153\n", 26) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=9154, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
write(1, "parent: pid:9153 tid:9153\n", 26) = 26
exit_group(0)                           = ?
+++ exited with 0 +++
```

Note the lack of `fork` in the above trace. In Linux and specifically programs
that use `glibc` will utilize the `clone` system call with specific flags
(`CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD`) to accomplish a `fork`.


Creating a new thread
---------------------

The following program creates two threads and also prints information about
process id and thread id for the newly created threads and even the main
thread.

```c
#include <pthread.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/syscall.h>
#include <unistd.h>

void *thread_function(void *ptr)
{
	pid_t tid = syscall(__NR_gettid);
	printf("thread:#%d pid:%ld tid:%ld\n", (int)ptr, (long)getpid(),
	       (long)tid);
	return ptr;
}

int main(int argc, char **argv)
{
	pthread_t thread1, thread2;
	pthread_create(&thread1, NULL, *thread_function, (void *)1);
	pthread_create(&thread2, NULL, *thread_function, (void *)2);

	/* main thread */
	pid_t tid = syscall(__NR_gettid);
	printf("thread:#%d pid:%ld tid:%ld\n", (int)0, (long)getpid(),
	       (long)tid);

	pthread_join(thread1, NULL);
	pthread_join(thread2, NULL);
	return 0;
}
```

Example output from this program:
```
thread:#1 pid:7918 tid:7919
thread:#0 pid:7918 tid:7918
thread:#2 pid:7918 tid:7920
```

Notice that the process ID remains the same, while the thread ID changes.


Strace output from the program:
```
execve("./a.out", ["./a.out"], 0x7fff882a00d0 /* 61 vars */) = 0
brk(NULL)                               = 0x55f25c150000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=99382, ...}) = 0
mmap(NULL, 99382, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f8a74eda000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000b\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=144976, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8a74ed8000
mmap(NULL, 2221184, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f8a74aad000
mprotect(0x7f8a74ac7000, 2093056, PROT_NONE) = 0
mmap(0x7f8a74cc6000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x19000) = 0x7f8a74cc6000
mmap(0x7f8a74cc8000, 13440, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f8a74cc8000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\260\34\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2030544, ...}) = 0
mmap(NULL, 4131552, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f8a746bc000
mprotect(0x7f8a748a3000, 2097152, PROT_NONE) = 0
mmap(0x7f8a74aa3000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e7000) = 0x7f8a74aa3000
mmap(0x7f8a74aa9000, 15072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f8a74aa9000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8a74ed5000
arch_prctl(ARCH_SET_FS, 0x7f8a74ed5740) = 0
mprotect(0x7f8a74aa3000, 16384, PROT_READ) = 0
mprotect(0x7f8a74cc6000, 4096, PROT_READ) = 0
mprotect(0x55f25a1c7000, 4096, PROT_READ) = 0
mprotect(0x7f8a74ef3000, 4096, PROT_READ) = 0
munmap(0x7f8a74eda000, 99382)           = 0
set_tid_address(0x7f8a74ed5a10)         = 9270
set_robust_list(0x7f8a74ed5a20, 24)     = 0
rt_sigaction(SIGRTMIN, {sa_handler=0x7f8a74ab2cb0, sa_mask=[], sa_flags=SA_RESTORER|SA_SIGINFO, sa_restorer=0x7f8a74abf890}, NULL, 8) = 0
rt_sigaction(SIGRT_1, {sa_handler=0x7f8a74ab2d50, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART|SA_SIGINFO, sa_restorer=0x7f8a74abf890}, NULL, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [RTMIN RT_1], NULL, 8) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f8a73ebb000
mprotect(0x7f8a73ebc000, 8388608, PROT_READ|PROT_WRITE) = 0
brk(NULL)                               = 0x55f25c150000
brk(0x55f25c171000)                     = 0x55f25c171000
clone(child_stack=0x7f8a746bafb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tidptr=0x7f8a746bb9d0, tls=0x7f8a746bb700, child_tidptr=0x7f8a746bb9d0) = 9271
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f8a736ba000
mprotect(0x7f8a736bb000, 8388608, PROT_READ|PROT_WRITE) = 0
clone(child_stack=0x7f8a73eb9fb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tidptr=0x7f8a73eba9d0, tls=0x7f8a73eba700, child_tidptr=0x7f8a73eba9d0) = 9272
gettid()                                = 9270
getpid()                                = 9270
write(1, "thread:#0 pid:9270 tid:9270\n", 28) = 28
exit_group(0)                           = ?
+++ exited with 0 +++
```

Here note two calls to `clone`. This time with different flags:
`CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID`.

One flag in particular stands out which is `CLONE_THREAD`. Now, let's look more into
the `clone` system call.

The `clone` system call
-----------------------

The description for clone says: `create a child process`. In addition looking
more into the man page for this call we notice that there are _many_ flags that
can be used for different behaviors. Above we saw that `fork` and
`pthread_create` just use `clone` with different flags.

A summary of the flags (as copied from the Linux kernel sources):
```
#define CLONE_VM    0x00000100	/* set if VM shared between processes */
#define CLONE_FS    0x00000200	/* set if fs info shared between processes */
#define CLONE_FILES 0x00000400	/* set if open files shared between processes */
#define CLONE_SIGHAND   0x00000800	/* set if signal handlers and blocked signals shared */
#define CLONE_PTRACE    0x00002000	/* set if we want to let tracing continue on the child too */
#define CLONE_VFORK 0x00004000	/* set if the parent wants the child to wake it up on mm_release */
#define CLONE_PARENT    0x00008000	/* set if we want to have the same parent as the cloner */
#define CLONE_THREAD    0x00010000	/* Same thread group? */
#define CLONE_NEWNS 0x00020000	/* New mount namespace group */
#define CLONE_SYSVSEM   0x00040000	/* share system V SEM_UNDO semantics */
#define CLONE_SETTLS    0x00080000	/* create a new TLS for the child */
#define CLONE_PARENT_SETTID 0x00100000	/* set the TID in the parent */
#define CLONE_CHILD_CLEARTID    0x00200000	/* clear the TID in the child */
#define CLONE_DETACHED      0x00400000	/* Unused, ignored */
#define CLONE_UNTRACED      0x00800000	/* set if the tracing process can't force CLONE_PTRACE on this clone */
#define CLONE_CHILD_SETTID  0x01000000	/* set the TID in the child */
#define CLONE_NEWUTS        0x04000000	/* New utsname namespace */
#define CLONE_NEWIPC        0x08000000	/* New ipc namespace */
#define CLONE_NEWUSER       0x10000000	/* New user namespace */
#define CLONE_NEWPID        0x20000000	/* New pid namespace */
#define CLONE_NEWNET        0x40000000	/* New network namespace */
#define CLONE_IO        0x80000000	/* Clone io context */
```


In the "fork" case above we see only flags specifing clearing and settings of
TID are expressed. Thus, the pids and tid are different.

In the above "thread" case we see that virtual memory, files, open files,
signal handlers are shared. In addition `CLONE_THREAD` is used which does the
following:
```
CLONE_THREAD (since Linux 2.4.0-test8)
If CLONE_THREAD is set, the child is placed in the same thread group as the
calling process.
Thread groups were a feature added in Linux 2.4 to support the POSIX threads
notion of a set of threads that share a single PID. Internally, this shared PID
is the so-called thread group identifier (TGID) for the thread group. Since
Linux 2.4, calls to getpid(2) return the TGID of the caller.
```

So this case makes since because the `pid` is the same; but the `tid` changes.
However the language is confusing; the `tgid` in the kernel actually maps to
the `pid` in userspace, and the `pid` in kernel space maps to the `tid` in
userspace!
