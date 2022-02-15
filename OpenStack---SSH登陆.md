
### 登陆控制台

- 访问该页面：http://192.168.42.150/dashboard/project/key_pairs
- 创建密钥对，密钥对创建完成后，浏览器会自动下载证书
- 将该证书放至该目录`~/.ssh/`
- 修改证书访问权限 `chmod 0600 yourPrivateKey.cer`
- 远程登录 `ssh -i ~/.ssh/demo-key.cer cirros@192.168.42.65`

[参考文档](https://docs.openstack.org/horizon/latest/user/configure-access-and-security-for-instances.html)