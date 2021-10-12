# Squid服务

## 代理服务器

代理服务器(Proxy Server)是以类似代理人的身份去获取用户所需数据.通过代理服务器可以实现防火墙与用户浏览数据分析功能.此外还能充当CDN(Content Delivery Network)即内容分发网络功能,加速网络访问.

一般代理服务器会搭建在局域网的单点对外防火墙上.

Linux下常用的代理服务使用Squid软件.



## 配置文件

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



## 默认配置

在默认情况下squid配置有下面一些设置:

- 仅有本机(localhost, 127.0.0.1)来源可以使用squid功能.
- squid所监听的代理服务端口为3128.
- ~~缓存目录/var/spool/squid/仅有100MB缓存量.~~
- ~~内存分配8MB大小作为高速缓存使用.~~
- 启动squid程序的用户为squid账号.

默认配置文件内容如下:

```sh
[root@server2 ~]# cat /etc/squid/squid.conf
# 信任用户与目标控制
acl localnet src 10.0.0.0/8     
acl localnet src 172.16.0.0/12  
acl localnet src 192.168.0.0/16 # 定义局域网来源
acl localnet src fc00::/7       # RFC 4193 local private network range
acl localnet src fe80::/10      # RFC 4291 link-local (directly plugged) machines

# 定义安全端口
acl SSL_ports port 443
acl Safe_ports port 80          # http
acl Safe_ports port 21          # ftp
acl Safe_ports port 443         # https
acl Safe_ports port 70          # gopher
acl Safe_ports port 210         # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280         # http-mgmt
acl Safe_ports port 488         # gss-http
acl Safe_ports port 591         # filemaker
acl Safe_ports port 777         # multiling http
acl CONNECT method CONNECT

# 拒绝非正规端口连接请求
http_access deny !Safe_ports

# 拒绝非正规加密端口连接请求
http_access deny CONNECT !SSL_ports

# 允许本机管理,拒绝其他来源地址管理
http_access allow localhost manager
http_access deny manager

# 放行本机与内部网络的用户来源,其他予以拒绝
http_access allow localnet
http_access allow localhost
http_access deny all

# 监听客户端请求的端口.如果想加密连接可改为https_port 923
http_port 3128

# 磁盘缓存放置目录与大小,内存高速缓存的大小
cache_dir ufs /var/spool/squid 100 16 256
cache_mem 8 MB

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid

# 额外设置
refresh_pattern ^ftp:           1440    20%     10080
refresh_pattern ^gopher:        1440    0%      1440
refresh_pattern -i (/cgi-bin/|\?) 0     0%      0
refresh_pattern .               0       20%     4320
```



## 服务启动

可以使用默认配置启动squid:

```sh
[root@server2 ~]# systemctl start squid
[root@server2 ~]# netstat -ntulp |grep squid
tcp6       0      0 :::3128                 :::*             LISTEN      101226/(squid-1)
udp        0      0 0.0.0.0:56694           0.0.0.0:*                    101226/(squid-1)
udp6       0      0 :::53318                :::*                         101226/(squid-1) 
```



## 缓存目录

磁盘缓存设置`cache_dir ufs /var/spool/squid 100 16 256`后三数字代表: 大小MB 一层索引数 二层索引数

```sh
[root@server2 ~]# ls /var/spool/squid/
00  01  02  03  04  05  06  07  08  09  0A  0B  0C  0D  0E  0F  swap.state
[root@server2 ~]# ls /var/spool/squid/00
00  0B  16  21  2C  37  42  4D  58  63  6E  79  84  8F  9A  A5  B0  BB  C6  D1  DC  E7  F2  FD
```

因为squid会将数据分成多个小块,这样便于索引.

可以多加一行配置来增加磁盘缓存目录.例如:`cache_dir ufs /cache 2000 64 64`.同时需要修改目录权限:

```sh
[root@server2 ~]# mkdir /cache
[root@server2 ~]# chmod 750 /cache
[root@server2 ~]# chown squid:squid /cache
[root@server2 ~]# chcon --reference /var/spool/squid/ /cache
[root@server2 ~]# ll -Zd /cache
drwxr-x---. squid squid system_u:object_r:squid_cache_t:s0 /cache
```

