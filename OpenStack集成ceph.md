
#### [创建 POOL](https://docs.ceph.com/en/latest/rbd/rbd-openstack/#create-a-pool)

> 注意：不建议使用QCOW2托管虚拟机磁盘。如果您想在Ceph中启动虚拟机，请使用Glance中的raw镜像格式。

我们分别为Cinder， Glance 创建 pool，同时也要确保ceph集群正在运行。


```
root@ceph-01 ~]# ceph osd pool create volumes
root@ceph-01 ~]# ceph osd pool create images
root@ceph-01 ~]# ceph osd pool create backups
root@ceph-01 ~]# ceph osd pool create vms
```

使用rbd工具初始化pool

```
root@ceph-01 ~]# rbd pool init volumes
root@ceph-01 ~]# rbd pool init images
root@ceph-01 ~]# rbd pool init backups
root@ceph-01 ~]# rbd pool init vms
```

#### 配置OpenStack的ceph客户端

运行 glance-api、inder-volume、nova-compute和cinder-backup的节点充当Ceph客户端。每个都需要ceph.conf文件：

```
ssh {your-openstack-server} sudo tee /etc/ceph/ceph.conf </etc/ceph/ceph.conf
```

> 提示：控制节点，计算节点，以及存储节点都需要ceph.conf文件.

##### 在控制节点，计算节点，存储节点安装ceph客户端

- 在 glance-api 节点上，您需要安装Python包`librbd` 
```
[root@controller-node ~]# sudo yum install python-rbd
```
- 在nova-compute、cinder-backup和cinder-volume节点上，安装以下包

```
sudo yum install ceph-common
```

#### 配置cephx认证

默认情况下，cephx认证是开启的。为Nova/Cinder和Glance创建一个新用户。

```
[root@ceph-01 ~]# ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images'

[root@ceph-01 ~]# ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms'

[root@ceph-01 ~]# ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups'
```


> 提示：若有cinder-backup服务，则创建该用户。

将client.cinder、client.glance和client.cinder-backup的秘钥key添加到适当的节点并更改相应的权限：

```
[root@ceph-01 ~]# ceph auth get-or-create client.glance | ssh {your-glance-api-server} sudo tee /etc/ceph/ceph.client.glance.keyring

[root@ceph-01 ~]# ssh  {your-glance-api-server} sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring

[root@ceph-01 ~]# ceph auth get-or-create client.cinder | ssh {your-cinder-volume-server} sudo tee /etc/ceph/ceph.client.cinder.keyring

[root@ceph-01 ~]# ssh {your-cinder-volume-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

[root@ceph-01 ~]# ceph auth get-or-create client.cinder-backup | ssh {your-cinder-backup-server} sudo tee /etc/ceph/ceph.client.cinder-backup.keyring

[root@ceph-01 ~]# ssh {your-cinder-backup-server} sudo chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring

```

运行nova-compute的节点需要nova-compute的秘钥key文件：

```
[root@ceph-01 ~]# ceph auth get-or-create client.cinder | ssh {your-nova-compute-server} sudo tee /etc/ceph/ceph.client.cinder.keyring
```

> 提示：若有多个计算节点，则每个计算节点都需要该秘钥key文件

他们还需要在libvirt中存储client.cinder用户的秘钥key。libvirt进程需要它访问集群，同时从Cinder连接块设备。

在运行nova-compute的节点上创建密钥key的临时副本：

```
[root@ceph-01 ~]# ceph auth get-key client.cinder | ssh {your-compute-node} tee client.cinder.key
```

然后，在计算节点上，将密钥添加到libvirt，并删除密钥key的临时副本：

secret.xml 模板文件
```
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>f4d17e93-104e-4feb-85f4-eab1b290c42b</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```


> 提示：这里计算节点1为例：
```
[root@compute-node-1 ~]# uuidgen
f4d17e93-104e-4feb-85f4-eab1b290c42b
[root@compute-node-1 ~]# cat > secret.xml <<EOF
> <secret ephemeral='no' private='no'>
>   <uuid>f4d17e93-104e-4feb-85f4-eab1b290c42b</uuid>
>   <usage type='ceph'>
>     <name>client.cinder secret</name>
>   </usage>
> </secret>
> EOF

[root@compute-node-1 ~]# sudo virsh secret-define --file secret.xml
Secret f4d17e93-104e-4feb-85f4-eab1b290c42b created

[root@compute-node-1 ~]# sudo virsh secret-set-value --secret f4d17e93-104e-4feb-85f4-eab1b290c42b --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml
Secret value set

rm: remove regular file ‘client.cinder.key’? y
rm: remove regular file ‘secret.xml’? y
```

> 提示：在执行该指令前，先检查一下client.cinder.key文件的路径，然后在cat指令后替换为你client.cinder.key文件的路径。

> 建议所有计算节点最好保持相同的UUID。同时注意此处生成的UUID的值，后面Cinder以及Nova的配置中需要用到


#### 配置openstack使用 CEPH

##### 设置Glance使用ceph存储镜像

- 在控制节点上，编辑 /etc/glance/glance-api.conf，在[glance_store] 部分添加以下内容

```
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

- 镜像属性

我们建议对您的镜像使用以下属性：


##### 配置CINDER

OpenStack需要一个驱动与Ceph块设备进行交互，你还必须为块设备指定pool名。在OpenStack节点，修改配置文件/etc/cinder/cinder.conf

> 提示：若你尚未单独部署存储节点，则在控制节点修改/etc/cinder/cinder.conf 文件即可

```
[DEFAULT]
...
enabled_backends = ceph
# glance_api_version = 2    # 若有多个enabled_backends，则需要配置该选项
...
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
```

如果你启用了cephx身份验证，还需要配置你添加到libvirt的secret的用户和uuid：

```
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
```

#### 配置 CINDER BACKUP

> 若有CINDER BACKUP 服务，则/etc/cinder/cinder.conf 文件中添加以下配置

```
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728
backup_ceph_pool = backups
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

#### 配置Nova



















