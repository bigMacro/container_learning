## PID namespace的init进程

PID namespace中第一个创建的进程的PID为1。和操作系统PID为1的进程类似，它可以执行一些初始化的操作，并且会变成当前namespace中孤儿进程的父进程。

ns_child_exec.c通过clone()创建子进程，并且创建通过参数指定的namespace。下面的例子使用-p选项创建了一个新的PID namespace，并输出子进程在新PID namespace的PID：

```
root@ubuntu:~/namespace# ./ns_child_exec -p sh -c 'echo $$'
1
```



simple_init.c实现了两个init的功能：

1. 初始化namespace，并且提供了简单的shell以支持在namespace中执行用户程序
2. 使用waitpid()回收停止运行的进程状态

```
root@ubuntu:~/namespace# ./ns_child_exec -p ./simple_init
init$
```

init$表示simple_init已经准备好读取并执行一个shell命令。



orphan.c是一个用来模拟在一个PID namespace中变成孤儿之后被init进程收养的程序。它会通过fork()创建一个子进程。父进程在创建子进程之后退出；而子进程通过轮询父进程，发现自己变成孤儿之后就退出。下面是使用这三个程序的实验：

```
root@ubuntu:~/namespace# ./ns_child_exec -p ./simple_init -v
	init: my PID is 1
init$ ./orphan
	init: created child 2
Parent (PID=2) created child with PID 3
Parent (PID=2; PPID=1) terminating
	init: SIGCHLD handler: PID 2 terminated
init$
Child  (PID=3) now an orphan (parent PID=1)
Child  (PID=3) terminating
	init: SIGCHLD handler: PID 3 terminated
```



## init进程和信号

对于操作系统的init进程，为了保持操作系统的稳定运行，只有建立了signal handler的信号才会被传递给init进程，其他信号都会被忽略。

PID namespace实现了类似的行为。PID namespace中的其他进程只能发送init进程建立了signal handler的信号给init进程。但是，内核仍然能够构造并发送信号(比如硬件异常、SIGTOU等结束信号、定时器超时)给init进程。

PID namespace父(或者祖先)PID namespace中的进程和PID namespace内的进程类似，只能够发送init进程建立了signal handler的信号给init进程。除此之外，还能够发送SIGKILL和SIGSTOP信号给init进程，init进程不能捕获这两个进程，所以只能退出。init进程退出之后，内核会给PID namespace内的所有进程发送SIGKILL信号。

通常情况下，init进程退出之后PID namespace将会被销毁。但是，如果PID namespace内某个进程的/proc/PID/ns/pid文件被绑定挂载或者处于打开状态，那么这个PID namespace不会被销毁。不过这种情况下这个PID namespace已经没什么用了，即使在这个PID namespace创建新进程(通过setns()加fork())也会失败(返回ENOMEM)。

## 挂载procfs文件系统

为了在PID namespace中使用ps命令，我们需要将procfs挂载在/proc目录下。同时，为了不影响PID namespace之外的进程，我们使用-m选项使得simple_init能够在一个隔离的mount namespace中运行。

```
root@ubuntu:~/namespace# ./ns_child_exec -p -m ./simple_init
init$ mount -t proc proc /proc
init$ ps a
  PID TTY      STAT   TIME COMMAND
    1 pts/0    S      0:00 ./simple_init
    3 pts/0    R+     0:00 ps a
```

### unshare()和setns()

前面介绍unshare()时说unshare()会根据flag创建指定的namepsace，并且将调用者放到新创建的namespace中。但是对于PID namespace来说，情况有些特殊：它并不会将调用者放到新的PID namespace中，而是将调用者的子进程放到新的PID namespace中，并且第一个这样的子进程将会变成新的PID namespace中的init进程。

如果setns()的参数fd是调用者子孙PID namespace的进程的PID namespace的文件描述符(/proc/PID/ns/pid)，setns()也不会将调用者移到对应的PID namespace，而是将调用者的子进程放到新的PID namespace中。



ns_run.c将会使用setns()加入通过-n选项指定的namespace中，然后执行给定的命令；如果指定了-f选项，那么将会使用fork创建一个子进程执行给定的命令。开始实验前，我们使用simple_init准备一个PID namespace：

```
root@ubuntu:~/namespace# ./ns_child_exec -p ./simple_init -v
	init: my PID is 1
init$
```

然后我们切换到另外一个terminal使用ns_run去运行orphan：

```
root@ubuntu:~/namespace# ps -C sleep -C simple_init
  PID TTY          TIME CMD
 3115 pts/0    00:00:00 simple_init
root@ubuntu:~/namespace# ./ns_run -f -n /proc/3115/ns/pid ./orphan
Parent (PID=2) created child with PID 3
Parent (PID=2; PPID=0) terminating
root@ubuntu:~/namespace#
Child  (PID=3) now an orphan (parent PID=1)
Child  (PID=3) terminating
```

通过Parent PPID=0可以看出父orphan进程与创建它的进程(ns_run)不在一个PID namespace中。此时的进程关系图如下：

```
------------------------------------------------------
|                      parent namespace              |
|     ns_child_exec                  ns_run          |
------------|---------------------------|-------------
            |                           |
------------|---------------------------|-------------
|           |														|							|
|          \|/                         \|/            |
|       simple_init                 orphan "Parent"   |
|          (PID 1)                    (PID 2)         |
|                                       |             |
|                                      \|/            |
|                                   orphan "Child"    |
|                                     (PID 3)         |
|                     child namespace                 |
-------------------------------------------------------
```



父orphan进程创建了PID=3的子orphan进程之后就退出了。ns_run在父orphan进程退出之后也退出了。最后PID=3的子进程被PID=1的进程(simple_init)收养。这点可以通过前面运行simple_init的terminal的输出可以看出:

```
init$ 	init: SIGCHLD handler: PID 3 terminated
```

此时的进程关系图如下：

```
------------------------------------------------------
|                      parent namespace              |
|     ns_child_exec                                  |
------------|-----------------------------------------
            |                           
------------|-----------------------------------------
|           |																				 |
|          \|/                                       |
|       simple_init                                  |
|          (PID 1)  \                                |
|                    \                               |
|                    _\|                             |
|                 orphan "Child"                     |
|                    (PID 3)                         |
|                     child namespace                |
------------------------------------------------------
```



这里需要强调setns()和unshare()对于PID namespace的特殊处理是合理的，因为移动调用者到新的PID namespace会导致PID的变化，而很多的用户进程和库都依赖于进程PID不会变化这一假设。所以可以下一个结论：进程在创建时就决定了其所属的PID namespace，并且之后不会改变。