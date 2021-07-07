## 基本管理操作
network namespace除了可以使用clone()通过CLONE_NEWNET参数创建，也可以通过ip命令创建及管理。ip netns后面可以是network namespace的名字，也可以是属于netowork namepsace的进程ID。

```
root@ubuntu:~# ip netns add netns1
root@ubuntu:~# ip netns exec netns1 ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
root@ubuntu:~# ip netns delete netns1
```

值得注意的是network namespace可以在没有进程的情况下创建，这一点方便于管理。

## 配置
新创建的network namespace除了loop设备外，没有其他任何设备。每个设备(物理或者虚拟)只可以放置在一个network namespace中，并且物理设备只能放置在root network namespace，而虚拟设备可以有创建并放置在其他network namespace中。不同network namespace中的设备可以根据配置进行通信。

新创建的network namespace中的loop设备是停止状态的，需要启动。

```
root@ubuntu:~# ip netns add netns1
root@ubuntu:~# ip netns exec netns1 ping 127.0.0.1
connect: Network is unreachable
root@ubuntu:~# ip netns exec netns1 ip link set dev lo up
root@ubuntu:~# ip netns exec netns1 ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.039 ms
^C
--- 127.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1028ms
rtt min/avg/max/mdev = 0.020/0.029/0.039/0.010 ms
```

不同network namespace可以通过veth进行通信：

```
root@ubuntu:~# ip link add veth0 type peer name veth1
Garbage instead of arguments "name ...". Try "ip link help".
root@ubuntu:~# ip link add veth0 type veth peer name veth1
root@ubuntu:~# ip link set veth1 netns netns1
root@ubuntu:~# ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
root@ubuntu:~# ifconfig veth0 10.1.1.2/24 up
root@ubuntu:~# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.054 ms
^C
--- 10.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.043/0.048/0.054/0.008 ms
root@ubuntu:~# ip netns exec netns1 ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.029 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.052 ms
64 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=0.056 ms
^C
--- 10.1.1.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2038ms
rtt min/avg/max/mdev = 0.029/0.045/0.056/0.014 ms
```

值得注意的是路由表和防火墙规则在不同的network namespace中是不共享的，这一点可以通过以下命令验证：

```
root@ubuntu:~# ip netns exec netns1 route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.1.1.0        0.0.0.0         255.255.255.0   U     0      0        0 veth1
root@ubuntu:~# ip netns exec netns1 iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

这时，可以为root namespace和netns1中的veth设备建立bridge或者在root namespace中通过IP forward建立NAT规则。


network namespace内的非root进程只能访问网络设备和配置，只有root用户才能添加网络设备以及修改配置。

## 用途

1. 管理员可以通过关闭网络设备以确保network namespace内的进程不能和外部通信。
2. 进程处理网络流量可以放置到一个严格的network namespace环境中，比如父进程接受连接，子进程在一个新的netowrk namespace中进行处理。
3. network namespace可以用于测试验证复杂的网络规则。
