---
title: https搭建(自签名证书)
---
上一篇博客探究了https（ssl）的原理，为了贯彻理论落实于实践的宗旨，本文将记录我搭建https的实操流程，使用Apache2+ubuntu+opensssl
# 1.使用自签证书配置https
一般来讲，正式的上线项目都需要购买域名，并且向权威机构申请证书。但本次工作属于测试环境，所以一切从简，我们使用openssl工具生自签名的CA证书以及服务器证书，来搭建https。具体步骤如下：
## （1）安装apache2、openssl(ubuntu16.04)
安装过程此处不做赘述，很简单。Apache2安装完成并启动后，通过http://ipaddress 来测试，如下图所示，说明安装成功
![image](http://pkzsficup.bkt.clouddn.com/apacheOk.png)
## （2）制作证书
### 生成自签CA证书
- 生成CA私钥
```
 openssl genrsa -out ca.key 2048

```
- 生成CA证书
```
 openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -config /etc/ssl/openssl.cnf
```
openssl.cnf的路径注意填对。最后在当前目录下生成的ca.key即为CA私钥，ca.crt即为CA证书（包含CA公钥）
### 生成服务器证书
- 生成服务器私钥
```
 openssl genrsa -out server.key 2048
```
- 生成服务器签署申请文件

```
 openssl req -new -out server.csr -key server.key -config /etc/ssl/openssl.cnf 
```
需要填写服务器信息，如实填写即可。需要注意Common Name 需要与openssl.cnf 中配置的域名相对应（alt_names），否则客户端无法验证。该命令最后生成的server.csr即为申请文件。openssl.cnf的具体配置见下文。
- 在签署证书之前需要确认openssl.cnf中的配置，如下所示：

```
#根据实际情况修改，将match改成optional，否则ca.crt必须与server.csr中的各个字段值一致才能签署
[ policy_match ]
countryName             = optional
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
# 确保req下存在以下2行（默认第一行是有的，第2行被注释了）
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
# 确保req_distinguished_name下没有 0.xxx 的标签，有的话把0.xxx的0. 去掉
[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = XX
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
localityName                    = Locality Name (eg, city)
localityName_default            = Default City
organizationName                = Organization Name (eg, company)
organizationName_default        = Default Company Ltd
organizationalUnitName          = Organizational Unit Name (eg, section)
commonName                      = Common Name (eg, your name or your server\'s hostname)
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64
#添加一行subjectAltName=@alt_names
[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName=@alt_names
#新增alt_names,注意括号前后的空格，DNS.x 的数量可以自己加
#如果没有IP这一项，浏览器使用IP访问时验证无法通过
[ alt_names ]
IP.1 = 192.168.50.115
DNS.1 = dfe.leida.org
DNS.2 = ex.abcexpale.net
```

- 使用CA证书和私钥签署服务器证书

```
openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -extensions v3_req -config /etc/ssl/openssl.cnf
```
直接运行该签署命令，会报错，提示缺少某些文件和目录。在当前目录下把相应的文件及文件夹创建上即可解决，如下

```
 mkdir -p demoCA/newcerts
 touch ./demoCA/index.txt ./demoCA/serial
 echo "01">> ./demoCA/serial
```
创建完毕后，运行签署命令即可完成服务器证书的制作。当前目录下生成的server.crt即为服务器证书，server.key为服务器私钥
## （3）配置apache2

- 启用ssl模块(此处可使用a2enmod命令)

```
 a2enmod ssl
```

- 启用ssl站点

```
 a2ensite default-ssl
```

- 加入监听端口443(因为https默认采用443端口，有别于http的80)

```
listen 80 443
# /etc/apache2/ports.conf中修改成如上所示即可
```
- 配置证书以及私钥的路径

```
SSLCertificateFile     /etc/ssl/certs/server.crt
SSLCertificateKeyFile /etc/ssl/private/server.key
# 在/etc/apache2/sites-available/default-ssl.conf中修改如上参数，确认证书和私钥的路径正确无误
```
重启apache2，搞定！
# 2.使用浏览器测试https
上述的制作证书步骤，已经是我测试结束、爬完坑后的正确教程，下文则是填坑记录。

实际上，我一开始按照网上的方法搭建完https后，出现过一些问题。使用浏览器访问https://ipaddress 会跳出如下提示界面
![image](http://pkzsficup.bkt.clouddn.com/httpsNotSafe.PNG)
实际上此时浏览器已获取到服务器证书server.crt，只是由于某些原因无法验证它。若选择高级选项中的“继续浏览”时，同样可以正常跳转至对应的页面，这时相当于强行让浏览器接受该证书，同时接受服务器公钥。在测试环境下，这样完全OK。但是，强迫症的我还是想让导航栏上出现安全的小锁，红叉叉看着很难受啊。

下面总结记录一下使用浏览器（chrome）测试的过程中遇到的问题以及解决方法
## （1）客户端缺少CA证书
点击导航栏左侧的感叹号，查看证书，如下图所示
![image](http://pkzsficup.bkt.clouddn.com/caNotBlve.PNG)
此时浏览器已获取到server.crt，由于其对应的CA证书不存在于“受信任的根证书机构”中，所以无法验证server.crt

我们需要将CA证书（上文使用OpenSSL生成的ca.crt）安装至客户端系统中的“受信任的根证书机构”中，保证浏览器可通过此CA证书来验证服务器证书server.crt。安装过程很简单，双击如下所示安装即可

![image](http://pkzsficup.bkt.clouddn.com/installCert.PNG)
安装结束后，重启浏览器，再次打开证书，如下所示
![image](http://pkzsficup.bkt.clouddn.com/caOk.PNG)

此时浏览器已经可以使用ca证书来验证server.crt，已完成了很重要的一步，但我的chrome浏览器依然没有出现小锁

## （2）缺少使用者备用名称
完成上一步后，浏览器的导航栏依然显示不安全，我打开开发者工具，查看security得知，是服务器证书缺少“使用者备用名称”所导致，如下图所示。
![image](http://pkzsficup.bkt.clouddn.com/missAltName.PNG)
如何解决这个问题，我参考了这个博客（http://blog.51cto.com/colinzhouyj/1566438 ），即修改openssl.cnf的部分配置，前文“制作证书”步骤中已经整合过此处的配置内容。修改完，重新签署证书，重启apache2，再次运行，Subject Alternative Name missing的问题解决。

## （3）IP验证不通过
在上一步操作中，我添加完几条DNS之后，发现chrome依然报错，如下图所示
![image](http://pkzsficup.bkt.clouddn.com/missCert.PNG)
我用的是虚拟机环境，没有域名解析，而是直接通过IP来访问站点的，又由于证书中不存在IP信息，是无法跟URL中的IP进行比对的，因此浏览器无法通过证书验证。该问题有两个解决方法

（1）在证书中添加IP地址。在openssl.cnf中配置使用者可选名称时，添加一条IP地址，即可保证浏览器通过对IP的验证，重新签署配置运行后，结果如下
![image](http://pkzsficup.bkt.clouddn.com/ipSafeLock.PNG)

打开证书，使用者可用名称字段如下

![image](http://pkzsficup.bkt.clouddn.com/altName.PNG)

（2）使用域名访问。在客户机的host文件中配置域名与IP的映射，直接使用域名来访问站点（证书中包含域名），即可通过验证，如下所示

![image](http://pkzsficup.bkt.clouddn.com/domainSafeLock.PNG)

使用这两种方法的其中一种，迟迟不肯露面的小锁终于出现了，打开security也是一片绿色，大功告成。但是，要注意一点，不同浏览器对于证书的验证方式可能存在差异。按照我的操作，firefox依然无法验证，暂时没细究，但chrome和edge浏览器都可以完美运行。