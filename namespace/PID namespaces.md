## pidns_init_sleep

pidns_init_sleep.c通过clone()创建一个子进程，同时通过CLONE_NEWPID创建一个PID namespace，并且将子进程的PID namespace设置为这个PID namespace。除此之外，子进程会将profs挂载在指定的挂载点上。

```
root@ubuntu:~/namespace# ./pidns_init_sleep /proc2
PID returned by clone(): 2141
childFunc(): PID  = 1
childFunc(): PPID = 0
Mounting procfs at /proc2
```

PID namespace会形成一个层级：在父进程的PID namespace中，父进程看到子进程的PID为2141，但是在子进程中，它看不到上层的PID namespace，所以它的父进程ID为0。同时，子进程在自己的PID namespace中看到自己的PID为1，这与上层的PID namespace并不相同。



## /proc/PID 和 PID namespaces

在Linux上每个进程都有一个/proc/PID目录，这个目录包含了在当前及其后代PID namespace中关于该进程的信息。为了使对应PID namespace的/proc/PID目录可见，需要在对应的PID namespace中将procfs挂载。对于一些工具(比如ps)，它们依赖/proc目录下的信息，所以procfs的挂载点必须是/proc目录。为了防止影响上层PID namespace的/proc挂载点，子进程要么使用CLONE_NEWNS使得自己与父进程的mount namespace隔离；要么子进程使用chroot()改变root目录，然后将procfs挂载在/proc目录。



回到pidns_init_sleep.c，我们通过/proc和/proc2目录观察父子进程的PID namespace。运行pidns_init_sleep之后，我们将其放到后台执行：

```
root@ubuntu:~/namespace# ./pidns_init_sleep /proc2
PID returned by clone(): 2360
childFunc(): PID  = 1
childFunc(): PPID = 0
Mounting procfs at /proc2
^Z
[1]+  Stopped                 ./pidns_init_sleep /proc2
```

之后我们查看父子进程pidns_init_sleep和sleep的PID：

```
root@ubuntu:~/namespace# ps -C sleep -C pidns_init_sleep -o "pid ppid stat cmd"
  PID  PPID STAT CMD
 2359  1399 T    ./pidns_init_sleep /proc2
 2360  2359 S    sleep 600
```

在/proc目录下查看父子进程的PID namespace:

```
root@ubuntu:~/namespace# readlink /proc/2359/ns/pid
pid:[4026531836]
root@ubuntu:~/namespace# readlink /proc/2360/ns/pid
pid:[4026532221]
```

在/proc2目录下查看子进程的PID namespace中看到的信息：

```
root@ubuntu:~/namespace# ls -d /proc2/[1-9]*
/proc2/1
root@ubuntu:~/namespace# cat /proc2/1/status | egrep '^(Name|PP*id)'
Name:	sleep
Pid:	1
PPid:	0
```



## 嵌套的PID namespaces

前面提到过，PID namespace是有层级关系的，在一个PID namespace中，能看到当前PID namespace和子孙后代PID namespace中的进程。这里的“看到”是指能够通过系统调用对特定的PID进行操作(比如通过kill()给指定进程发送信号)。但是当前PID namespace中不能看到父PID namespace及祖先PID namespace中的进程。



这里我们通过multi_pidns.c展示不同PID namespace层级上进程的可见性。multi_pidns.c递归地创建一个子进程，并且将procfs挂载到不同挂载点。递归的结束是一个sleep程序。

```
root@ubuntu:~/namespace# ./multi_pidns 5
Mounting procfs at /proc4
Mounting procfs at /proc3
Mounting procfs at /proc2
Mounting procfs at /proc1
Mounting procfs at /proc0
Final child sleeping
```

我们将这个程序挂起，然后查看每一层procfs能看见的进程：

```
^Z
[1]+  Stopped                 ./multi_pidns 5
root@ubuntu:~/namespace# ls -d /proc4/[1-9]*
/proc4/1  /proc4/2  /proc4/3  /proc4/4  /proc4/5
root@ubuntu:~/namespace# ls -d /proc3/[1-9]*
/proc3/1  /proc3/2  /proc3/3  /proc3/4
root@ubuntu:~/namespace# ls -d /proc2/[1-9]*
/proc2/1  /proc2/2  /proc2/3
root@ubuntu:~/namespace# ls -d /proc1/[1-9]*
/proc1/1  /proc1/2
root@ubuntu:~/namespace# ls -d /proc0/[1-9]*
/proc0/1
```

最后，我们看一下sleep进程在各个PID namespace中的PID：

```
root@ubuntu:~/namespace# grep -H 'Name:.*sleep' /proc?/[1-9]*/status
/proc0/1/status:Name:	sleep
/proc1/2/status:Name:	sleep
/proc2/3/status:Name:	sleep
/proc3/4/status:Name:	sleep
/proc4/5/status:Name:	sleep
```

> ​	注意：结束上面的实验之后会残留挂载点和挂载目录，需要我们手动回收:
>
> root@ubuntu:~/namespace# umount /proc?
>
> root@ubuntu:~/namespace# rmdir /proc?