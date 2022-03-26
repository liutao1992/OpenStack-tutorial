### [Nova组件](https://docs.openstack.org/nova/train/install/controller-install-rdo.html)
nova是openstack最核心的服务之一，负责维护和管理云环境的计算资源。通过nova来实现虚拟机生命周期管理。OpenStack计算服务需要与其他服务进行交互，如KeyStone服务用于认证，Glance服务提供磁盘和服务器镜像，Dashboard提供用户与管理员接口。

本节介绍如何在控制节点、计算节点上安装和配置nova服务。

### 控制节点

#### 数据库配置

- 新建数据库

```
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
```

- 权限设置

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
```

#### 创建Compute服务凭证

- 创建nova用户

```
[root@controller ~]# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c3b9f794f3c7420192ccd63c4e844f9e |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 将管理员（admin）角色授予nova用户和service项目

```
openstack role add --project service --user nova admin
```

-  创建nova服务

```
[root@controller ~]# openstack service create --name nova \
>   --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 5dc83d1b1b5040219d2632b64a261048 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

#### 创建Compute API 服务 endpoints

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   compute public http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4c0405b69b674ac49be357f3cd3a7f13 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5dc83d1b1b5040219d2632b64a261048 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   compute internal http://controller:8774/v2.1

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0b84178d7f4f433ebb1c6af716ea16cd |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5dc83d1b1b5040219d2632b64a261048 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   compute admin http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 61b2e2725eef43e9b81dae4888530c66 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5dc83d1b1b5040219d2632b64a261048 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

#### 安装配置nova组件

- 安装包

```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-novncproxy openstack-nova-scheduler
```

- 编辑/etc/nova/nova.conf文件并添加以下内容：

```
[DEFAULT]

enabled_apis = osapi_compute,metadata

transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/

my_ip = 192.168.42.169    # 控制节点的IP地址
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS


[vnc]
enabled = true
server_listen = 192.168.42.169                 # 控制节点的IP地址
server_proxyclient_address = 192.168.42.169    # 控制节点的IP地址

[glance]

api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]

region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

#### 初始化数据库

```
# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
# su -s /bin/sh -c "nova-manage db sync" nova
```

验证nova cell0 和 cell1 是否正确注册

```
# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
|  Name |                 UUID                 |                   Transport URL                    |                     Database Connection                      | Disabled |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                       none:/                       | mysql+pymysql://nova:****@controller/nova_cell0?charset=utf8 |  False   |
| cell1 | f690f4fd-2bc5-4f15-8145-db561a7b9d3d | rabbit://openstack:****@controller:5672/nova_cell1 | mysql+pymysql://nova:****@controller/nova_cell1?charset=utf8 |  False   |
+-------+--------------------------------------+----------------------------------------------------+--------------------------------------------------------------+----------+
```

#### 启动服务

```
# systemctl enable \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
# systemctl start \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```

### 计算节点

#### 安装、配置组件

- 安装包

```
yum install openstack-nova-compute
```

- 编辑/etc/nova/nova.conf文件并添加以下内容

```
[DEFAULT]
enabled_apis = osapi_compute,metadata

transport_url = rabbit://openstack:RABBIT_PASS@controller

my_ip = 192.168.42.154  # 计算节点compute1的IP地址

use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = 192.168.42.154  # 计算节点compute1的IP地址
novncproxy_base_url = http://192.168.42.169:6080/vnc_auto.html #  192.168.42.169 为控制节点的IP地址

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

#### 安装完成

确定您的计算节点是否支持虚拟机 hardware acceleration：
```
[root@compute1 ~]# egrep -c '(vmx|svm)' /proc/cpuinfo
2
```

如果此命令返回1，或者比1大，那表明你的计算节点支持（hardware acceleration）硬件加速，这通常不需要额外的配置。若返回0，则需要在 /etc/nova/nova.conf 文件中，在libvirt部分，修改成以下配置。

```
[libvirt]
virt_type = qemu
```

> 提示：若是在虚拟机安装，建议选择qemu。否则回报以下错误：RescheduledException: Build of instance 53bcf063-79d0-4f7f-afa4-ca58267b5e8d was re-scheduled: invalid argument: could not find capabilities for domaintype=kvm

启动服务

```
# systemctl enable libvirtd.service openstack-nova-compute.service
# systemctl start libvirtd.service openstack-nova-compute.service
```

#### 将计算节点添加到cell数据库

> 提示：在控制节点运行以下命令

```
[root@controller ~]#  openstack compute service list --service nova-compute
+----+--------------+----------+------+---------+-------+----------------------------+
| ID | Binary       | Host     | Zone | Status  | State | Updated At                 |
+----+--------------+----------+------+---------+-------+----------------------------+

|  8 | nova-compute | compute1 | nova | enabled | up    | 2022-03-12T10:19:32.000000 |
+----+--------------+----------+------+---------+-------+----------------------------+
```

将计算节点添加到cell数据库
```
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': d322dfe5-784f-4515-b3b0-c83cdcc2a03a
Checking host mapping for compute host 'compute1': 97065fc0-a822-44f0-b5fb-dd1b59c02ad2
Creating host mapping for compute host 'compute1': 97065fc0-a822-44f0-b5fb-dd1b59c02ad2
Found 1 unmapped computes in cell: d322dfe5-784f-4515-b3b0-c83cdcc2a03a
```