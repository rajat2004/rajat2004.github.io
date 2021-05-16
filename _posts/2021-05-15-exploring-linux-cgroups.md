---
title: Exploring Linux Control Groups
date:  2021-05-15 08:00:02 +0530
classes: wide
toc: true
excerpt: Linux cgroups is used for limiting the system resources used by a set of processes, and is a basic block for enabling container resource isolation
---

## Introduction

Linux Control groups (cgroups) is a mechanism in the Linux kernel for limiting the system resources such as CPU, memory, I/O, etc. used by a set of processes. Control groups along with Linux namespaces are two basic building blocks for enabling container resource isolation and management, used for enabling OS-level virtualization, including Linux Containers (or LXC).

The cgroups mechanism partitions groups of processes and their children into hierarchical groups with controlled resource limits.

Few technical terms related to cgroups and how it groups processes together -

- A *cgroup* associates a set of tasks (a thread) with parameters for one or more subsystems

- A *subsystem* is a module which uses the grouping facility of cgroups in a particular way. Typically, it's a resource controller that sets per-cgroup resource limits.

- A *hierarchy* is a set of cgroups arranged in a tree. Each task can be associated with only a single cgroup in a hierarchy, and a set of subsystems. Each hierarchy has one or more subsystems attached to it, and an instance of the cgroup virtual filesystem associated with it.

There can be multiple active hierarchies of task cgroups at any time. Each hierarchy is a partition of all the tasks in a system.

For more details and how it is implemented, see the [Kernel cgroups documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt).

Now let's see it in action!

## Experiments

The following tests have been conducted on Ubuntu 18.04

```shell
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.5 LTS
Release:    18.04
Codename:   bionic

$ uname -r
5.4.0-72-generic
```

