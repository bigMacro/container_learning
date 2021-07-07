## MS_SHARED和MS_PRIVATE例子

假设我们在初始的mount namespace存在两个挂载点/namespace/S和/namespace/P，其中/namespaceS是shared类型的，而/namespace/P是private类型的：

```
root@ubuntu:/namespace# mount --make-shared /dev/loop0 /namespace/S
root@ubuntu:/namespace# mount --make-private /dev/loop1 /namespace/P
root@ubuntu:/namespace# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
343 28 7:0 / /namespace/S rw,relatime shared:1
344 28 7:1 / /namespace/P rw,relatime
```

在另一个terminal中，我们创建一个新的mount namespace：

```
root@ubuntu:/namespace# unshare -m --propagation unchanged sh
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/S rw,relatime shared:1
411 360 7:1 / /namespace/P rw,relatime
```

我们可以看到新的mount namespace拷贝了初始mount namespace的挂载点，除了挂载点的编号不一样，其他都是一样的。



在第二个terminal中，我们分别在/namespace/S和/namespace/P下面创建挂载点：

```
# mkdir /namespace/S/a
# mount /dev/loop2 /namespace/S/a
# mkdir /namespace/P/b
# mount /dev/loop3 /namespace/P/b
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/S rw,relatime shared:1
411 360 7:1 / /namespace/P rw,relatime
412 410 7:2 / /namespace/S/a rw,relatime shared:205
414 411 7:3 / /namespace/P/b rw,relatime
```

从上面可以看到，/namespace/S/a是shared类型的，而/namespace/P/b是private类型的。



回到第一个ternimal，我们可以看到/namespace/S下面新创建的挂载点，而不能看到/namespace/P下面新创建的挂载点：

```
root@ubuntu:/namespace# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
343 28 7:0 / /namespace/S rw,relatime shared:1
344 28 7:1 / /namespace/P rw,relatime
413 343 7:2 / /namespace/S/a rw,relatime shared:205
```

## MS_SLAVE例子

在实验开始之前，我们首先在初始mount namespace创建两个shared类型的挂载点：

```
root@ubuntu:/namespace# mount --make-shared /dev/loop0 /namespace/X
root@ubuntu:/namespace# mount --make-shared /dev/loop1 /namespace/Y
root@ubuntu:/namespace# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
343 28 7:0 / /namespace/X rw,relatime shared:1
344 28 7:1 / /namespace/Y rw,relatime shared:205
```

然后在第二个ternimal创建一个新的mount namespace，它将初始mount namespace的挂载点拷贝过来：

```
root@ubuntu:/namespace# unshare -m --propagation unchanged sh
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/X rw,relatime shared:1
411 360 7:1 / /namespace/Y rw,relatime shared:205
```

在新的mount namespace中我们标记一个shared类型的挂载点为slave类型：

```
# mount --make-slave /namespace/Y
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/X rw,relatime shared:1
411 360 7:1 / /namespace/Y rw,relatime master:205
```

我们可以看到挂载点/namespace/Y被标记为master:2，它表示这个挂载点是共享组ID为2的slave。

我们继续在新的mount namespace中分别在/namespace/X和/namespace/Y中创建新的挂载点：

```
# mount /dev/loop2 /namespace/X/a
# mount /dev/loop3 /namespace/Y/b
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/X rw,relatime shared:1
411 360 7:1 / /namespace/Y rw,relatime master:205
412 410 7:2 / /namespace/X/a rw,relatime shared:206
414 411 7:3 / /namespace/Y/b rw,relatime
```

我们可以看到，/namespace/X/a被创建为shared类型，而/namespace/Y/b被创建为private类型。

回到第一个ternimal，我们可以看到/namespace/X/a被传递到了/namespace/X下面，但是/namespace/Y/b并没有：

```
root@ubuntu:/namespace# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
343 28 7:0 / /namespace/X rw,relatime shared:1
344 28 7:1 / /namespace/Y rw,relatime shared:205
413 343 7:2 / /namespace/X/a rw,relatime shared:206
```

接着，我们在第一个ternimal中的/namespace/Y下面创建一个新的挂载点：

