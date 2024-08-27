---
title: "cgroupv2介绍"
date: 2023-06-01T14:51:10-08:00
draft: false
categories: ["cgroup"]
tags: ["cgroup"]
---

记录一下 Linux Cgroup V2 版本基本使用操作，包括 cpu、memory 子系统演示。

## 1. 开启 Cgroup V2

### 版本检查

通过下面这条命令来查看当前系统使用的 Cgroups V1 还是 V2

```
stat -fc %T /sys/fs/cgroup/
```

如果输出是`cgroup2fs` 那就是 V2，就像这样

```
root@tezn:~# stat -fc %T /sys/fs/cgroup/ cgroup2fs
```

如果输出是`tmpfs` 那就是 V1，就像这样

```
[root@docker cgroup]# stat -fc %T /sys/fs/cgroup/ tmpfs
```

### 启用 cgroup v2

如果当前系统未启用 Cgroup V2，也可以通过修改内核 cmdline 引导参数在你的 Linux 发行版上手动启用 cgroup v2。

如果你的发行版使用 GRUB，则应在 `/etc/default/grub` 下的 `GRUB_CMDLINE_LINUX` 中添加 `systemd.unified_cgroup_hierarchy=1`， 然后执行 `sudo update-grub`。

具体如下：

1）编辑 grub 配置

```
vi /etc/default/grub
```

内容大概是这样的：

