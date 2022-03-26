## 集群部署Ceph


### [Ceph简介](https://docs.ceph.com/en/latest/start/intro/)

无论你希望向云平台提供 Ceph 对象存储或 Ceph 块设备服务, 部署 Ceph 文件系统或将 Ceph 用于其他用途，所有 Ceph 存储集群部署都从设置每个 Ceph 节点、网络和 Ceph 存储集群开始。Ceph 存储集群至少需要一个 Ceph Monitor 、 Ceph Manager 和 Ceph OSD (对象存储守护进程)。运行 Ceph 文件系统客户端时还需要 Ceph Metadata Server。


### [系统要求](https://docs.ceph.com/en/latest/start/os-recommendations/#ceph-dependencies)

> 提示：若要在生产环境部署Ceph，对于硬件以及操作系统有一定要求，具体详情可参考文档

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

### [硬件要求](https://docs.ceph.com/en/latest/start/hardware-recommendations/)

> 待补充

### 实验环境

| 主机名  |  系统 |  规格 |  IP地址 | 额外磁盘  |
| ------------ | ------------ | ------------ | ------------ | ------------ |
|  Ceph-01 |  CentOS 8 | 4GB内存 2核CPU  |  192.168.42.165 |   |
|  Ceph-02 |  CentOS 8 | 4GB内存 2核CPU  |  192.168.42.166 |   |
|  Ceph-03 |  CentOS 8 |  4GB内存 2核CPU | 192.168.42.167  |   |


#### 部署一个新的CEPH集群