```
root@ubuntu:/namespace# mkdir /namespace/Y/c
root@ubuntu:/namespace# mount /dev/loop4 /namespace/Y/c
root@ubuntu:/namespace# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
343 28 7:0 / /namespace/X rw,relatime shared:1
344 28 7:1 / /namespace/Y rw,relatime shared:205
413 343 7:2 / /namespace/X/a rw,relatime shared:206
415 344 7:4 / /namespace/Y/c rw,relatime shared:207
```

然后回到第二个ternimal检查新的mount namespace中的/namespace/Y/c已经通过master共享组传递过来了：

```
# cat /proc/self/mountinfo | grep 'namespace' | sed 's/ - .*//'
410 360 7:0 / /namespace/X rw,relatime shared:1
411 360 7:1 / /namespace/Y rw,relatime master:205
412 410 7:2 / /namespace/X/a rw,relatime shared:206
414 411 7:3 / /namespace/Y/b rw,relatime
416 411 7:4 / /namespace/Y/c rw,relatime master:207
```

## bind mounts

bind mount可以使一个文件或者目录在另一个位置的目录中被看到。它和硬连接有些类似，但是有以下区别：

* 不能创建一个到目录的硬连接，但是可以bind mount到一个目录
* 硬连接只能在相同的文件系统中创建，但是bind mount可以跨文件系统
* 硬连接会修改文件系统内容，但是bind mount只记录在mount namespace的mount list表中

我们首先创建一个包含文件的目录，然后bind mount到另一个位置：

```
root@ubuntu:/namespace# mkdir dir1
root@ubuntu:/namespace# touch dir1/x
root@ubuntu:/namespace# mkdir dir2
root@ubuntu:/namespace# mount --bind dir1 dir2
root@ubuntu:/namespace# ls dir2
x
```

然后我们在新的挂载点创建一个文件，接着在原来的目录也能观察到，这意味着bind mount引用了相同的目录对象：

```
root@ubuntu:/namespace# touch dir2/y
root@ubuntu:/namespace# ls -l dir1
x y
```

默认情况下，当一个目录被bind mount到另一个位置时，这个目录下的挂载点不会被复制到目标位置，除非在mount()系统调用使用MS_REC递归参数或者在mount命令中使用--rbind选项。

## MS_UNBINDABLE例子

MS_UNBINDABLE是为了解决一个被称为”挂载点爆炸“的问题，即将一个挂载树的高层挂载点递归的bind mount到低层的挂载点。

假设我们现在有两个挂载点：

```
# mount | awk '{print $1, $2, $3}'
/dev/sda1 on /
/dev/sdb6 on /mntX
```

然后我们递归的将root目录bind mount到一个目录。为了防止其他影响，我们在一个新的mount namespace中运行：

```
# unshare -m sh
# mount --make-rslave /
# mount --rbind / /home/cecilia
# mount | awk '{print $1, $2, $3}'
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
```

然后我们重复地为第二个用户执行类似的操作：

```
# mount --rbind / /home/henry
# mount | awk '{print $1, $2, $3}'
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sda1 on /home/henry
/dev/sdb6 on /home/henry/mntX
/dev/sda1 on /home/henry/home/cecilia
/dev/sdb6 on /home/henry/home/cecilia/mntX
```
在/home/henry下面，我们不仅添加了/mntX目录，也递归地添加了/home/cecilia目录。如果我们继续为第三个用户执行同样的操作，挂载目录就会爆炸。
我们通过--make-unbindable选择来禁止复制bind mount。

```
# mount --rbind --make-unbindable / /home/cecilia
# cat /proc/self/mountinfo | grep /home/cecilia | sed 's/ - .*//' 
108 83 8:2 / /home/cecilia rw,relatime unbindable
...
```
在/proc/self/mountinfo中，禁止复制bind mount通过unbindable标记表示。现在我们为之前的用户都加上--make-unbindable选项，挂载点爆炸的问题就背解决了：
```
# mount --rbind --make-unbindable / /home/henry
# mount --rbind --make-unbindable / /home/otto
# mount | awk '{print $1, $2, $3}'
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sda1 on /home/henry
/dev/sdb6 on /home/henry/mntX
/dev/sda1 on /home/otto
/dev/sdb6 on /home/otto/mntX
```
