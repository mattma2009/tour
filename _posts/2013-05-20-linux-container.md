---
layout: post
title: LinuxContainer-LXC
categories: [work]
tags: [LXC]
---

最近在考虑PAAS平台上应用的隔离和占用资源控制，目前考虑到两级虚拟化，第一级可以采用Vmware/XEN\KVM等，但是一级虚拟化本身对资源的消耗太大。第二级，考虑采用LXC的方式。

## 概述 ##
LXC是一个用户态的Linux Container管理工具。其主要作用是借用cgroup技术在当前linux环境下实现一个轻量化的虚拟机。LXC的一个实例我们一般以container来称呼。简单来讲，我们可以把lxc + cgroup看做是VirtualBox和Vmware一类的东西。Linux Container容器是一种内核虚拟化技术，可以提供轻量级的虚拟化，以便隔离进程和资源，而且不需要提供指令解释机制以及全虚拟化的其他复杂性。相当于C++中的NameSpace。容器有效地将由单个操作系统管理的资源划分到孤立的组中，以更好地在孤立的组之间平衡有冲突的资源使用需求。

Linux Container是OS级别的虚拟化方案，它相比于一般的虚拟机没有了硬件模拟以及指令模拟，相比传统虚拟机具有更低的开销，因此可以应用到私有云之中。LXC目前的版本支持对memory，cpu以及block IO的管理和限制，目前不支持对网络IO的管理，但该特性已经加入到其roadmap，这些资源的管理和限制对企业私有云的搭建份至关重要，可以提高集群资源的使用率。

与传统虚拟化技术相比，它的优势在于：


> （1）与宿主机使用同一个内核，性能损耗小；
> 
> （2）不需要指令级模拟；
> 
> （3）不需要即时(Just-in-time)编译；
> 
> （4）容器可以在CPU核心的本地运行指令，不需要任何专门的解释机制；
> 
> （5）避免了准虚拟化和系统调用替换中的复杂性；
> 
> （6）轻量级隔离，在隔离的同时还提供共享机制，以实现容器与宿主机的资源共享。

总结：Linux Container是一种轻量级的虚拟化的手段。

## LXC的实现 ##

Sourceforge上有LXC这个开源项目。LXC项目本身只是一个为用户提供一个用户空间的工具集，用来使用和管理LXC容器。LXC真正的实现则是靠Linux内核的相关特性，LXC项目只是对此做了整合。基于容器的虚拟化技术起源于所谓的资源容器和安全容器。

LXC在资源管理方面依赖于Linux内核的cgroups子系统，cgroups子系统是Linux内核提供的一个基于进程组的资源管理的框架，可以为特定的进程组限定可以使用的资源。LXC在隔离控制方面依赖于Linux内核的namespace特性，具体而言就是在clone时加入相应的flag（NEWNS NEWPID等等）。

## lxc常见命令 ##

lxc-version 用于显示系统LXC的版本号（可以通过此命令判断系统是否安装了lxc）
`
> `用法：lxc-version
例如:lxc-version``

lxc-checkconfig 用于判断linux内核是否支持LXC
`
> `用法：lxc-checkconfig
例如：lxc-checkconfig``

lxc-create用于创建一个容器




> 用法：lxc-create -n name [-f config_file]
-n 后面跟要创建的容器名字 例如：-n foo -f 后面跟容器配置文件的路径

注：

1.采用lxc-create创建的容器，在停止运行后，不会被销毁，要采用lxc-destroy命令才能销毁

2.容器命令空间是全局的，系统中不允许存在重名的容器，如果-n 后面跟一个已经存在的容器名，创建会失败

例如：lxc-create --n foo --f foo.conf

lxc-execute 用于在一个容器执行应用程序

>     用法： lxc-execute -n name [-f config_file] [ -s KEY=VAL ]command

-n 后面跟容器名字（容器名字用于管理容器）例如：-n foo

-f 后面跟容器配置文件的路径（如果没有配置文件，可以直接用-s指定配置选项，如果什么都没有，系统采用默认策略）例如：-f foo.conf

