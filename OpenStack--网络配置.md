
### 网络配置：将网络接口与外部桥接口br-ex进行关联

- 执行指令 `cd /etc/sysconfig/network-scripts/`
- 执行指令 `cp ifcfg-ens33 ifcfg-bx-ex`

- 编辑文件 `vim ifcfg-ens33`,内容如下：

```
TYPE=OVSPort
UUID="07904201-4e09-42e5-aec6-3d255e337986"
ONBOOT="yes"
DEVICE=ens33
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
```

- 编辑文件 `vim ifcfg-br-ex`, 内容如下：
  
```
OOTPROTO="static"
DEVICE="br-ex"
DEVICETYPE=ovs
TYPE=OVSBridge
ONBOOT="yes"
IPADDR=192.168.42.149
NETMASK=255.255.255.0
GATEWAY=192.168.42.2
DNS1=114.114.114.114
```

- 重启network `systemctl restart network`

### 配置虚拟网络


### 为虚拟机实例分配浮动IP地址


  