为了防止缓存占满硬盘,可以在配置中设置下面两个值,在磁盘使用率95%时删除旧数据直到到磁盘用量90%:

```sh
cache_swap_low 90
cache_swap_high 95
```

cache_mem的值指额外提供给squid作为缓存使用的内存大小.默认1GB磁盘高速缓存会占用约10M内存.



## 信任来源

squid中来源与目标过滤使用acl来定义列表,用http_access来对acl采取动作.设置acl的语法为: 

`acl <自定义acl名称> <要控制acl类型> <设置内容>`

重要的acl类型设置如下:

- 设置请求客户端列表方式有下面几种:
  - src ip-address/netmask: 通过IP设置网段,例如`acl plan src 10.1.1.0/24 10.1.2.0/24`.
  - src addr1-addr2/netmask: 连续IP地址设置,例如`acl plan src 10.1.1.100-10.1.1.200/24`.
  - srcdomain .domain.name: 通过主机名设置,例如`acl pdo srcdomain .ere.edu.us`.
- 设置访问目标列表方式有下面几种:
  - dst ip-addr/netmask: 通过IP设置,例如`acl drp dst 134.144.2.54/32`.
  - dstdomain .domain.name: 通过域名设置,例如`acl drp dstdomain .facebook.com`.
  - url_regex [-i] ^http://url: 用正则表达式匹配网址,例如`acl kurl url_regex ^http://baidu.com/.*`.
  - urlpath_regex [-i] \.gif\$: 用正则匹配url内容,例如`acl sx urlpath_regex /sex.*\.jpg$`.
  - 匹配文件: 可以将地址按行保存到文件来调用,例如`acl drplst dstdomain "/root/dpl.txt"`.

设置好acl后,可以用http_access来对acl列表执行允许(allow)与拒绝(deny)操作.http_access严格按顺序执行.

例如放行内部网络10.1.1.0/24,拒绝访问facebook.com网站设置如下:

```sh
acl lanet src 10.1.1.0/24
acl drpdm dstdomain .facebook.com
http_access deny drpdm
http_access allow lanet
```



## 其他设置

可以设置不要对某些网页进行缓存操作,否则客户端无法请求到网页最新副本.例如不要对php网页进行缓存:

```sh
acl denyphp urlpath_regex \.php$
cache deny denyphp
```

在refresh_pattern段设置,例如:`refresh_pattern ^ftp:  1440  20%  10080`从第二列开始代表:

- regex: 用正则表达式分析网址数据,例如^ftp表示以ftp开头的网址.
- 最小时间: 单位分钟,表示缓存数据存放到达这一时间后失效,新请求会从网络重新获取数据.
- 百分比: 与最大时间有关,当数据被获取到缓存后,经过最大时间多少百分比,数据会被重新获取.
- 最大时间: 数据存在缓存内的最大时间,到达这一时间后数据会被删除.

管理员的邮箱地址可以通过cache_mgr来设置,在发生错误时会发送邮件.例如设置邮箱为root@server2:

```sh
cache_mgr root@server2
visible_hostname server2
```



## 安全设置

针对防火墙需要开放3128端口:

```sh
[root@server2 ~]# iptables -A INPUT -i ens37 -p tcp -s 10.1.1.0/24 --dport 3128 -j ACCEPT
```

SELinux中没有规则限制,其中/etc/squid/内配置文件类型是squid_conf_t,缓存目录类型squid_cache_t.

通过将拒绝网站写入文件中调用处理起来更灵活.例如使用/root/drp.txt来记录禁止网站:

```sh
[root@server2 ~]# vi /root/drp.txt
.facebook.com
.google.com
[root@server2 ~]# vi /etc/squid/squid.conf
acl drplist dstdomain "/root/drp.txt"
http_access deny drplist
[root@server2 ~]# systemctl reload squid
```



## 客户端测试

这里使用curl来测试.默认情况下可以访问百度:

```sh
[root@server3 ~]# curl baidu.com
<html>
<meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
</html>
```

