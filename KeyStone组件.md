### [Keystone组件](https://docs.openstack.org/keystone/train/install/keystone-install-rdo.html)

Keystone是一项OpenStack服务，通过实现[OpenStack’s Identity API](https://docs.openstack.org/api-ref/identity/index.html)，提供API客户端身份验证、服务发现和分布式多租户授权。 在安装OpenStack身份服务之后，其他OpenStack服务必须要在其中注册才能使用。由于Keystone服务主要用于认证，因此它又被称为认证服务。

#### KeyStone主要功能

KeysStone的基本功能如下：
- 身份认证 （Authentication）：令牌的发放与校验
- 用户授权（Authorization）：授予用户在一个服务中所拥有的权限
- 用户管理（Account）：管理用户账号
- 服务目录（Service Catalog）：提供可用服务的API端点

#### [KeyStone基本概念](https://docs.openstack.org/keystone/latest/admin/identity-concepts.html)

Keystone为每一项OpenStack服务都提供了身份服务。而身份服务使用domain, projects, users, and roles等组合来实现。

- 域（domain）

身份服务API v3实体。域是项目（project）和用户的集合，用于定义管理身份实体的管理边界。域可以代表个人、公司或运营商拥有的空间。他们直接向系统用户公开管理活动。用户可以被授予域的管理员角色。域管理员可以在域中创建项目、用户和组，并将角色分配给域中的用户和组。

- 项目（project）

对资源或身份对象进行分组或隔离的容器，也是一个权限组织形式。一个项目可以映射到客户、账户、组织、或租户。OpenStack用户要访问资源，必须通过一个项目向KeyStone发出请求。项目是OpenStack服务调度的基本单元，其中必须包括相关的用户和角色。

- 用户（users）

用户是指OpenStack云服务的个人、系统或服务的账户名称。OpenStack各个服务在身份管理体系中都被视为一种系统用户。KeyStone为用户提供认证令牌，让用户在调用OpenStack服务时拥有相应的资源使用权限。

- 角色（roles）

角色是一个用于定义用户权利和权限的集合。身份服务向包含一系列角色的用户提供一个令牌。当用户调用服务时，该服务解析用户角色设置，决定每个角色被授权访问哪些操作或资源。

### 安装
本节介绍如何在控制节点上安装和配置Identity服务。

#### 创建数据库

```
MariaDB [(none)]> CREATE DATABASE keystone;
```

#### 权限设置

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```

> KEYSTONE_DBPASS 表示密码字段，将该字段替换为密码即可

#### 安装 Apache 服务

```
yum install openstack-keystone httpd mod_wsgi
```

#### 修改配置文件`/etc/keystone/keystone.conf `

```
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
provider = fernet
```

> 提示 controller 为当前控制节点的主机名。后面的所有示列中，则不再强调说明。

#### 初始化数据库

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

> 命令执行完成后可以登录数据库查看keystone库里面有没有对应的表，如果有则表示初始化成功。如果初始化数据库不成功可以查看keystone的日志，日志文件路径为：/var/log/keystone/keystone.log

#### 初始化Fernet库

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

#### 启动keystone服务

```
keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

> 将ADMIN_PASS替换为适合管理用户的密码。

- bootstrap-admin-url：管理员接口地址
- bootstrap-internal-url：内网接口地址
- bootstrap-public-url：外网接口地址
- bootstrap-region-id：指定Region的名称，可以指定一个高大上的名称，如阿里云的north-1、east-2等

#### 配置Apache服务

- 编辑`/etc/httpd/conf/httpd.conf`文件，并配置`ServerName`选项

```
ServerName controller 
```
> controller 为控制节点的主机名

- 创建软连接

```
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

#### 安装完成

- 设置开机启动、

```
systemctl enable httpd.service
systemctl start httpd.service
```

- 设置管理员账号

我们可以为OpenStack客户端创建一个环境变量脚本。在/root/目录下，新建文件`keystonerc_admin`，在该文件添加环境变量

```
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

这里显示的这些值是在keystone-manage bootstrap命令启动时创建的默认值。

> 该密码`ADMIN_PASS `需与刚刚`keystone-manage bootstrap`时设定的保持一致

#### 创建 domain, projects, users, and roles
Identity服务为每个OpenStack服务提供身份验证服务。身份验证服务使用一个组合，该组合包括域、项目、用户和角色。

- 创建域

在执行keystone-manage bootstrap命令时，已经创建了一个名为default的域了。这里我们处于演示的目的创建一个名为example的域。
```
openstack domain create --description "An Example Domain" example
```

- 创建项目

创建一个名为service的项目，域为default。

```
openstack project create --domain default \
  --description "Service Project" service
```

- 创建用户

创建一个名为demo的用户

```
openstack user create --domain default \
  --password-prompt demo
```

- 创建角色

```
openstack role create myrole
```  

- 将myrole角色添加到service项目和demo用户中：

```
openstack role add --project service --user demo myrole
```

#### 检查服务是否可以正常运行

- 在命令行，导入keystonerc_admin环境变量，然后执行命令执行以下指令：

```
[root@controller ~]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2022-03-14T03:47:13+0000                                                                                                                                                                |
| id         | gAAAAABiLqyxwpvlcPyKXnUDURiawDGxsvQa00srXm9hhGpcJlsk6iO7a4uT-KGHvJ1imOfZbiOQeMHXSB8OkI6L31jRjdkk6mhrUPXBJvsEVFYsXTptdiuKL_x7x8BSGmXA2jPOLUARe7PYY-r_5M-5Cprz8hpfLe2gBrPYQ6orFrQvzR10Zvw |
| project_id | ee480631b6784daaa5c2d083825ad778                                                                                                                                                        |
| user_id    | ccaf60e35dfb4665ab4bebf113f0d826                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

如果可以拿到用户token，则表明服务运行正常，则继续安装下一个组件。

### 基于命令行界面进行身份管理操作

- 列出域

```
[root@controller ~]# openstack domain list
+----------------------------------+---------+---------+--------------------+
| ID                               | Name    | Enabled | Description        |
+----------------------------------+---------+---------+--------------------+
| default                          | Default | True    | The default domain |
| eb05e7aaca094d58aa2b2ca70d5be0f3 | example | True    | An Example Domain  |
+----------------------------------+---------+---------+--------------------+
```
#### 服务管理

- 列出可用服务

```
[root@controller ~]# openstack service list
+----------------------------------+-----------+-----------+
| ID                               | Name      | Type      |
+----------------------------------+-----------+-----------+
| 0259d731bd2741839e281dc5fa23c12f | keystone  | identity  |
| 2b9111e6a85e4fd9b12628ede92905bc | placement | placement |
| 5dc83d1b1b5040219d2632b64a261048 | nova      | compute   |
| 6b224a261571453ebd94fa6cf1ae303f | glance    | image     |
| abf0d5cd3d7b40ad9beb2d66724e67a1 | neutron   | network   |
+----------------------------------+-----------+-----------+
```

- 创建服务

```
$ openstack service create --name SERVICE_NAME --description SERVICE_DESCRIPTION SERVICE_TYPE
```

其中，参数SERVICE_NAME表示新创建服务的唯一名称；SERVICE_TYPE为服务类型，值可以是identity、compute、image、network等；SERVICE_DESCRIPTION 表示该服务的说明信息。

- 查看服务的详细信息 `openstack service show SERVICE_NAME|SERVICE_ID`

```
[root@controller ~]# openstack service show keystone 
+---------+----------------------------------+
| Field   | Value                            |
+---------+----------------------------------+
| enabled | True                             |
| id      | 0259d731bd2741839e281dc5fa23c12f |
| name    | keystone                         |
| type    | identity                         |
+---------+----------------------------------+
```

- 创建服务用户

 1）创建一个服务用户专用的项目
 ```
 openstack project create service --domain default
 ```
 这个项目一般命名为service，也可以选择其他。
 
 2）要为部署的相关服务创建服务用户
 
 例如：创建nova用户的命令如下：
 ```
 $ openstack user create --domain default --password-prompt nova
 ```
 3）将admin角色分配给用户-项目对
 ```
 $ openstack role add --project service --user 服务用户名 admin
 ```
 
- 删除服务

```
openstack service delete  SERVICE_NAME|SERVICE_ID
```

##### 项目管理

- 列出项目

```
[root@controller ~]# openstack project list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 2966862f8f6548be983535b0d526ea25 | service   |
| e62d5bfbc04846e99dc67202701d191c | myproject |
| ee480631b6784daaa5c2d083825ad778 | admin     |
+----------------------------------+-----------+
```

- 查看项目信息 `openstack project show 项目名称或ID`

```
[root@controller ~]# openstack project show  2966862f8f6548be983535b0d526ea25
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 2966862f8f6548be983535b0d526ea25 |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

- 创建项目

创建一个名为new-project的项目，域为default。

```
[root@controller ~]# openstack project create --description 'my new project' new-project --domain default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | my new project                   |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 505815a5c02e4f028fe960fcd6e291c2 |
| is_domain   | False                            |
| name        | new-project                      |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

- 删除项目  `openstack project delete 项目名称或ID`

```
[root@controller ~]# openstack project delete 505815a5c02e4f028fe960fcd6e291c2
```

- 修改项目

修改项目需要指定项目名称或ID，可以修改项目的名称、描述信息和激活状态。

 临时禁用某项目：`OpenStack project set  项目名称或ID` --disable
 激活已禁用项目：`OpenStack project set  项目名称或ID` --enable
 修改项目名称：`OpenStack project set 项目名称或ID` --name xxxx`
 

##### 用户管理

- 列出用户

```
[root@controller ~]# openstack user list
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| ccaf60e35dfb4665ab4bebf113f0d826 | admin  |
| fe666027ccec4723bb83ba459c8a3ba2 | myuser |
+----------------------------------+--------+
```

- 创建用户

要创建用户，您必须指定用户名。或者，您可以指定项目ID、密码和电子邮件地址。建议你附上项目ID和密码，因为如果没有这些信息，用户就无法登录dashboard页面。

```
[root@controller ~]# openstack user create  --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | 2966862f8f6548be983535b0d526ea25 |
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a6075d0e02b34b7e98d4d08479371e7d |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 删除用户 `openstack user delete 用户名`

```
[root@controller ~]# openstack user delete demo
```

##### 角色管理

- 列出角色

```
[root@controller ~]# openstack role list
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 003fadce0e054ed997379e8215b13a4a | myrole |
| 12d6957cd89d419cbb3ec7ab638b6e29 | admin  |
| 18b76b4f31e54e6c855b61f0ab82763b | reader |
| 77ca8961f9c948ba84a0d3fe557221b2 | member |
+----------------------------------+--------+
```

- 创建角色

用户可以是多个项目的成员，要将用户分配给多个项目。需要定义一个角色，并将该角色分配给用户-项目对（user-project pair）。

创建一个叫 new-role 的角色

```
[root@controller ~]# openstack role create new-role
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | a34425c884c74c8881496dc2c2e84ffc |
| name        | new-role                         |
+-------------+----------------------------------+
```

- 分配角色

要将用户指派给项目，必须将角色赋予用户-项目对，这需要指定用户、角色和项目ID。将角色分配给用户-项目对的命令如下：

```
$ openstack role add --user USER_NAME --project PROJECT_NAME ROLE_NAME
```

如将myrole角色添加到service项目和demo用户中：

```
openstack role add --project service --user demo myrole
```

-  验证角色分配用法如下：

```
[root@controller ~]# openstack role assignment list --user demo --project service --names
+--------+--------------+-------+-----------------+--------+--------+-----------+
| Role   | User         | Group | Project         | Domain | System | Inherited |
+--------+--------------+-------+-----------------+--------+--------+-----------+
| myrole | demo@Default |       | service@Default |        |        | False     |
+--------+--------------+-------+-----------------+--------+--------+-----------+
```

- 查看角色详细 `openstack role show new-role`


- 移除角色

```
$ openstack role remove --user USER_NAME --project PROJECT_NAME ROLE_NAME
```

查看是否移除成功

```
$ openstack role assignment list --user USER_NAME --project PROJECT_NAME --names
```