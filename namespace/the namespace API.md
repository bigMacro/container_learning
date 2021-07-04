本文通过一些例子介绍namespace API。这些API由clone(), unshare()和setns()以及/proc下的一些文件组成。

## clone()

一种创建一个新的namespace的方式是通过clone()来创建一个新进程，它的原型如下：

```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

相比fork()，clone可以通过flag参数控制不同的行为，比如父子进程是否共享虚拟地址空间、打开的文件描述符、信号等资源。如果对应的CLONE_NEW*位被设置，那么一个新的namespace将会被创建，并且新的进程将会被放置在新的namespace中。



demo_uts_namespace.c使用CLONE_NEWUTS参数通过clone()创建了一个UTS namespace。正如总览中的介绍，UTS namespace隔离两个不同的系统描述符：nodename和domainname。下面运行这个例子：

```
root@ubuntu:~/namespace# ./demo_uts_namespace host-uts # 创建UTS namespace需要CAP_SYS_ADMIN权限
PID of child created by clone() is 1416
uts.nodename in child:  host-uts
uts.nodename in parent: ubuntu
```

> 需要注意一个set-user-ID程序通过使用意料之外的hostname而产生错误的行为。比如一个set-user-ID程序能够输出以hostname作为文件名称的一个文件。一个用户可以在一个特定的UTS namespace(比如hostname为/root/documents/file)中运行这个程序，从而访问到/root/documents/file这个文件。



## /proc/PID/ns 文件

每个进程都有一个/proc/PID/ns的目录，这个目录中每种类型的namespace都对应一个link文件。我们可以通过ls -l或者readlink判断链接的文件的inode号判断不同的进程是否属于同一个namespace。

```
root@ubuntu:~/namespace# ls -l /proc/$$/ns # $$表示shell的进程ID
total 0
lrwxrwxrwx 1 root root 0 Jul  3 16:17 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 net -> 'net:[4026531993]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jul  3 16:17 uts -> 'uts:[4026531838]'
root@ubuntu:~/namespace# readlink /proc/$$/ns/uts
uts:[4026531838]
```

除此之外，如果我们打开这个目录下的对应的文件， 只要这个文件描述符保持打开，对应的namespace仍然会一直存在。相同的效果可以通过绑定挂载link文件到另一个位置实现：

```
root@ubuntu:~/namespace# touch ~/uts # 创建挂载点
root@ubuntu:~/namespace# mount --bind /proc/$$/ns/uts ~/uts
```



## setns()

setns()允许调用的进程不再关联当前的namespace，而是关联到一个已经存在的相同类型的namespace：

```c
int setns(int fd, int nstype);
```

其中，fd是/proc/PID/ns目录下某个link文件的文件描述符，它代表着调用进程将加入的namespace。这个文件描述符可以通过打开link文件或者打开绑定挂载的目标文件获取。nstype可以用来校验fd对应的namespace是否和nstype一致，如果设置为0则跳过校验。



使用setns()和execve()允许我们实现一个简单实用的程序：加入一个特定的namespace，然后在这个namespace中执行一条命令。ns_exec.c即是这样一个程序，它通过第一个参数读取/proc/PID/ns/*文件获取并进入对应的namespace，然后将第二个及以后的参数作为命令在这个namespace中执行。

```
root@ubuntu:~/namespace# ./ns_exec ~/uts /bin/bash
root@ubuntu:~/namespace# hostname
ubuntu
root@ubuntu:~/namespace# readlink /proc/$$/ns/uts
uts:[4026531838]
root@ubuntu:~/namespace# exit
exit
root@ubuntu:~/namespace# readlink /proc/$$/ns/uts
uts:[4026531838]
```

## unshare()

最后一个系统调用是unshare()：

```c
int unshare(int flags);
```

unshare()创建flags(指定CLONE_NEW*位)指定的namespace，然后将调用者加入到新创建的namespace中。相比于clone()，unshare()不需要创建一个新的进程(线程)。



unshare.c实用unshare()实现了在一个独立的namespace中执行一个指定的程序。下面将使用unshare.c在一个独立的mount namespace中执行一个shell。

```
root@ubuntu:~/namespace# echo $$
1399
root@ubuntu:~/namespace# readlink /proc/1399/ns/mnt
mnt:[4026531840]
root@ubuntu:~/namespace# ./unshare -m /bin/bash
root@ubuntu:~/namespace# readlink /proc/$$/ns/mnt
mnt:[4026532219]
root@ubuntu:~/namespace# readlink /proc/1399/ns/mnt
mnt:[4026531840]
```

