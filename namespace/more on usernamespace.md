## User namespace与权限

user namespace对于effective capabilities的解释如下：

1. 如果一个进程在对应的user namespace拥有CAP_SYS_ADMIN权限，那么进程使用setns()切换user namespace之后它将在新的user namespace中获取所有的权限。但是它仅限于操作当前namespace中的资源。
2. 一个进程在一个user namespace中的权限还依赖于namespace的关系：
   1. 如果一个进程属于一个user namespace并且它的effective capabilities被设置，那么它在user namespace就有相应的权限
   2. 如果一个进程在user namespace中有权限，那么它在所有的子namespace中都拥有权限
   3. 当一个user namespace被创建的时候，内核会记录user namespace的拥有者为创建者的eUID。任何一个父user namespace的进程只要他的eUID和user namespace的拥有者一致，那么它在这个user namespace就有权限



我们首先用userns_child_exec创建一个user namespace：

```
$ id -u
1000
$ readlink /proc/$$/ns/user
user:[4026531837]
$ ./userns_child_exec -U -M '0 1000 1' -G '0 1000 1' ksh
# echo $$
8999
# readlink /proc/$$/ns/user
user:[4026532222]
```



userns_setns_test.c将会创建一个子进程，并且子进程会被放入新的user namespace中，而父进程仍然在当前user namespace中运行。我们在另一个terminal用这个程序在当前的user namespace中运行shell，然后在新的user namespace中运行子进程：

```
root@ubuntu:/namespace# readlink /proc/$$/ns/user
user:[4026531837]
root@ubuntu:/namespace# ./userns_setns_test /proc/8999/ns/user
parent: readlink("/proc/self/ns/user") ==> user:[4026531837]
parent: setns() succeeded

child:  readlink("/proc/self/ns/user") ==> user:[4026532226]
child:  setns() failed: Operation not permitted
```

因为userns_setns_test父进程的eUID与ksh所在的user namespace的创建者一致，所以它在ksh所在的user namespace拥有所有的权限，因此它能够通过setns()切换user namespace。而userns_setns_test的子进程虽然的eUID与ksh所在的user namespace的创建者一致，但是它所在的user namespace并不是ksh所在的user namespace的父user namespace(而是兄弟关系)，所以它没有CAP_SYS_ADMIN权限去执行setns()。

## user namespace与其他类型namespace

创建其他类型的user namespace需要CAP_SYS_ADMIN权限，但是user namespace不需要权限，而且在新的user namespace中进程拥有所有的权限，所以可以通过两步的方式创建其他类型的namespace。但是实际上不需要执行两步，只需要将其他CLONE_NEW*与CLONE_NEWUSER传递到clone()中就行，内核会自动先创建user namespace，然后在创建其他的namespace。