在squid中添加规则后重新载入配置:

```sh
[root@server2 ~]# vi /etc/squid/squid.conf
acl nobaidu dstdomain .baidu.com
http_access deny nobaidu
[root@server2 ~]# systemctl reload squid
```

再用curl通过代理访问百度:

```sh
[root@server3 ~]# curl -x 10.1.1.1:3128 baidu.com
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html><head>
<meta type="copyright" content="Copyright (C) 1996-2016 The Squid Software Foundation and contributors">
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>ERROR: The requested URL could not be retrieved</title>
```

出现squid页面提示禁止访问.



## 上层代理

可以在squid中设置上层代理,来将网络访问分流.设置上层代理服务器参数如下:

- **cache_peer**

  用法:`cache_peer [上层代理主机] [代理角色] [代理端口] [icp端口] [额外参数]`

  - 上层代理主机名: 例如192.168.1.101.

  - 代理角色: 角色一般为上层(parent).还有临近(sibling)协同运行角色.

  - 代理端口: 上层代理的端口,默认是3128.

  - icp端口: 默认是3130.

  - 额外参数: 针对上层代理的行为设置,主要选项有:
    - proxy-only: 数据不缓存到本地.
    - weight: 权重设置,有多台上层代理时用更高权重表示优先选择.
    - no-query: 可以不需要发送icp数据包.
    - no-digest: 不向附近主机要求建立digest记录表格.
    - no-netdb-exchange: 不向附近代理主机发送IMCP数据包.

- **cache_peer_domain**

  用法:`cache_peer_domain [上层代理主机名] [请求的域名]`

  设置请求用上层代理服务器访问的域名.

- **cache_peer_access**

  用法:`cache_peer_access [上层代理主机名] [allow|deny] [acl名称]`

  与cache_peer_domain作用类似,不过用acl来规范访问行为.

例如设置使用代理DESKTOP-QU8VM21:3213去访问谷歌:

```sh
[root@server2 ~]# vi /etc/squid/squid.conf
cache_peer DESKTOP-QU8VM21 parent 3213 3130 proxy-only
cache_peer_domain DESKTOP-QU8VM21 .google.com
[root@server2 ~]# systemctl reload squid
```



## 透明代理

可以在对外防火墙服务器nat上安装代理,在代理上启动transparent功能,最后加上80端口转3128端口的规则,那么所有通过nat转发的内网主机上网都会强制通过代理访问http.客户端的浏览器也不需要做任何额外设置.

开启透明代理功能只需要在http_port 3128后面加上transparent参数即可:

```sh
[root@server2 ~]# vi /etc/squid/squid.conf
http_port 3128 transparent
[root@server2 ~]# systemctl reload squid
```

接着增加一条防火墙转发规则:

```sh
[root@server2 ~]# iptables -t nat -A PREROUTING -i ens37 -s 10.1.1.0/24 -p tcp --dport 80 -j REDIRECT --to-ports 3128
```

至此便完成了.



## 代理认证

squid使用ncsa_auth认证模块,可以配合apache提供的htpasswd制作的密码文件作为验证依据,来对代理使用进行身份认证:

```sh
[root@server2 ~]# rpm -ql squid | grep ncsa
/usr/lib64/squid/basic_ncsa_auth
[root@server2 ~]# yum install -y httpd
[root@server2 ~]# whereis htpasswd
htpasswd: /usr/bin/htpasswd
```

设置squid.conf内容,账号密码保存在/root/squid_user.txt:

```sh
[root@server2 ~]# vi /etc/squid/squid.conf
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /root/squid_user.txt
auth_param basic children 5
#acl localnet src 10.0.0.0/8    # RFC1918 possible internal network
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /root/squid_user.txt
auth_param basic children 5
acl squid_user proxy_auth REQUIRED
http_access allow squid_user
```

通过htpasswd建立用户sqq,密码为1234:

```sh
[root@server2 ~]# htpasswd -c /root/squid_user.txt sqq
New password: 
Re-type new password: 
Adding password for user sqq
[root@server2 ~]# systemctl restart squid
```

