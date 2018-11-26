# [VMware中CentOS设置静态IP](https://www.cnblogs.com/zhanjindong/p/3250393.html)

因为之前搭建的MongoDB分片没有采用副本集，最近现网压力较大，所以准备研究一下，于是在自己电脑的虚拟机中搭建环境，但是发现之前VMware设置的是DHCP，所以每次重新resume后虚拟机中IP都变了，导致之前已经搭建好的mongodb环境老是出问题又要重新搭建很麻烦，所以设置一下静态静态IP，步骤很简单：

**首先关闭VMware的DHCP**：

Edit->Virtual Network Editor

![](https://img.okay.do/e7693d18fb472678dbe0c2cbe3100a48_W604_H534_G0)

选择VMnet8，去掉Use local DHCP service to distribute IP address to VMs选项。点击NAT Settings查看一下GATEWAY地址：

![](https://img.okay.do/ef2b6ab30a9fb3d2c000eec1e268ed86_W507_H143_G0)

点击OK就可以了。

**设置CentOS静态IP：**

涉及到三个配置文件，分别是：

/etc/sysconfig/network /etc/sysconfig/network-scripts/ifcfg-eth0 /etc/resolv.conf

首先修改/etc/sysconfig/network如下：
```
NETWORKING=yes
HOSTNAME=localhost.localdomain  
GATEWAY=192.168.129.2
```
指定网关地址:==GATEWAY=192.168.129.2== 是必须的，和上方设置的Gateway IP 保持一致

然后修改/etc/sysconfig/network-scripts/ifcfg-eth0：
```
DEVICE="eth0"
#BOOTPROTO="dhcp"
BOOTPROTO="static"
IPADDR=192.168.129.129
NETMASK=255.255.255.0 
HWADDR="00:0C:29:56:8F:AD" 
IPV6INIT="no" 
NM_CONTROLLED="yes" 
ONBOOT="yes" 
TYPE="Ethernet"
UUID="ba48a4c0-f33d-4e05-98bd-248b01691c20" 
DSN1="8.8.8.8"
   ```

**注意**：这里DNS1是必须要设置的否则无法进行域名解析。
这样很简单几个步骤后虚拟机的IP就一直是192.168.129.129了。
# 克隆虚拟机的修改部分
1. 修改mac 地址 
	1. /etc/udev/rules.d/70-persistent-net.rules
2. 修改网卡信息 
	1. /etc/sysconfig/network-scripts/ifcfg-eth0
	2. 修改mac 地址
	3. 修改ip地址
3. 修改主机名
	1. /etc/sysconfig/network
4. 如果原机器没有非root用户
	1. 可以新建一个用户，设置密码
	2. 给予这个用户所有执行权限
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYyNTk1NzM3NywtNzkxNzcyNjA0LDI0NT
IwOTczMywtMjA4ODc0NjYxMl19
-->