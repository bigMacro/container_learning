## 简介

mount namespace是第一个加入到Linux的namespace。它用来隔离mount namespace内的进程所能看到的挂载点。当系统启动时，只有一个mount namespace，之后新的mount namespace可以由clone()使用CLONE_NEWNS参数或者unshared()创建。不管是clone()还是unshare()创建的mount namespace，它都会拷贝调用者的挂载点。之后每个mount namespace能够独立的添加或者删除挂载点，这些变化是否可见取决于挂载点的类型。

## 共享子树

共享子树能够自动地将一个mount namespace中的mount和unmount事件传递到另一个mount namespace中。为了实现控制一个挂载点的事件能否传递到另一个mount namespace，每个挂载点都有一个传递类型标记。传递类型分为：

* MS_SHARED：当前挂载点的mount和unmount事件与共享组的成员共享，即当前挂载点能够从其他成员接收事件，也能将自己的事件传递给其他成员。
* MS_PRIVATE：与MS_SHARED相反，当前挂载点既不能从其他成员接收事件，也不能将自己的事件传递给其他成员。
* MS_SLAVE：slave mount有一个master，master的事件能够传递给slave，但是slave的事件不会传递给master。
* MS_UNBINDABLE：与MS_PRIVATE类似，但是更近一步，当前挂载点不能作为bind mount的源。

对于上面的说明我们有更进一步的说明：

* 传递类型是单个挂载点的配置，在一个mount namespace中，某些挂载点可以被标记为shared，某些挂载点可以被标记为private。
* 传递类型只能决定挂载点直接下层的事件。比如在一个共享挂载点X下面创建一个Y，创建Y的事件会传递到共享组的其他成员；但是共享挂载点X的传递类型不能控制Y下面的事件传递，Y下面的事件是否传递依赖于Y的传递类型。
* 事件是一个抽象的概念，并不存在一个具体的实现，它用来说明一个挂载点的mount或者unmount操作是否会触发另一些挂载点的操作。



## 共享组

共享组的成员能够相互之间传递mount和unmount事件。共享组通过以下两种方式添加成员：

* 创建一个新的mount namespace时，父mount namespace的shared类型的挂载点复制到新的mount namespace，被复制的挂载点会被加入共享组
* shared类型的挂载点作为bind mount源使用时，目标挂载点会被加入共享组



为了进行实验，我们创建两个虚拟块设备：

```
root@ubuntu:/namespace# dd if=/dev/zero of=/namespace/loopfile3 count=102400
102400+0 records in
102400+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 0.188648 s, 278 MB/s
root@ubuntu:/namespace# dd if=/dev/zero of=/namespace/loopfile4 count=102400
102400+0 records in
102400+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 0.201599 s, 260 MB/s
root@ubuntu:/namespace# losetup -fP loopfile3
root@ubuntu:/namespace# losetup -fP loopfile4
root@ubuntu:/namespace# mkfs.ext4 /namespace/loopfile3
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done
Creating filesystem with 51200 1k blocks and 12824 inodes
Filesystem UUID: 3158c680-1c00-41ab-b6f9-8f8484618cce
Superblock backups stored on blocks:
	8193, 24577, 40961

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@ubuntu:/namespace# mkfs.ext4 /namespace/loopfile4
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done
Creating filesystem with 51200 1k blocks and 12824 inodes
Filesystem UUID: 557b063a-9c59-4781-94f6-525c4cc58f5b
Superblock backups stored on blocks:
	8193, 24577, 40961

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

```

我们在初始mount namespace(系统启动时创建的mount namespace)中，标记root挂载点为private，并创建两个shared挂载点：

```
root@ubuntu:/namespace# mount --make-private /
root@ubuntu:/namespace# mount --make-shared /dev/loop2 /namespace/A
root@ubuntu:/namespace# mount --make-shared /dev/loop3 /namespace/B
```

然后在另一个ternimal中，使用unshared命令创建一个新的mount namespace，并且运行一个shell：

```
unshare -m --propagation unchanged sh
```

回到第一个ternimal，从/namespace/A创建一个bind mount：

```
root@ubuntu:/namespace# mkdir C
root@ubuntu:/namespace# mount --bind /namespace/A /namespace/C
```

最后形成的mount namespace关系如下：

```
           mount NS 1                          mount NS 2
             /namespace                           /namespace
     A     C      B                          A‘          B’
```

其中：

* 第一个共享组包括A，C，A‘，其中A’是mount NS 2创建是拷贝过去的，C是通过bind mount加入的
* 第二个共享组包括B，B‘，其中B’是mount NS 2创建时拷贝过去的。因为创建mount NS 2时没有C，所以C没有被拷贝；并且/是private类型的，所以创建C也不会传递到mount NS 2

## /proc/PID/mountinfo中关于传递类型和共享组的信息

/proc/PID/mountinfo显示了关于PID这个进程所在的mount namespace的挂载点的信息。同一个mount namespace中的进程看到的信息都是一样的。相比于/proc/PID/mounts，它能够提供更多(比如传递类型、共享组)的信息。

对于shared类型的挂载点，在/proc/PID/mountinfo中会包含shared:N的标记，其中，shared表示挂载类型为shared，并且它所在共享组的标识符为N。例如：

```
root@ubuntu:/namespace# cat /proc/self/mountinfo | sed 's/ - .*//'
28 0 252:1 / / rw,relatime
352 28 7:2 / /namespace/A rw,relatime shared:206
360 28 7:3 / /namespace/B rw,relatime shared:207
418 28 7:2 / /namespace/C rw,relatime shared:206
```

通过这个输出，我们可以看到：

* /namesapce/A，/namespace/B，/namespace/C是共享类型的，并且A和C在共享组206内，B在共享组207内
* 每行前两个数字代表当前挂载点在当前mount namespace内的编号和父挂载点的编号，其中A,B,C都是root mount的孩子

## 默认值

新挂载点的传递类型的默认值遵循以下规则：

* 如果父挂载点的类型是shared，那么新挂载点的默认传递类型为shared
* 否则，传递类型为private

因为root mount是private的，所以所有的孩子挂载点都默认是private。但是有些Linux发行版本，将root mount修改为了shared，这样方便使用。前面实验使用了`mount --make-rprivate /`确保root mount是private的，而`unshare -m --propagation unchanged <cmd>`防止传递类型改变。

