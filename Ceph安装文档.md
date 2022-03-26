## 集群部署Ceph

### [系统要求](https://docs.ceph.com/en/latest/start/os-recommendations/#ceph-dependencies)

#### CEPH依赖项

一般来说，我们建议在较新的Linux版本上或者是长期支持的版本上部署Ceph。

##### Linux 内核

- Ceph Kernel Client

如果你要使用Kernel Client映射RBD块设备或挂载CephFS，一般建议是要么使用由http://kernel.org 网站提供的稳定的内核版本或者是长期维护的内核版本，要么使用Linux发型版本。

对于RBD，如果您选择跟踪长期内核，我们目前建议基于4.x的“长期维护”内核系列或更高版本：

- 4.19.z
- 4.14.z
- 5.x

> For CephFS, see the section about [Mounting CephFS using Kernel Driver](https://docs.ceph.com/en/latest/cephfs/mount-using-kernel-driver/#which-kernel-version) for kernel version guidance.

> Older kernel client versions may not support your CRUSH tunables profile or other newer features of the Ceph cluster, requiring the storage cluster to be configured with those features disabled.

#### [硬件要求](https://docs.ceph.com/en/latest/start/hardware-recommendations/)

> 待补充

#### 实验环境

| 主机名  |  系统 |  规格 |  IP地址 | 额外磁盘  |
| ------------ | ------------ | ------------ | ------------ | ------------ |
|  Ceph-01 |  CentOS 8 | 4GB内存 2核CPU  |  192.168.42.165 |   |
|  Ceph-02 |  CentOS 8 | 4GB内存 2核CPU  |  192.168.42.166 |   |
|  Ceph-03 |  CentOS 8 |  4GB内存 2核CPU | 192.168.42.167  |   |


#### 部署一个新的CEPH集群

在本环境中，我们将使用[cephadm](https://docs.ceph.com/en/latest/cephadm/#cephadm)来安装Ceph。

> 注意：Cephadm是Ceph版本v15.2.0 (Octopus）中的新功能，对于旧的版本并不支持。

#### 环境要求

- Python 3
- Systemd
- Podman or Docker for running containers
- NTP服务
- LVM2

#### 安装步骤

- 以root用户登录，分别修改3个节点的主机名，这里以第一个节点为例：

```
[root@localhost ~]# hostnamectl set-hostname ceph-01
[root@localhost ~]# hostnamectl 
   Static hostname: ceph-01
         Icon name: computer-vm
           Chassis: vm
        Machine ID: d346ae38f8ca4c10a839da76b79d5219
           Boot ID: 6d4ef4261295421184b88999c1bf8df0
    Virtualization: vmware
  Operating System: CentOS Stream 8
       CPE OS Name: cpe:/o:centos:centos:8
            Kernel: Linux 4.18.0-365.el8.x86_64
      Architecture: x86-64
```

- 在在各个节点的`/etc/hosts`文件添加主机名解析

```
tee -a /etc/hosts<<EOF
192.168.42.165    ceph-01  ceph-01.localdomain
192.168.42.166    ceph-02  ceph-02.localdomain
192.168.42.167    ceph-03  ceph-03.localdomain
EOF
```

- 检测各个节点是否网络互通，这里以其中一个节点为例：

```
[root@ceph-03 ~]# ping -c 2 ceph-01
PING ceph-01 (192.168.42.165) 56(84) bytes of data.
64 bytes from ceph-01 (192.168.42.165): icmp_seq=1 ttl=64 time=1.95 ms
64 bytes from ceph-01 (192.168.42.165): icmp_seq=2 ttl=64 time=1.64 ms

--- ceph-01 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.637/1.791/1.945/0.154 ms
[root@ceph-03 ~]# ping -c 2 ceph-02
PING ceph-02 (192.168.42.166) 56(84) bytes of data.
64 bytes from ceph-02 (192.168.42.166): icmp_seq=1 ttl=64 time=0.944 ms
64 bytes from ceph-02 (192.168.42.166): icmp_seq=2 ttl=64 time=1.51 ms

--- ceph-02 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.944/1.228/1.513/0.286 ms
[root@ceph-03 ~]# ping -c 2 ceph-03
PING ceph-03 (192.168.42.167) 56(84) bytes of data.
64 bytes from ceph-03 (192.168.42.167): icmp_seq=1 ttl=64 time=0.189 ms
64 bytes from ceph-03 (192.168.42.167): icmp_seq=2 ttl=64 time=0.606 ms

--- ceph-03 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1025ms
rtt min/avg/max/mdev = 0.189/0.397/0.606/0.209 ms
```

- 关闭防火墙
```
systemctl stop firewalld && systemctl disable firewalld
```

- 禁用selinux服务

编辑`/etc/selinux/config`文件
```

```

- [设置NTP服务](https://www.linuxprobe.com/centos7-chrony-time.html)

> 该步骤可参考以上链接


- 设置SSH无密码登录
 > 该步骤可参考 实例迁移模块-冷迁移-启用SSH免密码登录
 
 
 - 更新包
 ```
 yum update -y
 yum upgrade -y
 ```
 
 - 安装  python3、lvm2、docker
 
 ```
  yum install -y python3 lvm2 docker
 ```
 - 基于curl方法安装
 
 ```
 curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
 ```