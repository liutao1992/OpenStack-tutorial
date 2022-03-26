
### [Horizon部署](https://docs.openstack.org/horizon/train/install/install-rdo.html)

本节介绍如何在控制节点上安装和配置仪表板。

#### 安装包

```
yum install openstack-dashboard
```

#### 设置

编辑/etc/openstack-dashboard/local_settings文件，然后设置为以下内容：

```
OPENSTACK_HOST = "controller"    # 设置Horizon组件在控制节点上运行

WEBROOT = '/dashboard'           # 这部分文档上并没有，但需要添加上，否则无法访问登录页面

ALLOWED_HOSTS = ['*']            # 生产环境则根据需要进行设置


SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Shanghai"
```

#### 修改/etc/httpd/conf.d/openstack-dashboard.conf，在文件中需要加入一行：WSGIApplicationGroup %{GLOBAL}，如下所示：

```
WSGIDaemonProcess dashboard
WSGIProcessGroup dashboard
WSGIApplicationGroup %{GLOBAL}  # 新添加项

WSGISocketPrefix run/wsgi
WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi
Alias /dashboard/static /usr/share/openstack-dashboard/static

<Directory /usr/share/openstack-dashboard/openstack_dashboard/wsgi>
  Options All
  AllowOverride All
  Require all granted
</Directory>

<Directory /usr/share/openstack-dashboard/static>
  Options All
  AllowOverride All
  Require all granted
</Directory>
```

#### 重启服务

```
systemctl restart httpd.service memcached.service
```

#### 浏览器访问

接下来就可以在浏览器中输入地址 http://controller/dashboard，就可以访问我们的控制台了。


> 提示：在环境搭建完成后，我来到登录界面，输入用户名密码后无法访问主页，后面查看日志 /var/log/httpd/error_log，提示这个：RuntimeError: Unable to create a new session key. It is likely that the cache is unavailable. 。
于是我更改了SESSION_ENGINE, 将其设为文本存储，django.contrib.sessions.backends.file，修改完成后重启httpd和memcached 即可登录。至于为何无法cache存储，暂时没做深入研究。