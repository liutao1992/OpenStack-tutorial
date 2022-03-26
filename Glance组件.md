### [Glance 组件简介](https://docs.openstack.org/glance/train/install/install-rdo.html)

镜像服务就是用来管理镜像的。在openstack中提供镜像服务的是Glance。其主要功能如下：
- 查询和获取镜像的元数据和镜像本身。
- 注册和上传虚拟机镜像，包括镜像的创建、上传、下载和管理。
- 维护镜像信息，包括元数据和镜像本身。
- 支持多种方式存储镜像，包括普通的文件系统、Swift等。
- 对虚拟机实例执行创建快照命令来创建新的镜像，或者备份虚拟机状态。



本节介绍如何在控制节点上安装和配置镜像服务。为了简单起见，此配置将镜像存储在本地文件系统上。

### 安装

#### 数据库配置

在安装和配置镜像服务之前，您必须创建一个数据库、服务凭据和API端点。

#### 创建数据库

```
CREATE DATABASE glance;
```

#### 权限设置

```
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
```

#### 创建服务凭证

- 创建 glance 用户

```
openstack user create --domain default --password-prompt glance
```

- 将管理员（admin）角色授予glance用户和service项目

```
openstack role add --project service --user glance admin
```  

- 创建 glance 服务

```
openstack service create --name glance \
  --description "OpenStack Image" image
```  

#### 创建镜像服务API endpoint

```
[root@controller ~]# openstack endpoint create --region RegionOne \
>   image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 30e8ff6bbd7d4e579042d0f92913f164 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b224a261571453ebd94fa6cf1ae303f |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

```

```
[root@controller ~]# openstack endpoint create --region RegionOne   image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6932034e097845f195c83788abde8697 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b224a261571453ebd94fa6cf1ae303f |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

```
[root@controller ~]# openstack endpoint create --region RegionOne   image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | af1ed182d4c54e17ac26aebebcf0e1ce |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 6b224a261571453ebd94fa6cf1ae303f |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

#### 安装和配置组件

- 安装Glance包

```
yum install openstack-glance
```

- 编辑 `/etc/glance/glance-api.conf` 文件，内容如下

```
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone

[glance_store] # 配置镜像存储，定义本地文件系统以及存储路径
stores = file,http  
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

- 初始化数据库

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

#### 启动服务

```
systemctl enable openstack-glance-api.service

systemctl start openstack-glance-api.service
```

#### 检查服务是否正常运行

我们将使用CirroS镜像验证Glance服务是否可以正常工作，CirroS是一个小型Linux映像，可帮助你测试OpenStack部署。

- 下载CirroS镜像

```
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

- 将该镜像上传到Glance服务

```
glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility public
```

```
--name 该参数表示镜像名称
--file 该参数表示镜像路径
--disk-format 该参数表示镜像磁盘格式
--container-format 该参数表示镜像容器格式
--visibility public 该参数表示镜像是公共的
```

- 列出当前可用镜像

```
$ openstack image list

+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |
+--------------------------------------+--------+--------+
```

> 提示：若看到类似结果，表明Glance服务安装无误。

### 基于命令行管理镜像

- 查看镜像详情

```
openstack image show IMAGE_NAME|IMAGE_ID
```

- 删除镜像

```
openstack image delete IMAGE_NAME|IMAGE_ID
```