```
GRUB_DEFAULT=0 GRUB_TIMEOUT_STYLE=hidden GRUB_TIMEOUT=0 GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian` GRUB_CMDLINE_LINUX_DEFAULT="quiet splash" GRUB_CMDLINE_LINUX=""
```

对最后一行`GRUB_CMDLINE_LINUX`进行修改

```
GRUB_CMDLINE_LINUX="quiet splash systemd.unified_cgroup_hierarchy=1"
```

2）然后执行以下命令更新 GRUB 配置

```
sudo update-grub
```

3）最后查看一下启动参数，确认配置修改上了

```
cat /boot/grub/grub.cfg | grep "systemd.unified_cgroup_hierarchy=1"
```

4）然后就是重启

```
reboot
```

重启后查看，不出意外切换到 cgroups v2 了

```
root@cgroupv2:~# stat -fc %T /sys/fs/cgroup/ cgroup2fs
```

### 发行版推荐

不过，推荐的方法仍是使用一个默认已启用 cgroup v2 的发行版。

有关使用 cgroup v2 的 Linux 发行版的列表，

- Container-Optimized OS（从 M97 开始）
- Ubuntu（从 21.10 开始，推荐 22.04+）
- Debian GNU/Linux（从 Debian 11 Bullseye 开始）
- Fedora（从 31 开始）
- Arch Linux（从 2021 年 4 月开始）
- RHEL 和类似 RHEL 的发行版（从 9 开始）

## 2. 基本使用

cgroup v2 使用上和 v1 版本基本一致，v2 版本也是默认在`/sys/fs/cgroup/`目录。

```
root@mydocker:~# ls /sys/fs/cgroup/ cgroup.controllers      cgroup.subtree_control  init.scope       system.slice cgroup.max.depth        cgroup.threads          io.cost.model    user.slice cgroup.max.descendants  cpu.pressure            io.cost.qos cgroup.procs            cpuset.cpus.effective   io.pressure cgroup.stat             cpuset.mems.effective   memory.pressure
```

- 创建 sub-cgroup： 只需要创建一个子目录

```
cd /sys/fs/cgroup mkdir $CGROUP_NAME
```

- 将进程移动到指定 cgroup：将 PID 写入到相应 cgroup 的 cgroup.procs 文件即可，就像这样：

```
echo 1001 > /sys/fs/cgroup/test/cgroup.procs
```

- 删除 cgroup/sub-cgroup： 也是直接删除对应目录即可
  - 如果一个 cgroup 已经没有任何 children 或活进程，那直接删除对应的文件夹就删除该 cgroup 了
  - 如果一个 cgroup 已经没有 children，但是还有僵尸进程，也认为这个 cgroup 是空的，可以直接删除

```
rmdir /sys/fs/cgroup/test
```

- 修改 cpu、memory 限制：往对应配置文件写入配置内容即可
  - cpu.max 用于配置 cpu 使用限制
  - memory.max 则用于配置 内存使用限制
  - …

### 创建 cgroup

接下来，以 cpu、memory 为例，简单演示一下 cgroup v2 版本使用

```
root@mydocker:~# cd /sys/fs/cgroup/ root@mydocker:/sys/fs/cgroup# mkdir test root@mydocker:/sys/fs/cgroup# cd test root@mydocker:/sys/fs/cgroup/test# ls cgroup.controllers      cpu.uclamp.max         memory.current cgroup.events           cpu.uclamp.min         memory.events cgroup.freeze           cpu.weight             memory.events.local cgroup.max.depth        cpu.weight.nice        memory.high cgroup.max.descendants  cpuset.cpus            memory.low cgroup.procs            cpuset.cpus.effective  memory.max cgroup.stat             cpuset.cpus.partition  memory.min cgroup.subtree_control  cpuset.mems            memory.oom.group cgroup.threads          cpuset.mems.effective  memory.pressure cgroup.type             io.max                 memory.stat cpu.max                 io.pressure            pids.current cpu.pressure            io.stat                pids.events cpu.stat                io.weight              pids.max
```

### CPU

启动一个死循环

```
root@mydocker:/sys/fs/cgroup/test# while : ; do : ; done & [1] 90482
```

不出意外的话，应该占用了 100% cpu

```
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND     90482 root      20   0   10160   1772      0 R  99.3   0.1   0:05.01 bash
```

接下来使用 cgroup v2 限制该进程只能使用 20% cpu

1）修改配置

```
echo 2000 10000 > cpu.max
```

> 含义是在 10000 微秒的 CPU 时间周期内，有 2000 微秒是分配给本 cgroup 的，也就是本 cgroup 管理的进程在单核 CPU 上的使用率不会超过 20%。

2）将进程加入当前 cgroup

```
echo 90482 > cgroup.procs
```

再次查看

```
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND     90482 root      20   0   10160   1772      0 R  20.2   0.1   2:07.78 bash
```

可以看到，已经被限制到了 20%

### Memory

接下来演示内存限制，使用以下代码来模拟内存消耗，

```
cat <<EOF > ~/mem-allocate.c #include <stdio.h> #include <stdlib.h> #include <string.h> #include <unistd.h> #define MB (1024 * 1024) int main(int argc, char *argv[]) {    char *p;    int i = 0;    while(1) {        p = (char *)malloc(MB);        memset(p, 0, MB);        printf("%dM memory allocated\n", ++i);        sleep(1);    }     return 0; } EOF
```

编译

```
gcc ~/mem-allocate.c -o ~/mem-allocate
```

然后启动该文件

```
root@mydocker:/sys/fs/cgroup/test# ~/mem-allocate 1M memory allocated 2M memory allocated 3M memory allocated 4M memory allocated 5M memory allocated 6M memory allocated 7M memory allocated 8M memory allocated 9M memory allocated 10M memory allocated 11M memory allocated 12M memory allocated ^C
```

可以看到，每秒会消耗 1M 内存，若不停止会一直运行直到 OOM。

接下来使用 cgroup v2 限制最多消耗 10M 内存。

1）修改配置

> 单位为字节 10485760= 10 _ 1024 _ 1024

```
echo 10485760 > memory.max
```

> 也就是本 cgroup 管理的进程内存使用不会超过 10M

2）将进程加入当前 cgroup

```
#将当前bash加入到test中，这样这个bash创建的所有进程都会自动加入到test中 sh -c "echo $$ >> cgroup.procs"
```

再次查看

```
root@mydocker:/sys/fs/cgroup/test# ~/mem-allocate 1M memory allocated 2M memory allocated 3M memory allocated 4M memory allocated 5M memory allocated 6M memory allocated 7M memory allocated 8M memory allocated 9M memory allocated Killed
```

可以看到，到 10M 时就因为达到内存上限而被 Kill 了。

### 删除 cgroup

演示完成，把 cgroup 删除。

首先把进程 kill 一下

```
root@mydocker:/sys/fs/cgroup# cat test/cgroup.procs  90444 90630 root@mydocker:/sys/fs/cgroup# kill -9 90630 root@mydocker:/sys/fs/cgroup# kill -9 90444
```

然后删除目录

```
root@mydocker:/sys/fs/cgroup# rmdir test
```

这样 cgroup 就删除了。

## 3. v1 v2 对比

v1 的 cgroup 为每个控制器都使用独立的树(目录)

```
[root@docker cgroup]# ls /sys/fs/cgroup/ blkio  cpu  cpuacct  cpuacct,cpu  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  net_cls,net_prio  net_prio  perf_event  pids  rdma  systemd
```

每个目录就代表了一个 cgroup subsystem，比如要限制 cpu 则需要到 cpu 目录下创建子目录(树)，限制 memory 则需要到 memory 目录下去创建子目录(树)。

比如 Docker 就会在 cpu、memory 等等目录下都创建一个名为 docker 的目录，在 docker 目录下在根据 containerID 创建子目录来实现资源限制。

> 各个 Subsystem 各自为政，看起来比混乱，难以管理

因此最终的结果就是：

1. 用户空间最后**管理着多个非常类似的 hierarchy**，
2. 在执行 hierarchy 管理操作时，**每个 hierarchy 上都重复着相同的操作**。

**v2 中对 cgroups 的最大更改是将重点放在简化层次结构上**

- v1 为每个控制器使用独立的树（例如 `/sys/fs/cgroup/cpu/GROUPNAME`和 `/sys/fs/cgroup/memory/GROUPNAME`）。
- v2 将统一`/sys/fs/cgroup/GROUPNAME`中的树，如果进程 X 加入`/sys/fs/cgroup/test`，则启用 test 的每个控制器都将控制进程 X。

更多 v1 和 v2 差异见 [`v1` 存在的问题及 `v2` 的设计考虑](https://www.kernel.org/doc/html/v5.10/admin-guide/cgroup-v2.html#issues-with-v1-and-rationales-for-v2)

## 4. 小结

本文主要分享了 Linux cgroup v2 版本的基本使用，以及 v1 和 v2 版本的差异。

更多 cgroup v2 信息推荐阅读：[Control Group v2](https://www.kernel.org/doc/html/v5.10/admin-guide/cgroup-v2.html) 及其译文 [Control Group v2（cgroupv2 权威指南）（KernelDoc, 2021）](https://arthurchiao.art/blog/cgroupv2-zh)
