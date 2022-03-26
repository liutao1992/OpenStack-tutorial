## 基础环境
本环境搭建主要基于学习的目的，若是要部署正式环境，请参考[文档](https://docs.openstack.org/xena/deploy/)

### OpenStack 版本

本环境是基于`Train`版本进行搭建。

### 运行系统：CentOS7

> 注意：从Ussuri版本开始，您需要使用CentOS8或RHEL 8。以前的OpenStack版本需要使用CentOS7或RHEL 7。不同发行版和版本都包含说明。

### 硬件要求

为了获得最佳性能，我们建议您的环境需满足或超过以下[硬件](https://docs.openstack.org/install-guide/overview.html#figure-hwreqs)要求。

The following minimum requirements should support a proof-of-concept environment with core services and several CirrOS instances:


- 控制器节点：1个处理器、4 GB内存和5 GB存储

- 计算节点1：1个处理器、2 GB内存和10 GB存储

- 计算节点2：1个处理器、2 GB内存和10 GB存储

随着OpenStack服务和虚拟机数量的增加，对最佳性能的硬件要求也随之增加。如果启用其他服务或虚拟机后性能下降，请考虑向您的环境添加硬件资源。


> 注意：If you choose to install on VMs, make sure your hypervisor provides a way to disable MAC address filtering on the provider network interface.

###  系统服务设置
####  禁用 NetworkManager 服务
```
systemctl disable NetworkManager && systemctl stop NetworkManager

```
> 因为openstack的neutron组件会和networkmanager冲突

####  禁用 firewalld

```
systemctl disable firewalld && systemctl stop firewalld
```

####  启用 network

```
systemctl enable network && systemctl restart network
```

#### 禁用 selinux 服务自启，并重启虚拟机使之生效

- 编辑`/etc/selinux/config`文件，将其设置为`SELINUX=disabled`
- 重启系统，通过以下命令检查是否修改成功: 
  ```
  [root@controller]# getenforce
  Disabled
  ```


## 主机网络 [Host networking](https://docs.openstack.org/install-guide/environment-networking.html)


在各个节点进行以下配置

### 设置网卡，将其配置静态IP，设置完成后重启网络

> 这里以控制节点为例

- 使用vim编辑 `/etc/sysconfig/network-scripts/ifcfg-ens33`文件

```
DEVICE="ens33"
TYPE=Ethernet
ONBOOT="yes"         # 开机启动
BOOTPROTO="none"     # 将其IP设置静态ip

# 新增内容
IPADDR=192.168.42.150
NETMASK=255.255.255.0
GATEWAY=192.168.42.2
DNS=114.114.114.114
```

> 提示：其他内容不做改动

- 重启服务：systemctl restart network


### 设置各个节点主机名

- 控制节点

`hostnamectl set-hostname controller`

- 计算节点1

`hostnamectl set-hostname compute1`

- 计算节点2

`hostnamectl set-hostname compute2`


### 编辑hosts域名解析文件

> 这里以控制节点为例，其他节点内容一样。

编辑控制节点的`/etc/hosts`文件

```
tee -a /etc/hosts<<EOF
192.168.42.169    controller  controller.localdomain
192.168.42.154    compute1    compute1.localdomain
192.168.42.155    compute2    compute2.localdomain
EOF
```

### 检查网络连接

在进行下一步之前，检查网络和各节点之间的网络连接。

> 这里我们以控制节点为例，其他节点操作类似。

#### 控制节点

- 检查该节点是否可以访问互联网

```
# ping -c 4 docs.openstack.org
PING files02.openstack.org (23.253.125.17) 56(84) bytes of data.
64 bytes from files02.openstack.org (23.253.125.17): icmp_seq=1 ttl=43 time=125 ms
64 bytes from files02.openstack.org (23.253.125.17): icmp_seq=2 ttl=43 time=125 ms
64 bytes from files02.openstack.org (23.253.125.17): icmp_seq=3 ttl=43 time=125 ms
64 bytes from files02.openstack.org (23.253.125.17): icmp_seq=4 ttl=43 time=125 ms


--- files02.openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 125.192/125.282/125.399/0.441 ms
```

- 检查控制节点是否可以访问其他计算节点

```
# ping -c 4 compute1

PING compute1 (10.0.0.31) 56(84) bytes of data.
64 bytes from compute1 (10.0.0.31): icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=2 ttl=64 time=0.202 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=3 ttl=64 time=0.203 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=4 ttl=64 time=0.202 ms

--- compute1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.202/0.217/0.263/0.030 ms
```

```
# ping -c 4 compute2

PING compute1 (10.0.0.31) 56(84) bytes of data.
64 bytes from compute1 (10.0.0.31): icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=2 ttl=64 time=0.202 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=3 ttl=64 time=0.203 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=4 ttl=64 time=0.202 ms

--- compute1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.202/0.217/0.263/0.030 ms
```

### 安装[NTP服务](https://docs.openstack.org/install-guide/environment-ntp.html)

在各个节点安装`NTP`服务

> 这里以控制节点为例
#### 安装  `yum install chronyc -y`

#### 使用vim编辑 /etc/chrony.conf

```
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst

server ntp.aliyun.com iburst

allow 192.168.42.0/24       # 使其他节点能够连接到该控制节点上chrony服务
```

> 提示：除了控制节点，其他节点可省略该配置 allow 192.168.42.0/24  

#### 启动服务
```
systemctl enable chronyd
systemctl restart chronyd
```

#### 检查NTP服务

在进行下一步操作之前，我们先检查NTP服务是否如我们预期一样可以正常工作。

##### 在控制节点执行以下指令

```
[root@controller ~]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^? controller                    0   9     0     -     +0ns[   +0ns] +/-    0ns
```

##### 在其他节点运行同样的命令

```
[root@compute-1 liutao]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^? controller               0   9     0     -     +0ns[   +0ns] +/-    0ns
```

> 注意：若是系统当前时区不对，我们可以通过以下指令进行设置 `timedatectl set-timezone Asia/Shanghai`

##### 查看时间同步状态

```
[root@controller ~]# timedatectl
      Local time: Wed 2022-02-23 10:42:51 CST
  Universal time: Wed 2022-02-23 02:42:51 UTC
        RTC time: Wed 2022-02-23 02:42:51
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

- NTP enabled：需要确保状态为yes
- NTP synchronized：需要确保状态为yes
- Time zone：需要确保时间为Asia/Shanghai (CST, +0800)



### [安装源](https://docs.openstack.org/install-guide/environment-packages.html)
在各个节点安装源。CentOS默认YUM源使用的是国外的，我们需要将源替换为国内的源，这样可以提高YUM安装RPM包的速度。

> 注意这里我只以控制节点为例，其他节点也一样。


#### 先下载 OpenStack Train 版本的库：

`yum install centos-release-openstack-train -y`

#### 备份

备份 repo 源，保留 CentOS-Base.repo 和 CentOS-Media.repo，再新建阿里源。

```
[root@controller ]# cd /etc/yum.repos.d/
[root@controller yum.repos.d]# ls
CentOS-Base.repo           CentOS-Debuginfo.repo  CentOS-OpenStack-train.repo  CentOS-Storage-common.repo
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo  CentOS-QEMU-EV.repo           CentOS-Vault.repo
CentOS-CR.repo             CentOS-Media.repo      CentOS-Sources.repo           CentOS-x86_64-kernel.repo
[root@controller yum.repos.d]# mkdir ./repo_backup
[root@controller yum.repos.d]# mv *.repo ./repo_backup/
[root@controller yum.repos.d]# ls
repo_backup
[root@controller yum.repos.d]# cd ./repo_backup/
[root@controller repo_backup]# ls
CentOS-Base.repo           CentOS-Debuginfo.repo  CentOS-OpenStack-queens.repo  CentOS-Storage-common.repo
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo  CentOS-QEMU-EV.repo           CentOS-Vault.repo
CentOS-CR.repo             CentOS-Media.repo      CentOS-Sources.repo           CentOS-x86_64-kernel.repo
[root@controller repo_backup]# mv CentOS-Base.repo CentOS-Media.repo ..
[root@controller repo_backup]# ls
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo         CentOS-Sources.repo         CentOS-x86_64-kernel.repo
CentOS-CR.repo             CentOS-OpenStack-train.repo  CentOS-Storage-common.repo
CentOS-Debuginfo.repo      CentOS-QEMU-EV.repo           CentOS-Vault.repo
[root@controller repo_backup]# cd ..
[root@controller yum.repos.d]# ls
CentOS-Base.repo  CentOS-Media.repo  repo_backup
```

#### 新建源文件

`vim /etc/yum.repos.d/CentOS-OpenStack-train.repo`

内容如下：

```
[centotack-train]
name=openstack-train
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-train/
enabled=1
gpgcheck=0
[qume-kvm]
name=qemu-kvm
baseurl= https://mirrors.aliyun.com/centos/7/virt/x86_64/kvm-common/
enabled=1
gpgcheck=0
```
#### 更新包

升级所有节点上的软件包：`yum upgrade -y`

> 如果更新的过程中含有内核，那么则需要重启系统来激活它。

#### 安装OpenStack客户端工具

`yum install python-openstackclient`

#### OpenStack SELinux管理工具

安装openstack-selinux让其自动管理OpenStack服务的安全策略:

`yum -y install openstack-selinux`