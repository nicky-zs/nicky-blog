---
title: 通过ICS分享VirtualBox虚拟机上的Pulse Secure VPN网络
tags:
  - ICS
  - NAT
  - VirtualBox
  - Pulse Secure
  - VPN
date: 2017-08-26 09:01:43
---


我们公司的VPN用的是Pulse Secure + MOTP（一种动态口令验证）。目前只有在Windows系统和MacOS系统中才能连接成功；在Linux系统上，由于MOTP的存在，Pulse Secure总是连接不上。还好，平时为了使用只有Windows下才有的一些软件，我的Linux上一直都有VirtualBox的Win7虚拟机。折腾好久之后，最后终于摸索出一种基于VirtualBox的Win7虚拟机连接的VPN的方法：通过VirtualBox Win7虚拟机来进行Pulse Secure VPN的连接，再通过ICS将VPN连接共享给Linux系统使用。

<!-- more -->

步骤记录如下：

1. 在Linux上安装VirtualBox，并在VirtualBox中安装一个Windows系统（在我的系统上，这个VirtualBox Guest是32位的Win7，以下简称Win7）。
2. 在VirtualBox中设置Win7通过Host的网卡桥接上网，并且要为Win7虚拟出两个网卡NIC1和NIC2，一块用于ICS共享，一块用于Win7连接外网。
3. 在Win7上安装并配置好Pulse Secure客户端，并连接到VPN。连接到VPN之后，Win7会出现3个网络适配器：本地连接1（NIC1）、本地连接2（NIC2）、Pulse Secure。
4. 在Pulse Secure上右键-->属性-->共享，勾选“允许其他网络用户通过此计算机的Internet连接来连接”，并在下面的网络连接列表中选择“本地连接1”（可以改名叫“VPN共享连接”便于记忆），然后确定。这时，本地连接1的IP会从原来的使用DHCP自动获得变成一个固定IP：192.168.137.1/24（默认是这个IP，可以在注册表里修改）。
5. 这时，192.168.137.1已经成为了一个NAT网关可以使用了。在Linux上，将IP 192.168.137.2/24绑在网卡上，然后设置公司网段的路由指向192.168.137.2，DNS服务器为192.168.137.2就OK了。
```bash
sudo ip a add 192.168.137.2/24 dev wlp3s0
sudo ip r add 10.0.0.0/8 via 192.168.137.1
echo 'nameserver 192.168.137.1' | sudo tee /etc/resolve.conf
```

有几个地方需要注意一下：
- 步骤2给Win7两块网卡是必须的。因为本地连接1被拿来共享VPN连接之后，它会获得一个静态且没有网关的IP，没有本地连接2整个Win7就无法访问外网，VPN也会失效。
- 步骤4我原以为是一劳永逸的，后来发现每次在Win7上连接完VPN之后，都要重新把4中的设置取消了，然后再来一遍，不然就不生效。
- 步骤5中的`dev wlp3s0`是网卡设备名，不同的系统上不一样。
- 步骤5中路由的网段10.0.0.0/8根据不同的公司情况不一样，也有公司会用到192.168.0.0/16网段，要注意不要与自己家用路由器的路由产生冲突了。

<hr>

\# TODO     // 程序员最大谎言之一
1. 除了ICS，还可以在Win7上安装socks代理，让Linux通过Win7暴露的socks代理来访问VPN。通过shadowsocks-redir的方式，然后Linux上配iptables来实现。
2. 貌似有个叫redsocks的东西可以研究一下。
3. 有空再记录下通过iptables把Linux主机变成NAT网关的方法。