在本环境中，我们将使用[cephadm](https://docs.ceph.com/en/latest/cephadm/#cephadm)来安装Ceph。

> 注意：Cephadm是Ceph版本v15.2.0 (Octopus）中的新功能，对于旧的版本并不支持。

#### 系统环境要求

- Python 3
- Systemd
- Podman or Docker for running containers
- NTP服务
- LVM2

#### 准备

> 注意：以下操作除了设置SSH无密码登录之外，其他操作均需要在所有节点上执行。

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
SELINUX=disabled
```

- [设置NTP服务](https://www.linuxprobe.com/centos7-chrony-time.html)

	> 该步骤可参考以上链接


- 设置SSH无密码登录
   设置ssh无密码登陆，需要需要在ceph-01上生成key，然后将公钥拷贝到其他节点(包括ceph-01节点)
 ```
 [root@ceph-01 ceph]# ssh-keygen
 [root@ceph-01 ceph]# ssh-copy-id -i /root/.ssh/id_rsa.pub ceph-01
 [root@ceph-01 ceph]# ssh-copy-id -i /root/.ssh/id_rsa.pub ceph-02
 [root@ceph-01 ceph]# ssh-copy-id -i /root/.ssh/id_rsa.pub ceph-03
 ```

- 更新包
```
yum update -y
yum upgrade -y
```
- 安装  python3、lvm2
 ```
  yum install -y python3 lvm2 # 默认系统已经安装
 ```
- 安装docker
 ```
 # sudo yum install -y yum-utils
 # sudo yum-config-manager  --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# sudo yum install docker-ce docker-ce-cli containerd.io --allowerasing
# systemctl enable docker
# systemctl status docker
 ```
 > 主要：我在Centos8 上安装docker出现冲突，因此增加了参数 --allowerasing。
 
 #### 使用CEPHADM 安装
 `cephadm`命令可以用来
 - 引导新集群
 - 使用有效的Ceph CLI启动容器化的Shell
 - aid in debugging containerized Ceph daemons（帮助调试容器化的Ceph守护进程。）

- 基于curl方法安装
 ```
[root@ceph-01 ceph]# curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
 ```
> 提示：由于环境原因，我未能成功下载脚本。于是我直接在浏览器中访问https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm 该脚本。把代码复制下来，在本地目录新建文件cephadm.py，将代码粘贴文本中。

- 添加执行权限
```
[root@ceph-01 ceph]# chmod +x cephadm.py 
```
- 添加仓库
```
[root@ceph-01 ceph]# ./cephadm add-repo --release pacific
```

- 执行脚本安装
```
./cephadm.sh install
```

- 检测是否安装成功
```
[root@ceph-01 ceph]# which cephadm
/usr/sbin/cephadm
```
> 提示：若显示以下内容，表明安装成功

#### 引导集群

创建新的 Ceph 集群的第一步是在 Ceph 群集的首台主机上运行 `cephadm bootstrap` 命令。在 Ceph 集群的第一台主机上运行 `cephadm bootstrap` 命令的行为会创建 Cep h 集群第一个“监控守护进程”，而该监控守护程序需要一个 IP 地址。你必须将 Ceph 集群的第一台主机的 IP 地址传递给 ceph bootstrap 命令，因此你需要知道该主机的 ip 地址。

```
cephadm bootstrap --mon-ip 192.168.42.165
```

此命令将会进行以下操作：
- 为本地主机上的新群集创建monitor和manager守护程序。
- 为 Ceph 群集生成新的 SSH 密钥，并将其添加到root用户的文件`/root/.ssh/authorized_keys`
- 将与新集群通信所需的最小配置文件保存到 `/etc/ceph/ceph.conf`
- 将`client.admin`管理（特权！）密钥的副本写入`/etc/ceph/ceph.client.admin.keyring`
将公钥的副本写入`/etc/ceph/ceph.pub`



![启动成功](/uploads/openstack/images/m_a6696a299a967445bcfa8cf2026decf7_r.png "启动成功")


![登录界面](/uploads/openstack/images/m_20ce9b89b2d4b05e94c2748464c947a1_r.png "登录界面")

> 提示：由于系统默认使用https，所以部分浏览器并不支持访问。

#### [启用`CEPH CLI`](https://docs.ceph.com/en/latest/cephadm/install/#enable-ceph-cli)

- 我们可以采用以下方式启用CEPH CLI

```
[root@ceph-01 ceph]# cephadm add-repo --release pacific
Writing repo to /etc/yum.repos.d/ceph.repo...
Enabling EPEL...
Completed adding repo.
[root@ceph-01 ceph]# cephadm install ceph-common
Installing packages ['ceph-common']...
```

- 确认`CEPH CLI`是否安装成功

```
[root@ceph-01 ceph]# ceph -v
ceph version 16.2.7 (dd0603118f56ab514f133c8d2e3adfc983942503) pacific (stable)
```

#### [添加节点](https://docs.ceph.com/en/latest/cephadm/host-management/#cephadm-adding-hosts)

所有节点必须满足以下要求，否则无法成功添加到集群中。

- Python 3
- Systemd
- Podman or Docker for running containers
- NTP服务
- LVM2

要将每个新的节点添加到集群中，需执行以下两个步骤：

- Install the cluster’s public SSH key in the new host’s root user’s authorized_keys file:

```
[root@ceph-01 ceph]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-02
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/etc/ceph/ceph.pub"

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@ceph-02'"
and check to make sure that only the key(s) you wanted were added.

[root@ceph-01 ceph]# ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-03
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/etc/ceph/ceph.pub"

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@ceph-03'"
and check to make sure that only the key(s) you wanted were added.

```

- 告诉Ceph，新节点是集群的一部分

```
[root@ceph-01 ceph]# ceph orch host add ceph-02 192.168.42.166
Added host 'ceph-02' with addr '192.168.42.166'
[root@ceph-01 ceph]# ceph orch host add ceph-03 192.168.42.167
Added host 'ceph-03' with addr '192.168.42.167'
```

- 列出与集群关联的节点

```
[root@ceph-01 ceph]# ceph orch host ls 
HOST     ADDR            LABELS  STATUS  
ceph-01  192.168.42.165  _admin          
ceph-02  192.168.42.166                  
ceph-03  192.168.42.167 
```

#### [添加额外的MONS](https://docs.ceph.com/en/latest/cephadm/services/mon/#deploy-additional-monitors)

一个典型的Ceph集群有三到五个的`monitor daemons`，它们分布位于不同的节点上。如果您的集群中有五个或更多节点，我们建议部署五个`monitor daemons`。

> 提示：该步骤可省略

#### [添加存储空间](https://docs.ceph.com/en/latest/cephadm/services/osd/#cephadm-deploy-osds)

`ceph-volume`会不时扫的描集群中的每个节点，以确定哪些设备存在，以及它们是否符合用作OSD的条件。

- 列出设备

```
[root@ceph-01 ceph]# ceph orch device ls 
HOST     PATH          TYPE  DEVICE ID                                   SIZE  AVAILABLE  REJECT REASONS  
ceph-01  /dev/nvme0n2  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes                        
ceph-01  /dev/nvme0n3  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes                        
ceph-02  /dev/nvme0n2  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes                        
ceph-02  /dev/nvme0n3  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes                        
ceph-03  /dev/nvme0n2  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes                        
ceph-03  /dev/nvme0n3  ssd   VMware Virtual NVMe Disk_VMware NVME_0000  10.7G  Yes     
```

如果满足以下所有条件，则存储设备被视为可用:

1. 设备必须没有分区
2.  设备不得具有任何 LVM 状态。
3.  设备不得挂载。
4.  设备不得包含文件系统。
5.  设备不得包含 Ceph BlueStore OSD。
6.  设备必须大于 5 GB。

- 添加新的OSDS

可以使用以下方式来创建新的OSDS

1. 告诉Ceph使用任何可用和未使用的存储设备：
```
ceph orch apply osd --all-available-devices
```

2. 从特定节点上的指定设备创建OSD：
```
ceph orch daemon add osd *<host>*:*<device-path>*
```

	如：`ceph orch daemon add osd host1:/dev/sdb`
	
- DRY RUN

The --dry-run flag causes the orchestrator to present a preview of what will happen without actually creating the OSDs.

```
ceph orch apply osd --all-available-devices --dry-run
```

#### 服务管理

- 列出服务列表

```
[root@ceph-01 ~]# ceph orch ls
NAME                       PORTS        RUNNING  REFRESHED  AGE  PLACEMENT                        
alertmanager               ?:9093,9094      1/1  4m ago     6d   count:1                          
crash                                       3/3  4m ago     6d   *                                
grafana                    ?:3000           1/1  4m ago     6d   count:1                          
mds.cephfs                                  3/3  4m ago     2d   ceph-01;ceph-02;ceph-03;count:3  
mgr                                         2/2  4m ago     6d   count:2                          
mon                                         3/5  4m ago     6d   count:5                          
node-exporter              ?:9100           3/3  4m ago     6d   *                                
osd.all-available-devices                     6  4m ago     6d   *                                
prometheus                 ?:9095           1/1  4m ago     6d   count:1                          
rgw.myorg                  ?:80             3/3  4m ago     18h  ceph-01;ceph-02;ceph-03;count:3  
```


