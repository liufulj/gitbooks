# CentOS 7防火墙设置开放80端口

在CentOS 6.x版本中，默认使用的是`iptables`防火墙。到了CentOS 7.x版本，默认防火墙变成了`firewalld`。本篇通过使用`firewalld`开启、关闭 HTTP（80）端口，来讲述`firewalld`的基本使用方法。

`firewalld` 的一切设置均使用 `firewall-cmd` 命令完成。

配置前先确保防火墙是运行着的：

```
firewall-cmd --state
```

输出**running**就说明运行着，否则需要开启：

```
service firewalld startRedirecting to/bin/systemctlstart  firewalld.service
```

服务器上可能会有多张网卡，每张网卡可能有多个网口。`firewalld` 最细可以控制每个网口的进出流量。所以配置前需要知道要控制的网口的名字，用`ifconfig`命令获取：

```
ifconfig
```

```
eno16777736: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.28.128  netmask 255.255.255.0  broadcast 192.168.28.255
        inet6 fe80::20c:29ff:fef4:6dd1  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:f4:6d:d1  txqueuelen 1000  (Ethernet)
        RX packets 13558  bytes 16041550 (15.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4380  bytes 315435 (308.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 72  bytes 6140 (5.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 72  bytes 6140 (5.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

一般买来的云服务器，只有一张网卡一个网口，这种情况下ifconfig会列出两个网口，比如这里是eno16777736和lo。lo是本地回路，是用于调试的，不是真正的网口。剩下的eno16777736就是真实网口的名字。如果发现机器上除了lo网口，还是有多个网口，说明服务器上有多张网卡或多个网口。这时候要自己判断开操作哪个网口。知道了要操作哪个网口。还需要了解下下firewalld中zone的概念：firewalld将服务器网络环境划分为几个zone。就如同美国划分了很多个州，各个州都有各自的法律，一个生活在美国的人必须处在某一个洲（比如Ohio洲），行为受到该洲的限制，如果把此人从Ohio洲移动到Texas洲，那么他收到的法律限制就会发生变化。  
同样的道理，一个网口必须处在某一个zone之内，zone有一套流量进出的规则，网口的进出流量就得遵循这套规则。如果把网口从一个zone移动到另一个zone后，该网口的流量进出规则就会改变。根据这样的解释可以知道，防火墙的流量规则都是配置在zone上的，而不是直接配置在网口上的。所以先给public这个zone添加规则：

允许80端口的流量通过：

```
firewall-cmd --zone=public --add-port=80/tcp
```

返回success即代表成功。然后把网口`eno16777736`添加到`public`这个zone里面：

```
firewall-cmd --zone=public --add-interface=eno16777736
```

最后用浏览器访问服务器，可以发现就能正常访问HTML内容了。80端口成功开启！

