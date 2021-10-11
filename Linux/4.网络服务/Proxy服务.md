# Proxy服务

## 代理服务器

代理服务器(Proxy Server)是以类似代理人的身份去获取用户所需数据.通过代理服务器可以实现防火墙与用户浏览数据分析功能.此外还能充当CDN(Content Delivery Network)即内容分发网络功能,加速网络访问.

一般代理服务器会搭建在局域网的单点对外防火墙上.

Linux下常用的代理服务使用Squid软件.



## Squid配置文件

先使用yum安装squid服务:

```sh
[root@server2 ~]# yum install -y squid
```

squid服务使用的配置文件有:

- /etc/squid/squid.conf: squid的主要配置文件.
- /etc/squid/mime.conf: 设置squid所支持的文件格式,一般不需要改动.

squid服务相关目录有:

- /var/spool/squid: 默认squid缓存储存陌路.
- /usr/lib64/squid: 提供squid额外的控制模块.