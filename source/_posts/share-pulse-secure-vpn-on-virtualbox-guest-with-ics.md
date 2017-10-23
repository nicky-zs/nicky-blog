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
2. 在VirtualBox中设置给Win7添加两块网卡，`网卡1`和`网卡2`，其中：
	- `网卡1`将用于访问外网，没有外网则无法建立VPN连接。这块网卡可以使用NAT、NAT Network、Bridged Adapter。
	- `网卡2`将用作共享上网的网关。这块网卡可以使用Host-Only Network、Bridged Adapter。
推荐将`网卡1`设置为Bridged Adapter或NAT（如果宿主机所在的网络环境无法进行网卡桥接或虚拟机不需要与宿主机同网段的IP），将`网卡2`设置为Host-Only Network：192.168.137.254/24，并开启DHCP，DHCP服务器192.168.137.100，DHCP地址池从192.168.137.101到192.168.137.199。
3. 在Win7上安装并配置好Pulse Secure客户端，并连接到VPN。连接到VPN之后，Win7会出现第3个网络适配器：`Pulse Secure`。
4. 在Pulse Secure上右键-->属性-->共享，勾选“允许其他网络用户通过此计算机的Internet连接来连接”，并在下面的网络连接列表中选择`网卡2`，然后确定。这时，`网卡2`的IP会从原来的使用DHCP自动获得变成一个固定IP：192.168.137.1/24（默认是这个IP，可以在注册表里修改）。
5. 这时，192.168.137.1已经成为了一个NAT网关可以使用了。若第2步中，`网卡2`使用Bridged Adapter模式，则在Linux上，将IP 192.168.137.2/24绑在网卡上，然后设置公司网段的路由10.0.0.0/8指向192.168.137.1，DNS服务器为192.168.137.1就OK了。若第2步中，`网卡2`使用的是推荐的Host-Only Network模式且使用推荐的IP配置，则宿主机上已经有一个IF绑定了192.168.137.254/24，直接修改路由增加10.0.0.0/8 via 192.168.137.1即可。

```bash
sudo ip a add 192.168.137.2/24 dev wlp3s0
sudo ip r add 10.0.0.0/8 via 192.168.137.1
echo 'nameserver 192.168.137.1' | sudo tee /etc/resolve.conf
```

有几个地方需要**注意**一下：
- 步骤4我原以为是一劳永逸的，后来发现每次在Win7上连接完VPN之后，都要重新把4中的设置取消了，然后再来一遍，不然就不生效。其实应该在每次结束VPN连接之前，先结束掉`Pulse Secure`网卡的共享。
- 步骤5中的`dev wlp3s0`是网卡设备名，不同的系统上不一样。如果使用推荐设置，则会有一个默认叫`vboxnet0`的设备已经绑好了合适的IP。
- 步骤5中路由的网段10.0.0.0/8根据不同的公司情况不一样，也有公司会用到192.168.0.0/16网段，要注意不要与自己家用路由器的路由产生冲突了。

<hr>

\# TODO     // 程序员最大谎言之一
1. 除了ICS，还可以在Win7上安装socks代理，让Linux通过Win7暴露的socks代理来访问VPN。通过shadowsocks-redir的方式，然后Linux上配iptables来实现。
2. 貌似有个叫redsocks的东西可以研究一下。
3. 有空再记录下通过iptables把Linux主机变成NAT网关的方法。
