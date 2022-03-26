### [Placement](https://docs.openstack.org/placement/train/install/)

> 提示：在Stein版本之前，placement模块位于Nova模块中。请确保本文的发布版本与您要部署的发布版本匹配。

本小节将简单描述在控制节点如何安装placement服务。

#### 部署API服务

Placement提供了一个placement-api WSGI脚本，使用Apache、nginx或其他支持WSGI的Web服务器来运行该服务。

#### 数据库配置

- 创建数据库

```
CREATE DATABASE placement;
```

- 权限设置

```
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';

GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';  
```

#### 用户以及endpoint配置

- 在命令行导入 keystonerc_admin 环境变量

```
[root@controller ~]# source keystonerc_admin 
```

- 创建Placement服务

```
[root@controller ~]# openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 76384607366540a3b52e78ae3ba57249 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@controller ~]# 
```

- 将Placement用户添加到 service项目中,使用admin角色：

```
[root@controller ~]# openstack role add --project service --user placement admin
```

- 在服务目录中创建Placement API条目：

```
[root@controller ~]# openstack service create --name placement \
>   --description "Placement API" placement

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 2b9111e6a85e4fd9b12628ede92905bc |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

- 创建 Placement API 服务 endpoints

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   placement public http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 274326500d4b444aa3d8ad32805341ff |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2b9111e6a85e4fd9b12628ede92905bc |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   placement internal http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9519db4421074c6dbb90122b1e111ee8 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2b9111e6a85e4fd9b12628ede92905bc |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   placement admin http://controller:8778

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | d85dac2ac9614d13972c415eb1f0a003 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 2b9111e6a85e4fd9b12628ede92905bc |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

#### 安装和配置组件

- 安装Placement包

```
yum install openstack-placement-api
```

- 编辑 /etc/placement/placement.conf 文件，并添加以下内容:

```
[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
```

#### 初始化数据库

```
su -s /bin/sh -c "placement-manage db sync" placement
```

> 提示：若在执行该命令的过程中，产生任意异常信息，可以暂时忽略。

#### 重启Apache服务



```
systemctl restart httpd
```

#### 检查该服务是否可以正常运行

- 在命令行导入 keystonerc_admin 环境变量

```
root@controller ~]# source keystonerc_admin 
```

- 进行状态检查，以确保一切井然有序：

```
[root@controller ~]# placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```

-  安装 osc-placement 插件

```
 pip install osc-placement
```

> 提示：若当前系统没有安装pip，则需要进行安装。可参考该[链接](https://linuxize.com/post/how-to-install-pip-on-centos-7/)

- 列出可用的资源类和特征

```
[root@controller conf.d]# openstack --os-placement-api-version 1.2 resource class list --sort-column name
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| FPGA                       |
| IPV4_ADDRESS               |
| MEMORY_MB                  |
| MEM_ENCRYPTION_CONTEXT     |
| NET_BW_EGR_KILOBIT_PER_SEC |
| NET_BW_IGR_KILOBIT_PER_SEC |
| NUMA_CORE                  |
| NUMA_MEMORY_MB             |
| NUMA_SOCKET                |
| NUMA_THREAD                |
| PCI_DEVICE                 |
| PCPU                       |
| PGPU                       |
| SRIOV_NET_VF               |
| VCPU                       |
| VGPU                       |
| VGPU_DISPLAY_HEAD          |
+----------------------------+
```

```
 openstack --os-placement-api-version 1.6 trait list --sort-column name
```


> 提示：我在执行以上指令时，并没有执行成功。刚开始我尚未在意。直到在安装nova组件时，发现，发现计算节点并不能成功添加到cell数据库。然后查看计算节点的日志，发现以下错误：

```
WARNING keystoneauth.discover [req-f35b7fd4-f2ca-453f-8d44-0d87a5d07563 - - - - -] Failed to contact the endpoint at http://controller:8778 for discovery. Fallback to using that endpoint as the base url.: Forbidden: Forbidden (HTTP 403)
```

后面经过一番排查，发现placement组件存在异常,错误信息如下：

```
AH01630: client denied by server configuration: /usr/bin/placement-api
```

最后修改/etc/httpd/conf.d/00-nova-placement-api.conf配置文件，在文件最后追加下面的内容，并重启Apache服务后，该组件就能正常运行了。

```
<Directory /usr/bin>
  <IfVersion >= 2.4>
    Require all granted
  </IfVersion>
  <IfVersion < 2.4>
    Order allow,deny
    Allow from all
  </IfVersion>
</Directory>
```





