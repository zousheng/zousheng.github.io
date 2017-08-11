---
layout: post
title: Mesos isolation cgroups/cpu & cgroups/mem
categories: [Technology]
tags: [mesos, isolation, cgroups/cpu, cgroups/mem]
permalink: /mesos-isolation-cgroups/
---


<!--excerpt-->

> 最近一直在纠结 mesos/marathon 中 isolation cgroups/cpu
> 是如何工作的，为什么明明设置了 cpus 限制， 但是任务用的 cpu
> 还是会超过阈值呢？最近偶然看到了 cgruops cfs
> 相关介绍才明白，设置cgroups/cpu 的同时还要设置cgroups cfs, 这样 cpu
> 才是独占的， 而默认情况下 cpu 是共享的。下面是收集的相关资料和总结。


## Mesos isolation cgroups/cpu & cgroups/mem


### Runtime Isolators

These isolators are used to ensure that a task behaves well at runtime and also provide runtime usage metrics for the given resource.

### posix/cpu

No actual resource isolation but does support returning usage metrics.

Metrics: cpu user time & system time See: https://github.com/apache/mesos/blob/037a346a205ad7bdba99d771855f8caeea835d4a/src/usage/usage.cpp#L35

### posix/mem

No actual resource isolation but does support returning usage metrics.

Metrics: mem_rss_bytes See: https://github.com/apache/mesos/blob/037a346a205ad7bdba99d771855f8caeea835d4a/src/usage/usage.cpp#L35

### posix/disk

Uses du -k -s to ensure tasks stay within disk usage limits.

#### Can Kill Tasks? Yes

Metrics: disk_limit_bytes, disk_used_bytes

// This isolator monitors the disk usage for containers, and reports
// ContainerLimitation when a container exceeds its disk quota. This
// leverages the DiskUsageCollector to ensure that we don't induce too
// much CPU usage and disk caching effects from running 'du' too
// often.
#### disk/du

Alias for posix/disk

#### Can Kill Tasks? Yes


#### **marathon cpu:**

* Marathon’s cpu setting is both a relative weight for scheduling all Docker containers across all of the Mesos slave’s CPUs and an amount of the Mesos slave’s available CPU capacity to use up
* A process running in a Docker container on a Mesos slave thinks it has the same number of CPUs as the underlying machine
* The OS should give relative weight to the Docker containers running on a Mesos slave according to their cpus values


### CPU Allocation


**There are several flags that influence the way how Mesos limits resources. For CPU is most important isolation (we're talking about mesos-slave/mesos-agent settings):**

* --isolation=posix/cpu,posix/mem None CPU limiting is applied mesos-executor is just a process that runs other process. You can use nice, e.g. nice -20 (for highest priority) or cpulimit commands to influence kernel planning, but Mesos's e.g. cpu=0.1 won't be taken into consideration.
* --isolation=cgroups/cpu,cgroups/mem cgroups (part of Linux Kernel since 2.6.29) allows limiting resources used by each process or group of processes. Some distributions does not enable memory limiting by default and cgroup_enable=memory need to be passed to the kernel. But let's focus on CPU. By default cgroups takes conservative approach where cpu=1.0 means that at least one CPU core will be reserved for the task. But in case that there is no other task running on the host it can consume all of the CPUs. Assuming that we have a host with 12 CPUs and there are two tasks running with cpu=2.0. Then each task might get up to 6 CPUs cores! (assuming no other Mesos task is running on that host). This is very dangerous, when cluster is at low load all tasks will look fine, but once there are many tasks performance of some hosts will decrease.
  * --cgroups_enable_cfs CFS stands for Completely Fair Scheduler which takes more strict approach. By default it is turned off, also not all distributions support this (you can use e.g. Docker's check-script.sh to verify support on your system). CFS will guarantee that each process can use at most the portion specified (e.g. cpu=2.5). This comes at a cost that no other process can utilize reserved cores when some task is idle. So, make sure you'll define your requirement well.

<br />


#### 总结:


  * `mesos默认isolation使用的是 posix/cpu,posix/mem, 这个配置只适合开发环境用，不适合生产环境，因为它没有对资源做任何的限制。(生产环境应该使用 cgroups/cpu,cgroups/mem.) `
  
  * `mesos isolation 配置 cgroups/cpu,cgroups/mem （默认 CFS 不会启用）， cpu 使用的方式是cpu shared，这种方式对 cpu 没有严格限制，机器上的任何 task 都可以访问机器上所有 cpu 资源; cgroups/mem 对内存限制严格，如果超过配置的数值，cgroup manager 会销毁对应的容器，利用 oom-killer 来杀掉对应的进程，相当于 kill -9 杀掉进程。 生产环境要合理配置任务的 mem 值来避免oom-killer发生`
  
  * `mesos isolation 配置 cgroups/cpu,cgroups/mem （启用 CFS），这个相当于配置了独占 cpu, 例如 marathon 中服务的 cpus 配置为1， 那么这个容器的 cpu 使用率就不会超过 100%. 相当于设定了一个 hard limit。 k8s 和 DC/OS 都是使用的这种资源隔离方式。当服务要用到的 cpu 时间片大于设定的阈值时，服务所在的容器不会被销毁，但是服务性能会受到影响， 吞吐量下降。而 mem 超过阈值 容器会被销毁。`  
  
  * `对于是否启用 CFS 要根据应用场景和服务类型来选择`

  * `marathon 上配置的 cpus 数值，不仅是一个 cpu 的数值，同时也可以指 cpu 资源的相对权重值, 理解这点对合理设置任务的 cpus 很有帮助`


#### 调优

 * `沙箱 marathon 上，所有服务的 cpus 设置 改为0.1。目前沙箱 mesos slave 机器整体 cpu 使用率很低，大约在10% ~ 15%左右，cpus 配置为0.1以后，每个 mesos slave 可以运行更多的任务。`
 
 * `针对私有部署中的服务 marathon cpus 可以调低 0.1， 个别 cpu 密集型的服务 e.g. webapp, gateway 可以改为0.2 或者0.3`

* `针对线上服务 cpus 值可以适当调低， 线上的 mesos slave CPU 使用率一般在 30% ～ 40% 之间，使用率其实不高， 调整后整体资源使用率会更合理`

`Reference:`

   * https://gist.github.com/chetan/9c519c68549b55d71709cb8cc62206ae
  
   *  https://zcox.wordpress.com/2014/09/17/cpu-resources-in-docker-mesos-and-marathon/



