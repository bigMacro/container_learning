为了实现全局资源在不同进程中的隔离，Linux提供了六种不同类型的namespace：

* Mount namespace(CLONE_NEWNS, Linux 2.4.19)：隔离进程所能看见的文件系统挂载点，使得不同mount namespace的进程能够拥有不同的文件系统层级。
* UTS namespace(CLONE_NEWUTS, Linux 2.6.29)：隔离uname()系统调用返回的两个系统标识符nodename和domainname。
* IPS namespace(CLONE_NEWIPC, Linux 2.6.19)：隔离IPC资源(message queues, semaphore sets, shared memory segments)。
* PID namesapce(CLONE_NEWPID, Linux 2.6.24)：隔离PID，使得不同PID namespace的进程可以拥有相同的PID。PID namespace允许每个容器拥有自己的init(PID 1)进程，它能够管理各种系统初始化任务以及孤儿进程。PID namespace可以嵌套，一个进程可以看见它自己所在的PID namespace中的进程，也能够看见它所嵌套的PID namespace中的进程。一个进程在不同的PID namespace中所观察到的进程号是不同的。
* Network namespace(CLONE_NEWNET, Linux 2.6.24-2.6.29)：隔离网络相关的系统资源。每个network namespace拥有它自己的网络设备、IP地址、路由表、/proc/net目录、端口号等等。
* User namespace(CLONE_NEWUSER, Linux 2.6.23-3.8)：隔离user ID和group ID，使得同一个进程在不同的user namespace中拥有不同的user ID 和group ID。这会产生一个有趣的现象：一个进程在一个user namespace之外关联一个普通用户，而在user namespace之内关联root用户，即这个进程在一个user namespace之外没有任何特权，而在user namespace之内却能够执行所有特权操作。



后面的文章将通过一些例子详细介绍这些namespace：

1. the namespace API
   1. demo_uts_namespace.c：UTS namespace例子
   2. ns_exec.c：使用setns()加入一个namespace，然后执行一条命令
   3. unshare.c：脱离一个namespace，然后执行一条命令
2. PID namespaces
   1. pidns_init_sleep.c：PID namespace例子
   2. multi_pidns.c：在嵌套的PID namespace中创建一系列的子进程
3. more on PID namespaces
   1. ns_child_exec.c：在一个新的namespace中创建一个执行shell命令的子进程
   2. simple_init.c：在PID namespace中作为init进程使用的进程
   3. orphan.c：模拟子进程在父进程退出变成孤儿进程之后，被init进程所收养
   4. ns_run.c：使用setns()加入一个或者多个namespace，然后(可能在子进程中)执行一条命令
4. user namespace
   1. demo_userns.c：创建一个user namespace然后显示进程的用户信息和权限信息
   2. userns_child_exec.c：类似于ns_child_exec.c，增加了一些user namespace相关的参数
5. more on user namespace
   1. userns_setns_test.c：测试在两个不同的user namespace中setns()的操作
6. network namespace
7. Mount namespaces and shared subtrees
8. Mount namespaces, mount propagation, and unbindable mounts