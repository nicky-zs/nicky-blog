---
title: Implement Single-Host-Multiple-IPs on Linux
date: 2014-04-09 21:03:06
tags:
  - network namespace
  - veth
  - network bridge
---

单主机可发起的总连接数受限于本地端口数，即基本上只能有（65535 － 1024）个。有时候这个量级的连接数并不够用，比如在测试可支持百万长连接的服务的时候，为了节约测试机的数量，则需要尽量增大单机可发起的连接数。解决这个问题需要利用TCP协议的性质，考虑到TCP的本质是一个四元组（源IP，源Port，目的IP，目的Port），在这种情况下，目的IP和目的Port相同，为了突破源Port的限制，办法就只能是增加主机上源IP的数量。

达成这个目标的办法有很多，比如在一台主机上虚拟出多个主机，这样就可以在每个虚拟主机上发起（65535 - 1024）个连接，但是这样做太重了，而且严重浪费了许多资源。一个更好的解决办法是给一台主机分配多个IP，这种做法很轻，不会造成资源的浪费，而且能很好地达到效果。在客户端发起连接之前，可以通过调用bind系统调用将一个socket绑定到一个指定的IP和Port上再发起连接，就可以完全利用所有的（源IP，源Port）组合，这样理论上可以发起并建立的连接数就是 本机拥有的IP的数量 * (65535 - 1024)。

有很多办法可以给一台主机分配多个IP，但各有各的问题，比如使用IP alias就没办法使用DHCP来获得IP，需要网络管理员来手动指派，自己指派则有可能冲突；之前也试着使用macvlan来解决，但是怀疑是不是被交换机给Ban了，比如STP，导致主机的网络很不稳定，一会儿就上不了网了。这里记录一下Network Namespace + VETH + Bridge的方法，这种方法到目前为止工作良好。

<!-- more -->

一、Linux网络设备
----------------

Bridge直译过来是网桥，但是要比现实世界中的网桥功能更多，更类似于现实世界中的二层交换机。可以将Linux上的多个设备连接到Bridge上，这就相当于是在二层交换机上插入了一根连有终端设备的网络。当有数据包到达时，Bridge会根据二层frame的目的MAC来进行转发、广播或者丢弃。

VETH总是以成对的形式出现，它的作用像是一个管道，送到一端请求发送的数据总是从另一端以请求接受的形式出现。该设备不能被用户程序直接操作，但使用起来比较简单。创建并配置正确后，向其一端输入数据，VETH 会改变数据的方向并将其送入内核网络核心，完成数据的注入。在另一端能读到此数据。

Netns是Linux虚拟出来的一个网络环境，在一个netns中的所有网络环境和配置都是独立的，这个环境包括一个独立的网卡空间，路由表，ARP表，ip地址表，iptables，ebtables等等。它其实是在/var/run/netns/目录下建立了相关的特殊文件，当程序打开了该文件并保持它的文件描述符时，该netns便是alive的，然后再通过调用setns系统调用便能改变一个程序关联的netns。在bash下，通过命令`sudo ip netns exec xx_ns 'command'`便能在xx_ns下执行command。该command可以是bash，从而直接开启一个虚拟网络环境下的bash。

二、操作步骤
-----------

```bash
# 创建 namespace
ip netns add ns1
ip netns add ns2

# 创建 bridge
brctl addbr br0
brctl stp   br0 off
ip link set dev br0 up

### 创建 veth
ip link add tap1 type veth peer name br-tap1
ip link add tap2 type veth peer name br-tap2

# 将 veth 的一端加入到 bridge 中
brctl addif br0 br-tap1
brctl addif br0 br-tap2

# 将 veth 的另一端加入到 namespace 中
ip link set tap1 netns ns1
ip link set tap2 netns ns2

### 打开各设备
# 在各 namespace 中打开 veth 的一端
ip netns exec ns1 ip link set dev tap1 up
ip netns exec ns2 ip link set dev tap2 up

# 在当前系统中打开 veth 的另一端
ip link set dev br-tap1 up
ip link set dev br-tap2 up

### 连接到物理设备
# 关闭后 eth0 将 eth0 加入到 bridge
ifconfig eth0 down
brctl addif br0 eth0

# 以 0.0.0.0 启动 eth0
# 注意必须以 0.0.0.0 启动 eth0，否则不成功
# 有些系统上这里可能会受 NetworkManager 等服务的影响，直接DHCP了
# 那么需要把 NetworkManager 给停掉
ifconfig eth0 0.0.0.0 up

# 使用 DHCP 为各设备获取IP
dhclient br0
ip netns exec ns1 dhclient tap1
ip netns exec ns2 dhclient tap2

# 添加路由
# 参考原 eth0 的路由，为 br0 添加相同的路由，同时 eth0 的路由可以去掉
route -n
route add ... dev br0
```

注：上面的各种命令都不是唯一的，ip命令系、ifconfig命令系、相应的brctl、arp、route等命令都可以互换。

附：创建20个IP的脚本：

```bash
#!/bin/bash

set -x

# create Bridge
sudo brctl addbr br0 
sudo brctl stp br0 off 
sudo ip link set dev br0 up

for i in {0..19}
do
    # create veth peers
    sudo ip link add tap$i type veth peer name br-tap$i

    # create Network Namespaces
    sudo ip netns add ns$i

    # add one peer of veth into ns
    sudo ip link set tap$i netns ns$i

    # add one peer of veth into bridge
    sudo brctl addif br0 br-tap$i

    # open veth peer in ns
    sudo ip netns exec ns$i ip link set dev tap$i up

    # open veth peer in bridge
    sudo ip link set dev br-tap$i up
done

sudo ifconfig eth0 down
sudo service network-manager stop
sudo brctl addif br0 eth0
sudo ifconfig eth0 0.0.0.0 up

sudo dhclient br0 && sudo dhclient eth0
for i in {0..19}
do
    sudo ip netns exec ns$i dhclient tap$i
done
```

<br>

参考资料：

1. http://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/
2. http://www.opencloudblog.com/?p=66
3. https://blog.kghost.info/2013/03/01/linux-network-emulator/
4. http://man7.org/linux/man-pages/man8/ip-netns.8.html

