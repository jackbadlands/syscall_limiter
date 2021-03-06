This program allows user to set up simple [seccomp][1] filter that limits 
which syscalls can be called. All the rest syscalls return pre-defined error 
value `Channel number out of range`.

The supplied `monitor.sh` script assists executing of binaries under strace for automatic creating of the allowed syscalls list.

This program uses [libseccomp][2] library. 
The program supports setting comparator for syscall arguments and 
specifying alternative default or syscall actions; for simple version see 
[nocreep][5] branch.

Here is pre-built static simple x86 binary: [limit_syscalls_static][3].

Note that rules for syscall_limiter are not very versatile. 
There are inherent limitations what they can't do 
(for example, dereference memory or be stateful). 
To address this, you can "ban too much", but also provide external helper that will, 
for example, open files or sockets and send them using SCM_RIGHTS to us, implementing 
more sophisticated security checks on the way. [Seccomp-nurse][6] uses this approach. 
With syscall_limiter you can reduce the amount of what should be helped externally and access helper in more ways.
LD_PRELOAD approach can be used to detour system functions to our helper. This way we will have something in middle between seccomp-nurse and pure-BPF-seccomp-no-helper approaches: some syscalls are just banned, some are just allowed and some are redirected to helper.

See writelimiter subdirectory for example of such approach. It allows users to start programs with only specified parts of filesystem being writable.


You can use `strace -e raw=somesyscall` to discover syscall parameter numbers and values.

```
# strace -e mount -e raw=mount /bin/mount -t tmpfs tmp /tmp/foo -o remount,ro
mount(0x80634c8, 0x80634d8, 0x80634e8, 0xc0ed0021, 0) = 0

# strace -e mount -e raw=mount /bin/mount -t tmpfs tmp /tmp/foo -o remount,rw
mount(0x80634c8, 0x80634d8, 0x80634e8, 0xc0ed0020, 0) = 0
```


Examples
===

Basic
---

```
$ ./limit_syscalls
Usage: limit_syscalls syscall1 syscall2 ... syscallN -- program [arguments]
Example:
   limit_syscalls execve exit write read open close mmap2  fstat64 access  mprotect set_thread_area -- /bin/echo qqq
Advanced:
   LIMIT_SYSCALLS_DEFAULT_ACTION={a,k,eN} - by default allow, kill or return error N
   some_syscall,A0>=3,A4<<1000,A1!=4,A2&&0x0F==0x0F - compare arguments
   some_syscall,{a,k,eN} - allow, kill or return error N for this rule
Some more example:
    LIMIT_SYSCALLS_DEFAULT_ACTION=a  limit_syscalls  'write,A0==1,e0' -- /usr/bin/printf --help
     (this makes write to stdout in /usr/bin/printf silently fail, looping it)
Restrict user namespace (CLONE_NEWUSER):
   LIMIT_SYSCALLS_DEFAULT_ACTION=a limit_syscalls clone,A0\&\&0x10000000==0x10000000,e1 unshare,e1  -- /bin/bash
   for this use case there is a separate executable ban_CLONE_NEWUSER
   
   
$ ./limit_syscalls execve exit write read open close mmap2  fstat64 access  mprotect set_thread_area -- /bin/echo qqq
qqq

$ # Let's cut off "exit".
$ ./limit_syscalls execve write read open close mmap2  fstat64 access  mprotect set_thread_area -- /bin/echo qqq
qqq
Segmentation fault

```

ping
---

```
# ./monitor.sh ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.132 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.108 ms
^C
--- 127.0.0.1 ping statistics ---
<cut>
# limit_syscalls access brk close connect execve exit_group fstat64 getpid getsockname getsockopt gettimeofday getuid32 ioctl mmap2 mprotect munmap open poll read recvmsg rt_sigaction sendmsg setsockopt set_thread_area setuid32 sigreturn socket write  -- /bin/ping

# ./limit_syscalls access brk close connect execve exit_group fstat64 getpid getsockname getsockopt gettimeofday getuid32 ioctl mmap2 mprotect munmap open poll read recvmsg rt_sigaction sendmsg setsockopt set_thread_area setuid32 sigreturn socket write  -- /bin/ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_req=1 ttl=64 time=0.132 ms
64 bytes from 127.0.0.1: icmp_req=2 ttl=64 time=0.108 ms
^C
--- 127.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.108/0.120/0.132/0.012 ms

# 
```

firefox
---

```
$ ./monitor.sh /opt/firefox-17esr/firefox
$
$ limit_syscalls access bind brk chmod clock_getres clock_gettime clone close connect dup2 epoll_create epoll_ctl epoll_wait eventfd2 execve exit exit_group fallocate fcntl64 fstat64 fstatfs fsync ftruncate64 futex getcwd getdents getdents64 getegid32 geteuid32 getgid32 getpeername getresgid32 getresuid32 getrlimit getrusage getsockname getsockopt gettid gettimeofday getuid32 ioctl _llseek lseek lstat64 madvise mkdir mmap2 mprotect munmap open pipe poll prctl pwrite64 read readahead readlink recv recvfrom recvmsg rename rmdir rt_sigaction rt_sigprocmask sched_getaffinity sched_getparam sched_get_priority_max sched_get_priority_min sched_getscheduler sched_setscheduler send sendmsg sendto set_robust_list setsockopt set_thread_area set_tid_address shmat shmctl shmdt shmget shutdown sigaltstack socket socketpair stat64 statfs statfs64 symlink sysinfo time umask uname unlink utime waitpid write writev  -- /opt/firefox-17esr/firefox

$ ./limit_syscalls access bind brk chmod clock_getres clock_gettime clone close connect dup2 epoll_create epoll_ctl epoll_wait eventfd2 execve exit exit_group fallocate fcntl64 fstat64 fstatfs fsync ftruncate64 futex getcwd getdents getdents64 getegid32 geteuid32 getgid32 getpeername getresgid32 getresuid32 getrlimit getrusage getsockname getsockopt gettid gettimeofday getuid32 ioctl _llseek lseek lstat64 madvise mkdir mmap2 mprotect munmap open pipe poll prctl pwrite64 read readahead readlink recv recvfrom recvmsg rename rmdir rt_sigaction rt_sigprocmask sched_getaffinity sched_getparam sched_get_priority_max sched_get_priority_min sched_getscheduler sched_setscheduler send sendmsg sendto set_robust_list setsockopt set_thread_area set_tid_address shmat shmctl shmdt shmget shutdown sigaltstack socket socketpair stat64 statfs statfs64 symlink sysinfo time umask uname unlink utime waitpid write writev  -- /opt/firefox-17esr/firefox

(firefox:10950): Gdk-WARNING **: shmget failed: error 44 (Channel number out of range)

$ # firefox seems to be working although

```

[1]:http://en.wikipedia.org/wiki/Seccomp
[2]:http://sourceforge.net/projects/libseccomp/
[3]:http://vi-server.org/pub/limit_syscalls_static
[5]:https://github.com/vi/syscall_limiter/tree/nocreep
[6]:http://chdir.org/~nico/seccomp-nurse/
