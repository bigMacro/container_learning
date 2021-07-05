##  创建user namespaces

通过clone()或者unshare()指定CLONE_NEWUSER创建user namespaces。从Linux 3.8开始，创建user namespaces不需要任何特权。



demo_userns.c创建一个新的user namespace，并且输出它的eUID, eGID和capabilities。

```
$ id -u
1000
$ id -g
1000
$ ./demo_userns
eUID = 65534;  eGID = 65534;  capabilities: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+ep
```

> 编译demo_usernc.c时可以使用gcc -o demo_userns -Wl,--no-as-needed -lcap demo_userns.c

user namespace的说明如下：

1. capabilities表示进程的permitted capabilities和effective capabilites。当一个user namespace创建的时候，这个user namespace的第一个进程会被赋予所有的capabilities，使得这个进程能够执行任何初始化操作。值得说明的是尽管在新的user namespace中的第一个进程拥有所有的capabilities，但是这个进程在父namespace中并没有capabilities。
2. getuid()和getgid()总是返回调用该系统调用的进程所属的user namespace的ID。如果没有定义映射，那么将会返回定义在/proc/sys/kernel/overflowuid中的值(默认为65534)。
3. user namespace也是有层级的，一个user namespace的父user namespace是那个创建当前user namespace的进程所属的user namespace。

## user IDs 和 group IDs的映射

user IDs和group IDs的映射关系定义在/proc/PID/uid_map和/proc/PID/gid_map文件中，定义格式为：

```
ID-inside-ns ID-outside-ns length
```

其中ID-insize-ns和length定义了当前user namespace内需要映射的ID范围；ID-outside-ns定义了user namespace外的ID范围，它的解释依赖于打开映射文件的进程是否和映射文件所属进程的user namespace是否在同一user namespace中：

* 如果两个进程在同一个user namespace中，那么ID-outside-ns解释为父user namespace中的ID。通常这种情况是当前进程在写自己的映射文件。
* 如果两个进程不在同一个user namespace中，那么ID-outside-ns解释为打开映射文件的进程所属的user namespace中的ID。通常这种情况是当前进程在定义关联到自己user namespace的映射关系。



为了验证映射关系，我们在一个terminal中运行demo_userns：

```
$ ./demo_userns x
eUID = 65534;  eGID = 65534;  capabilities: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+ep
```

然后在另一个terminal中找到demo_userns的PID(当前的user namespace为demo_userns的父user namespace)，修改其映射关系：

```
root@ubuntu:~# ps -C demo_userns -o 'pid uid comm'
  PID   UID COMMAND
 5296  1000 demo_userns
 5297  1000 demo_userns
root@ubuntu:~# echo '0 1000 1' > /proc/5297/uid_map
root@ubuntu:~# echo '0 1000 1' > /proc/5297/gid_map
```

观察demo_userns输出的eUID和eGID：

```
eUID = 0;  eGID = 0;  capabilities: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+ep
```



## 映射文件规则

定义映射文件是一次性的操作，即只能执行一次写uid_map(gid_map)文件。/proc/PID/uid_map(gid_map)的拥有者是创建这个进程所属user namespace的用户，并且只能由这个用户写(特权用户也可)。除此之外，还需要遵循以下规则：

1. 写进程必须在PID所属的user namespace中拥有CAP_SETUID(CAP_SETGID)权限
2. 写进程必须是PID所属的user namespace或者父user namespace中的进程
3. 以下至少一条需要满足：
   1. 写入映射文件的数据包括写进程在父user namespace的eUID(eGID)到当前user namespace的UID(GID)的映射。这条规则允许user namespace中的init进程为它自己的用户写映射关系。
   2. 写进程在父user namespace中拥有CAP_SETUID(CAP_SETGID)权限。这条规则允许父user namespace中的进程定义映射规则。



## 权限、execve()和user ID 0

我们使用nc_child_exec在一个新的user namespace中运行bash，然后我们尝试定义UID映射关系：