-s 后面跟配置键值对 例如：lxc.cgroup.cpu.shares=512

command 为要执行的命令 例如：/bin/bash

这个命令会mount /proc 并且会自动创建/销毁容器。

注：
1.如果容器还不存在，lxc-execute会自动创建一个,容器停止运行后会被自动销毁

2.用lxc-execute启动应用程序，配置优先级如下：

如果指定-f选项，那么之前创建容器（如果容器是已存在的）的配置文件不会被使用

如果指定-s选项，则在命令行中的配置键值对会覆盖配置文件（无论之前的还是-f指定的）相同配置

例如：lxc-execute --n foo --s lxc.cgroup.cpu.shares=512 /bin/bash

使用实际例子:

lxc-execute -n test /bin/bash

这个会启动一个lxc并给出类似的一个cmd窗口，网络是与操作系统共用的，这里好像仅仅是创建了一个命名空间

如果没有指定-f，默认的隔离将被使用，这个命令当你需要一个快速在一个隔离的环境中运行程序。在物理机上和container中都会运行lxc-init，在宿主机上面，这个程序用于转发lxc-kill 信号到已经启动的程序中 ，在container中，这个程序的pid为1，它会fork出要执行的命令（pid为2）并执行。

lxc-start 用于在容器中执行给定命令
>     用法：lxc-start -n name [-f config_file] [-c console_file] [-d] [-s KEY=VAL] [command]

-d 将容器当做守护进程执行

-f 后面跟配置文件

-c 指定一个文件作为容器console的输出，如果不指定，将输出到终端

-s 指定配置

如果没有指定命令，lxc-start 将要运行 /sbin/init

例如：lxc-start -n foo -f foo.conf -d /bin/bash

注：1.如果容器还不存在，lxc-start会自动创建一个,容器停止运行后会被自动销毁

2.lxc-start配置优先级与lxc-execute相同

3.lxc-start 与lxc.execute的异同：

lxc-start 和 lxc-execute都可以在容器中启动进程，区别在于lxc-start直接创建进程，lxc-execute先创建lxc-init进程，然后在lxc-init中fork一个进程来执行。(关于第4点，lxc-init所占的是一个什么样的地位？)

The orphan process group and daemon are not supported by this command,
use the lxc-execute command instead
If no command is specified, lxc-start will use the default "/sbin/init"
command to run a system container.

4.lxc-start用于在容器启动system，lxc-execute用于在容器执行应用程序

lxc-kill 发送信号给容器中的第一个用户进程（容器内部进程号为2的进程）

>     用法：lxc-kill -n name SIGNUM

-n 后面跟容器名

SIGNUM 信号 （此参数可选，默认SIGKILL）

例如：lxc-kill -n foo

lxc-stop 用于停止容器中所有的进程

>     用法：lxc-stop -n name

-n后面跟要停止的容器名

例如:lxc-stop --n foo

lxc-destroy 用于销毁容器

>     用法：lxc-destroy -n name

-n后面跟要停止的容器名

例如: lxc-destroy --n foo

lxc-cgroup 用于获取或调整与cgroup相关的参数

>     用法：lxc-cgroup -n name subsystem value

-n 后面跟要调整的容器名

例如： lxc-cgroup -n foo devices.list

lxc-cgroup -n foo cpuset.cpus "0,3"

lxc-info 用户获取一个容器的状态

>     用法:lxc-info -n name

-n后面跟操作的容器名

例如: lxc-info --n foo

注：容器的状态有：STARTING RUNNING STOPPING STOPPED ABORTING

lxc-monitor 监控一个容器状态的变换，当一个容器的状态变化时，此命令会在屏幕上打印出容器的状态

>     用法:lxc-monitor -n name

例如：lxc-monitor -n foo

lxc-ls 列出当前系统所有的容器

>     用法：lxc-ls

例如：lxc-ls

lxc-ps 列出特定容器中运行的进程

>     用法:lxc-ps

例如:lxc-ps -n foo


## 学习资料 ##

[淘宝Cgroup资源控制实践](http://sdrv.ms/19jo47p)
