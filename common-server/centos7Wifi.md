# Centos7 最小安装 ， 配置无线网络

#### 方法1：使用NetworkManager自带的nmcli命令

```bash
// 查看无线网卡是否已经成功驱动
nmcli
// 我的无线网卡显示的是wlp3s0，表示已经成功驱动，如果看不到无线网卡名称，利用lspci（需要安装pciutils包）命令查
// 看自己的网卡型号，下载相应的驱动程序进行安装
// 配置无线网卡
nmcli dev wifi con “无线网络名称” password “无线网络密码” name “任意连接名称（删除，修改时用）”
// 利用nmcli查看连接信息，能看到IP地址表示连接成功
nmcli
// 删除此次连接，可以使用如下命令
nmcli c del “连接名称”
```

#### 方法2：使用wpa\_supplicant

因为NetworkManager底层也是使用的wpa\_supplicant，同时使用会发生冲突，所以在使用wpa\_supplicant之前，需要关闭NetworkManager服务

```
// 关闭NetworkManager服务
systemctl stop NetworkManager
// 配置无线网卡
wpa_supplicant -B -i wlp3s0 -c < (wpa_passphrase “网络名称” “网络密码”)
// 自动获取IP地址
dhclient wlp3s0
// 查看是否成功获取ip地址，如果使用ifconfig命令需要安装net-tools
ip addr show wlp3s0
// 删除此次连接，可以使用如下命令
dhclient -r
```



#### 方法3：手动建立无线网络配置文件

在/etc/sysconfig/network-scripts/目录下新建ifcfg-wlp3s0配置文件和keys-wlp3s0密码文件

