### 创建虚拟机CentOS7
> 配置如下：内存16G，CPU12核，硬盘250G，NAT连接
> 注意：CPU要开启硬件虚拟化


### 更改主机名 node-a
- 执行该命令 `hostnamectl set-hostname node-a`
- 执行指令查看是更改成功 `hostname -f`

### 禁用服务 NetworkManager
```
systemctl disable NetworkManager
systemctl stop NetworkManager

```
> 因为openstack的neutron组件会和networkmanager冲突

### 禁用 firewalld

```
systemctl disable firewalld
systemctl stop firewalld
```

### 设置网卡，配置静态IP，设置完成后重启网络

- 使用vim编辑 `/etc/sysconfig/network-scripts/ifcfg-ens33`文件
- 内容如下：

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static" # 更改项
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="10c5a3d4-e123-490c-8fd6-864426f6c110"
DEVICE="ens33"
ONBOOT="yes"

# 添加项
IPADDR=192.168.42.150
NETMASK=255.255.255.0
GATEWAY=192.168.42.2
DNS=114.114.114.114                
```

- 执行指令 `systemctl enable network`
- 执行指令 `systemctl restart network`
- 检测网络 `ping www.baidu.com`

### 编辑hosts域名解析文件

使用vim编辑 `/etc/hosts` 

内容如下：

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.42.150 node-a node-a.localdomain
```

### 禁用 selinux 服务自启，并重启虚拟机使之生效

- Configure SELINUX=disabled in the /etc/selinux/config file
- Reboot your system. After reboot, confirm that the getenforce command returns Disabled: 
  ```
  [root@node-a l]# getenforce
  Disabled
  ```

### NTP 配置

配置文件中注释掉其他 server，添加 ESXi 主机作为 NTP server，然后重启 服务，开启并测试  

- 若没安装时间同步软件chronyc时，则执行该指令进行安装 `yum install chronyc -y`
- 使用 vim 编辑 `/etc/chrony.conf`

 编辑内容如下：
```
# server 0.centos.pool.ntp.org iburst
# server 1.centos.pool.ntp.org iburst
# server 2.centos.pool.ntp.org iburst
# server 3.centos.pool.ntp.org iburst
server 192.168.42.150 iburst   # 当前主机ip作为NTP服务
```

> 注释所有server，然后添加`server 192.168.42.150 iburst`. 若是以后部署多节点，则所有节点以当前IP的虚拟时钟进行同步

- 执行指令 `systemctl enable chronyd`
- 执行指令 `systemctl restart chronyd`
- 执行指令 `chronyc sources`

```
[root@node-a l]# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^? node-a                        0   7     0     -     +0ns[   +0ns] +/-    0ns
[root@node-a l]# 
```

- 执行指令 `chronyc sourcestats`
```
[root@node-a liutao]# chronyc sourcestats
210 Number of sources = 1
Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
node-a                      0   0     0     +0.000   2000.000     +0ns  4000ms
[root@node-a liutao]# 
```

- 设置时区 `timedatectl set-timezone Asia/Shanghai`
- 读取当前时区，查看是否配置对 `timedatectl status`

### 更改语言编码
- 创建这个配置文件 `vim /etc/environment`
- 内容如下：
```
LANG=en_US.utf-8
LC_ALL=en_US.utf-8     
```

### 安装 openstack 的库配置
- 先下载 OpenStack Queens 版本的库：`yum install centos-release-openstack-queens -y`
- 备份 repo 源，保留 CentOS-Base.repo 和 CentOS-Media.repo，再新建阿里源。

```
[root@node-a l]# cd /etc/yum.repos.d/
[root@node-a yum.repos.d]# ls
CentOS-Base.repo           CentOS-Debuginfo.repo  CentOS-OpenStack-queens.repo  CentOS-Storage-common.repo
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo  CentOS-QEMU-EV.repo           CentOS-Vault.repo
CentOS-CR.repo             CentOS-Media.repo      CentOS-Sources.repo           CentOS-x86_64-kernel.repo
[root@node-a yum.repos.d]# mkdir ./repo_backup
[root@node-a yum.repos.d]# mv *.repo ./repo_backup/
[root@node-a yum.repos.d]# ls
repo_backup
[root@node-a yum.repos.d]# cd ./repo_backup/
[root@node-a repo_backup]# ls
CentOS-Base.repo           CentOS-Debuginfo.repo  CentOS-OpenStack-queens.repo  CentOS-Storage-common.repo
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo  CentOS-QEMU-EV.repo           CentOS-Vault.repo
CentOS-CR.repo             CentOS-Media.repo      CentOS-Sources.repo           CentOS-x86_64-kernel.repo
[root@node-a repo_backup]# mv CentOS-Base.repo CentOS-Media.repo ..
[root@node-a repo_backup]# ls
CentOS-Ceph-Luminous.repo  CentOS-fasttrack.repo         CentOS-Sources.repo         CentOS-x86_64-kernel.repo
CentOS-CR.repo             CentOS-OpenStack-queens.repo  CentOS-Storage-common.repo
CentOS-Debuginfo.repo      CentOS-QEMU-EV.repo           CentOS-Vault.repo
[root@node-a repo_backup]# cd ..
[root@node-a yum.repos.d]# ls
CentOS-Base.repo  CentOS-Media.repo  repo_backup
```

-  使用vim新建文件`vim /etc/yum.repos.d/CentOS-OpenStack-queens.repo`, 内容如下：

```
[centotack-queens]
name=openstack-queens
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-queens/
enabled=1
gpgcheck=0
[qume-kvm]
name=qemu-kvm
baseurl= https://mirrors.aliyun.com/centos/7/virt/x86_64/kvm-common/
enabled=1
gpgcheck=0
```
- 执行指令`yum upgrade -y`
- 执行指令 `yum install openstack-packstack -y`
- 执行指令 `packstack --gen-answer-file=answer --default-password=000000`
- 执行指令 `packstack --answer-file=answer`

### 查看OpenStack核心组建nova的版本 `nova-manage version`

[参考链接](https://www.alibabacloud.com/blog/how-to-install-single-node-openstack-on-centos-7_594048?spm=a2c41.12117054.0.0)



  



  





