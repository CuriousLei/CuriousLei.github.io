---
title: linux常用命令总结
---

- 根目录下查找名称前缀为my的文件(文件夹)

```
 find / -name 'my'
```

- linux向linux复制文件，-P（大写P）表示端口号（默认22）

```
scp -P 2233 passwordModify_encrypt.php     root@118.192.66.57:/var/www/opensips-cp/web/
```

- ssh远程登录（小写p）

```
ssh -p 2233 root@118.192.66.57
```
- windows下向linux拷贝文件

```
pscp -l leida file.zip 192.168.50.24:/home/leida/
```


- 查找目录下包含某一字段的文件

```
grep "client" ./ -nr
```
- iptables添加开放端口

```
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
service iptables save
service iptables restart

-A 指append添加规则(末尾)，-I指插入，-D指删除
INPUT指输入数据包，相对的是OUTPUT
--dport指目的端口，相对的是sport源端口
-j ACCEPT 表示动作，相对的是DROP
注意：新添加的规则要插入到 RH-Firewall-1-INPUT 之前才能生效，具体原因有待探究
```
- ubuntu下卸载apache2

```
apt purge remove apache2
apt purge remove apache2.2-common
apt autoremove

find /etc/ -name "*apache*" | xargs rm -rf
rm -rf /var/www

```
- 查看linux版本

```
cat /etc/issue
```

- 查看仓库包

```
yum repolist all
```
- centos7加载本地光盘yum源

```
mount /dev/cdrom/ /mnt/
cd /etc/
mv yum.repos.d yum.repos.d.bak
mkdir yum.repos.d
vim yum.repos.d/Centos-Local.repo

[Local]
name=Local Yum
baseurl=file:///mnt/
gpgcheck=0
enabled=1

yum clean 
yum makecache
```
- centos修改root密码
https://blog.csdn.net/akipa11/article/details/81395386
- nginx相关命令

```
/usr/local/nginx/sbin/nginx
# 启动
/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
# 检查配置是否正确
/usr/local/nginx/sbin/nginx -s reload
# 修改配置后使配置生效

```

- centos7命令行和图形界面互相切换

```
vim /etc/inittab
systemctl get-default 
systemctl set-default multi-user.target
systemctl set-default multi-user.target
```
- 创建多层文件夹

```
 mkdir -p demoCA/newcerts
 touch ./demoCA/index.txt ./demoCA/serial #创建多个文件 
```
- grep满足多个可选关键字

```
grep -E "me|data" meDec.php
```
- grep排除字段

```
grep -v "message" meDec.php
```