```
root@ubuntu:~/namespace# ./ns_child_exec -U bash
nobody@ubuntu:/namespace$ echo '0 1000 1' > /proc/$$/uid_map
bash: echo: write error: Operation not permitted
```

这个错误说明bash在新的user namespace中没有权限，我们可以通过以下命令验证：

```
nobody@ubuntu:/namespace$ id -u
65534
nobody@ubuntu:/namespace$ id -g
65534
nobody@ubuntu:/namespace$ cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
```

问题的原因在于execve()执行bash时如果执行execve()的进程的eUID不为0，那么进程的权限将会被清空。为了避免这个问题，需要在执行execve()之前创建UID映射。userns_child_exec.c和ns_child.exec.c类似，只不过它支持-M和-G选项，这两个选项能够在新user namespace中创建user和group的映射。前面介绍过，可以在父进程和子进程中创建UID映射。但是如果在子进程中创建映射的话，映射的ID需要包括子进程的eUID。为了能够映射任意的UID，userns_child_exec.c通过父进程设置UID映射。此外，通过父进程写映射文件设置UID/GID需要为userns_child_exec设置权限：

```
setcap 'CAP_SETUID,CAP_SETGID,CAP_DAC_OVERRIDE+ep' userns_child_exec
```

我们重新以下命令进行实验，就能看到映射正常地建立：

```
$ ./userns_child_exec -U -M '0 1000 1' -G '0 1000 1' bash
root@ubuntu:/namespace# id -u
0
root@ubuntu:/namespace# id -g
0
root@ubuntu:/namespace# cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
```

## 观测user ID和group ID的映射

根据ID-outside-ns的解释规则，如果打开/proc/PID/uid_map(gid_map)的进程与PID所在的user namespace相同，ID-outside-ns解释为父user namespace的ID，如果user namespace不一致，则解释为打开文件的进程所在的user namespace的ID。我们可以使用这条规则查看ID映射治理情况。



首先创建一个user namespace，并且在里面运行shell：

```
$ id -u
1000
$ exit
root@ubuntu:~# cd /namespace/
root@ubuntu:/namespace# su test
$ id -u
1000
$ ./userns_child_exec -U -M '0 1000 1' -G '0 1000 1' bash
root@ubuntu:/namespace# echo $$
8895
root@ubuntu:/namespace# cat /proc/8895/uid_map
         0       1000          1
root@ubuntu:/namespace# id -u
0
```

现在，我们切换到另一个ternimal创建另外一个user namespace，并且使用不同的ID映射：

```
$ ./userns_child_exec -U -M '200 1000 1' -G '200 1000 1' bash
groups: cannot find name for group ID 200
I have no name!@ubuntu:/namespace$ cat /proc/self/uid_map
       200       1000          1
I have no name!@ubuntu:/namespace$ id -u
200
I have no name!@ubuntu:/namespace$ echo $$
8953
I have no name!@ubuntu:/namespace$
```

继续在第二个terminal中查看第一个user namespace中的进程的映射关系：

```
I have no name!@ubuntu:/namespace$ cat /proc/8895/uid_map
         0        200          1
```

这意味着第一个user namespace的UID 0映射到了当前user namespace的UID为200。如果我们在第一个ternimal查看第二个user namespace中的进程映射关系：

```
root@ubuntu:/namespace# cat /proc/8953/uid_map
       200          0          1
```

同样的，因为打开8953进程的user namespace与8953进程所在的user namespace并不相同，所以ID-outside-n s为0。当然，如果在相同的user namespace，ID-outside-ns就是父user namespace的ID，因为我们在两个user namespace中都将ID映射到了1000，所以我们可以通过在各自的user namespace执行以下命令进行验证：

```
root@ubuntu:/namespace# cat /proc/8895/uid_map
         0       1000          1
 ---------------------------------------------
 I have no name!@ubuntu:/namespace$ cat /proc/8953/uid_map
       200       1000          1
```