The cgroup virtual filesystem is located in `/sys/fs/cgroup`, it lists the various subsystems present. Checkout the [cgroups man page](https://man7.org/linux/man-pages/man7/cgroups.7.html) for details on each.

```shell
$ ls /sys/fs/cgroup
blkio  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls,net_prio  perf_event  pids  rdma  systemd  unified  cpu  cpuacct  net_cls  net_prio
```

For example, to check the current memory usage (see [this](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) for more details) -

```shell
$ cat /sys/fs/cgroup/memory/memory.usage_in_bytes
5955682304
```

Which is about 5.54 GiB, and is very close to what `free -m` says (total - available)

**Note**:: All memory value reads will be a multiple of the kernel's page size (i.e. 4096 bytes or 4KiB). This is the smallest allocatable size of memory

Inside container, it gives -

```shell
$ docker run -it ubuntu:20.04 bash

$ cat /sys/fs/cgroup/memory/memory.usage_in_bytes
7639040
```

Around 7.2 MiB, pretty less!

By default, containers are allowed to use as much memory as the kernel scheduler allows. This can be seen from the host machine -

```shell
$ cat /sys/fs/cgroup/memory/memory.limit_in_bytes
9223372036854771712

$ cat /sys/fs/cgroup/memory/docker/cc29550dc71dcd17768aadf04d08f33f4371ae8d065cb37470b9664ed81836d2/memory.limit_in_bytes
9223372036854771712
```

Docker creates a new cgroup for each container, with the container ID (`docker container ps`) as the directory. Various resource constraints can be applied to a container, see [Docker Resource Constraint docs](https://docs.docker.com/config/containers/resource_constraints/) and [Docker Run options](https://docs.docker.com/engine/reference/commandline/run/#options).

Applying the `--memory` limit of 20 MB -

```shell
$ docker run -it --memory 20m ubuntu:20.04 bash

$ cat /sys/fs/cgroup/memory/docker/014735a67658dfd8307bb032905cf0a12b731d60a7977df16df99ec89935b03a/memory.limit_in_bytes
20971520
```

### Memory Limits

The following steps are from [this article](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process) with some modifications so that it works on my system.

Install some packages -

```shell
sudo apt-get install libcgroup1 cgroup-tools
```

Test Script -

```sh
$ cat test.sh
#!/bin/sh

while true; do
    echo "hello world"
    sleep 60
done
```

The manual approach is used below. For using the utilities provided by the `libcgroup` package, and persistent cgroups, etc. check out the article.

To create a cgroup called `foo` under the memory subsystem, create a directory under `/sys/fs/cgroup/memory` -

```shell
sudo mkdir /sys/fs/cgroup/memory/foo
```

To set a limit for the `foo` cgroup, we'll have to write to the file `memory.limit_in_bytes`. Set the limit to 50 MB:

```shell
$ echo 50000000 | sudo tee /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
50000000
```

Verify that the value was written:

```shell
$ cat /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
49999872
```

Remember that value written will be a multiple of the page size, 4096 bytes in this case.

Start the process in background:

```shell
$ sh test.sh &
[1] 5709
hello world
```

Using the PID, move the process to `foo` cgroup under the `memory` subsystem:

```shell
$ echo 5709 | sudo tee /sys/fs/cgroup/memory/foo/cgroup.procs
5709
```

Using the same PID, verify that the process is running within the desired cgroup:

```shell
$ ps -o cgroup 5709
CGROUP
10:memory:/foo,8:blkio:/user.slice,6:cpu,cpuacct:/user.slice,5:devices:/user.slice,4:pids:/user.slice/user-1000.slice/user@1000.service,1:name=systemd:/user.slice/user-1000.slice/user@1000.service/gnome-t
```

Memory used by the process can be seen as -

```shell
$ cat /sys/fs/cgroup/memory/foo/memory.usage_in_bytes
495616
```

Now let's see what happens when the process exceeds the memory limit. Kill the original process, and then delete the `foo` cgroup. I've used some `libcgroup` tools below -

```shell
sudo cgdelete memory:foo
```

Again create the `foo` cgroup -

```shell
sudo cgcreate -g memory:foo
```

Set the limit to 5000 bytes which is greater than what the process normally used -

```shell
$ echo 5000 | sudo tee /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
5000

$ cat /sys/fs/cgroup/memory/foo/memory.limit_in_bytes
4096
```

Limit is set to a multiple of 4096 lesser than the desired value. Start the script and move it to the cgroup, and then check the system logs-

```shell
$ sh test.sh &
[1] 6339
hello world

$ echo 6339 | sudo tee /sys/fs/cgroup/memory/foo/cgroup.procs
6339

$ tail /var/log/syslog
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744476] pglazyfreed 0
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744476] thp_fault_alloc 0
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744476] thp_collapse_alloc 0
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744476] Tasks state (memory values in pages):
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744476] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744478] [   6339]  1000  6339     1158      223    53248        0             0 sh
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744479] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/foo,task_memcg=/foo,task=sh,pid=6339,uid=1000
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744483] Memory cgroup out of memory: Killed process 6339 (sh) total-vm:4632kB, anon-rss:68kB, file-rss:824kB, shmem-rss:0kB, UID:1000 pgtables:52kB oom_score_adj:0
May 15 12:25:53 rajat-G5-5587 kernel: [17565.744588] oom_reaper: reaped process 6339 (sh), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

Notice that the Out-of-Memory killer (oom-killer) killed the process as soon as it hit the 4KB limit. Verify that the process is no longer running-

```shell
$ ps -o cgroup 6339
CGROUP

```

### PID limits

By default, the number of processes which can be created has no limit. To add such a limit, set it in the `pids.max` file inside the `pids` subsystem. This file is not available in the root cgroup, see [`pids` subsystem docs also](https://www.kernel.org/doc/Documentation/cgroup-v1/pids.txt).

Docker container by default also has no limit of processes -

```shell
$ cat /sys/fs/cgroup/pids/docker/bb33d6ba5fb72615615d593c144b1d6c85c684e63241ccb57df6bc737c5bd0bb/pids.max
max
```

Here's an example of limiting the number of processes which can be created:

```shell
$ sudo cgcreate -g pids:foo

$ cat /sys/fs/cgroup/pids/foo/pids.max
max

$ echo 20 | sudo tee /sys/fs/cgroup/pids/foo/pids.max
20

$ cat /sys/fs/cgroup/pids/foo/pids.max
20
```

We've set the limit to be 20 processes. Now the test script containing the famous fork bomb (Side note: [here's](https://www.vidarholen.net/contents/blog/?p=766) a next-level one)-

```shell
$ cat test.sh
#!/bin/bash

# Fork bomb
:(){ :|:& };:
```

Run the script under the `foo` cgroup using the `cgexec` tool -

```shell
sudo cgexec -g pids:foo ~/test.sh
```

You'll get lots of `fork: retry: Resource temporarily unavailable` messages, but no crash.

```shell
$ tail /var/log/syslog
...
May 15 13:17:48 rajat-G5-5587 kernel: [ 1367.341566] cgroup: fork rejected by pids controller in /foo
...
```

There's still a catch here! Deleting the cgroup directly will cause problems, since when a cgroup is deleted, all the tasks will move to the parent group, which is the `user` group, and it doesn't have a limit on PIDs and hence the fork bomb will continue.

First, kill all the running processes under the cgroup, and then delete the cgroup-

```shell
sudo kill -9 $(< /sys/fs/cgroup/pids/foo/tasks)

sudo cgdelete pids:foo
```

Note that killing the processes directly worked here, but to reliably kill all the processes, you'll have to freeze all the processes (using the `freezer` subsystem), send SIGKILL and then unfreeze them.

This is intended to be the first of a series of blogs while trying to implement some of the concepts in the paper [Houdini’s Escape: Breaking the Resource Rein of Linux Control Groups](https://dl.acm.org/doi/10.1145/3319535.3354227).

References:

- [Kernel cgroup-v1 documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
- [Kernel cgroup-v2 documentation](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
- [cgroups man page](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [Everything You Need to Know about Linux Containers, Part I: Linux Control Groups and Process Isolation](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process)
