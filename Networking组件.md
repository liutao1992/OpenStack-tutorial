### [neutron组件](https://docs.openstack.org/neutron/train/install/)

本章将介绍如何使用[provider networks](https://docs.openstack.org/neutron/train/install/overview.html#network1)或[self-service networks](https://docs.openstack.org/neutron/train/install/overview.html#network2)安装和配置网络服务。

OpenStack项目中的虚拟网络分为两种类型：provider networks 和 self-service networks。

provider networks 由管理员创建，并直接映射到现有的一个物理网络上，可以在多个project之间共享。它可以为虚拟机实例提供基于二层桥接和交换网络的虚拟网络，虚拟网络中的DHCP为实例提供ip地址。每个物理网络最多只能实现一个虚拟网络。




> 提示：在安装neutron组件时，我选择了provider networks进行网络配置。

### 控制节点

#### 数据库配置

- 创建数据库

```
CREATE DATABASE neutron;
```

- 权限设置

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'NEUTRON_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
```

#### 创建服务凭证

-  创建neutron用户:

```
[root@controller conf.d]# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | cf32565e651c4fe5b5f5742274ce7b84 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 向neutron用户添加admin角色：

```
[root@controller conf.d]# openstack role add --project service --user neutron admin
```

- 创建neutron服务

```
[root@controller conf.d]# openstack service create --name neutron \
>   --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | abf0d5cd3d7b40ad9beb2d66724e67a1 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

#### 创建 Networking服务 API endpoints:

```
[root@controller conf.d]# openstack endpoint create --region RegionOne \
>   network public http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | baef213dca5547b6bc6309fcf6ea4291 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | abf0d5cd3d7b40ad9beb2d66724e67a1 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

```
[root@controller conf.d]# openstack endpoint create --region RegionOne \
>   network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3f9cf331d5634726a9aae91166e67b32 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | abf0d5cd3d7b40ad9beb2d66724e67a1 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

```
[root@controller conf.d]# openstack endpoint create --region RegionOne \
>   network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fc2cf0aff001468e85c3ee9f1bf8b297 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | abf0d5cd3d7b40ad9beb2d66724e67a1 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

#### 配置网路选项

你可以使用选项1、选项2表示的两个体系架构之一的一种来部署网络服务。

选项1部署了最简单的架构，该架构仅支持将实例附加到提供商（外部）网络。没有自助（专用）网络、路由器或浮动IP地址。只有admin或其他特权用户才能管理 provider networks。

#### Networking Option 1: Provider networks

- 在控制节点上安装和配置网络组件。

```
# yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

- 设置服务组件

编辑/etc/neutron/neutron.conf 文件，并添加以下内容：

```
[database]
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[DEFAULT]
core_plugin = ml2
service_plugins =

transport_url = rabbit://openstack:RABBIT_PASS@controller

auth_strategy = keystone

notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS

[oslo_concurrency]

lock_path = /var/lib/neutron/tmp
```

#### 设置ML2模块插件

编辑/etc/neutron/plugins/ml2/ml2_conf.ini文件，并添加以下内容：

```
[ml2]
type_drivers = flat,vlan

tenant_network_types =

mechanism_drivers = linuxbridge

extension_drivers = port_security

[ml2_type_flat]

flat_networks = provider

[securitygroup]
enable_ipset = true
```

#### 设置Linux bridge 代理

编辑/etc/neutron/plugins/ml2/linuxbridge_agent.ini文件，添加以下内容：

```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME 

[vxlan]

enable_vxlan = false

[securitygroup]

enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

> 提示：该值 PROVIDER_INTERFACE_NAME 换成我们的物理网卡。比如我当前的物理网卡为：ens33。因此为 physical_interface_mappings = provider:ens33

要启用networking bridge支持， 在CentOS上还需要加载br_netfilter模块，此模块默认没有加载，需要执行下面的命令手动加载。

```
[root@controller ~]# modprobe br_netfilter
```

加载完成后可以使用下面的命令确认模块是否加载成功

```
[root@controller ~]# lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter
```

确保2层网络报文过滤已经开启，下面两个内核参加需要值为1

```
[root@controller ~]# cat /proc/sys/net/bridge/bridge-nf-call-iptables 
1
[root@controller ~]# cat /proc/sys/net/bridge/bridge-nf-call-ip6tables 
1
```

#### 设置DHCP代理

编辑/etc/neutron/dhcp_agent.ini文件，然后添加以下内容：

```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

#### 设置metadata代理

编辑/etc/neutron/metadata_agent.ini文件，然后添加以下内容：

```
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

> METADATA_SECRET 替换为合适的秘钥，后面会用到。

#### 设置Compute服务使用Networking服务

编辑/etc/nova/nova.conf文件，然后添加以下内容：

```
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

#### 安装完成

- 设置软连接

```
[root@controller ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

- 初始化数据库

```
[root@controller ~]# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

- 重启 Compute API 服务:

```
[root@controller ~]# systemctl restart openstack-nova-api.service
```

- 启动网络服务

```
[root@controller ~]# systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service

[root@controller ~]# systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```



### 计算节点

#### 安装包

```
[root@compute1 ~]# yum install openstack-neutron-linuxbridge ebtables ipset
```

#### 设置该组件配置文件

编辑/etc/neutron/neutron.conf 文件，并添加以下内容：

```
[DEFAULT]
transport_url = rabbit://openstack:RABBIT_PASS@controller

auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

```

#### 配置网络选项

编辑/etc/neutron/plugins/ml2/linuxbridge_agent.ini，新增内容如下：

```
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

```

> PROVIDER_INTERFACE_NAME:换成我们的物理网卡的值

- 要启用networking bridge支持， 在CentOS上还需要加载br_netfilter模块，此模块默认没有加载，需要执行下面的命令手动加载。

```
[root@compute1 ~]# modprobe br_netfilter
```

加载完成后可以使用下面的命令确认模块是否加载成功

```
[root@compute1 ~]# lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter
```

确保2层网络报文过滤已经开启，下面两个内核参加需要值为1

```
[root@compute1 ~]# cat /proc/sys/net/bridge/bridge-nf-call-iptables 
1
[root@compute1 ~]# cat /proc/sys/net/bridge/bridge-nf-call-ip6tables 
1
```

#### 设置Compute服务使用Networking服务

编辑/etc/nova/nova.conf文件，然后添加以下内容：

```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

#### 重启服务

```
[root@compute1 ~]#  systemctl restart openstack-nova-compute.service
```

#### 启动桥接代理服务

```
[root@compute1 ~]# systemctl enable neutron-linuxbridge-agent.service

[root@compute1 ~]# systemctl start neutron-linuxbridge-agent.service
```

### 检查服务是否正常运行

```
[root@controller ~]# openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 3cd5300c-6ec2-4e52-aa67-b1d8b42f411b | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 5faf270f-fa81-4720-9855-2ace0ebfd132 | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 6a926a47-704a-454b-835f-35ed543ecf67 | Linux bridge agent | compute2   | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 92686d0a-db40-46a6-b3a3-f53014d535d4 | Linux bridge agent | compute1   | None              | :-)   | UP    | neutron-linuxbridge-agent |
| fa5322aa-d5ef-4e17-9b89-1b6b78e6e053 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

> 提示：Alive是 :-)，State是UP时表明服务正常，反之，则需要认真检查配置文件