# NIS服务

## NIS服务器

NIS是Network Information Services的缩写,也称为Sun Yellow Pages.NIS服务器主要提供用户的账号,密码,主目录,UID等信息给客户端主机用来查询之用.

NIS服务器使用RPC协议传输数据,并且可以使用Master/Slave构架.整个运行流程如下:

- NIS Master先将账号密码相关文件制作成数据库格式.
- NIS Slave连接Master同步数据.
- 如果账号密码有变动会重新制作数据库并同步.
- 当NIS Client有登录需求时会先查询本机passwd,shadow等文件.
- 若找不到相关账号数据,则向整个NIS网络的主机广播查询.
- 由NIS Server响应Client的请求.



## 服务端配置

NIS服务器同时也可以被当作客户端,它依赖的软件有下面一些:

- yp-tools: 提供NIS相关查询命令.
- ypbind: 提供NIS Client端设置.
- ypserv: 提供NIS Server端设置.
- rpcbind: RPC服务主程序.

默认情况下需要手动安装:

```sh
[root@server2 ~]# yum install -y yp-tools
[root@server2 ~]# yum install -y ypserv
```

和NIS服务器配置有关的文件:

- /etc/ypserv.conf: NIS服务器主要配置文件,可以定义NIS客户端是否有可登录权限.
- /etc/hosts: 由于NIS Server/Client会用到网络主机名,因此要在hosts中有记录.
- /etc/sysconfig/network: 可以在这个文件内指定NIS的网络.
- /var/yp/Makefile: 与建立数据库有关的操作控制文件.

首先需要设置NIS的域名,NIS通过域名来分辨不同的账号密码数据,因此在服务端与客户端都要指定相同的NIS域名.

例如设置域名为vserver,启动端口固定在1011:

```sh
[root@server2 ~]# echo "NISDOMAIN=vserver" >> /etc/sysconfig/network
[root@server2 ~]# echo 'YPSERV_ARGS="-p 1011"' >> /etc/sysconfig/network
[root@server2 ~]# cat /etc/sysconfig/network
# Created by anaconda
NISDOMAIN=vserver
YPSERV_ARGS="-p 1011"
```

然后NIS的服务端配置文件保持默认值即可:

```sh
[root@server2 ~]# cat /etc/ypserv.conf 
files: 30
xfr_check_port: yes

# Host                     : Domain  : Map              : Security 
192.168.2.0/255.255.255.0  : vserver : *   				: none
```

其中主要参数设置如下:

- files: 设置多少个数据库被读入内存当中.
- xfr_check_port: 和主从构架有关,将同步更新的数据库对比所使用的端口设置成小于1024.
- Host: 主机名或IP,例如192.168.2.0/255.255.255.0.
- Domain: NIS域名,例如vserver.
- Map: 可用数据库名称.
- Security: 安全限制,包括none(无限制),port(仅用<1024号端口),deny(拒绝).

在设置客户端查询的权限部分,可以使用星号*来代表接受所有数据.

最后修改本地hosts,将服务端和客户端的主机名与IP对应记录写入:

```sh
[root@server2 ~]# echo "192.168.2.254  server2" >> /etc/hosts
[root@server2 ~]# echo "192.168.2.234  server1" >> /etc/hosts
```

为了将yppasswdd启动在固定端口1012还需要修改下配置:

```sh
[root@server2 ~]# sed -i 's/YPPASSWDD_ARGS=/YPPASSWDD_ARGS="--port 1012"/g' /etc/sysconfig/yppasswdd
```



## 服务端管理

依次启动ypserv和yppasswdd:

```sh
[root@server2 ~]# systemctl start ypserv
[root@server2 ~]# systemctl start yppasswdd
[root@server2 ~]# systemctl enable ypserv
[root@server2 ~]# systemctl enable yppasswdd
[root@server2 ~]# rpcinfo -p
   program vers proto   port  service
    100004    2   udp   1011  ypserv
    100004    1   udp   1011  ypserv
    100004    2   tcp   1011  ypserv
    100004    1   tcp   1011  ypserv
    100009    1   udp   1012  yppasswdd
```

在服务端建立一个测试用户并转成数据库:

```sh
[root@server2 ~]# useradd -p password nisuser
[root@server2 ~]# /usr/lib64/yp/ypinit -m

At this point, we have to construct a list of the hosts which will run NIS
servers.  server2 is in the list of NIS server hosts.  Please continue to add
the names for the other hosts, one per line.  When you are done with the
list, type a <control D>.
        next host to add:  server2
        next host to add:  
The current list of NIS servers looks like this:

server2

Is this correct?  [y/n: y]  y
We need a few minutes to build the databases...
Building /var/yp/vserver/ypservers...
Running /var/yp/Makefile...
gmake[1]: Entering directory `/var/yp/vserver'
Updating passwd.byname...
Updating passwd.byuid...
Updating group.byname...
Updating group.bygid...
Updating hosts.byname...
Updating hosts.byaddr...
Updating rpc.byname...
Updating rpc.bynumber...
Updating services.byname...
Updating services.byservicename...
Updating netid.byname...
Updating protocols.bynumber...
Updating protocols.byname...
Updating mail.aliases...
gmake[1]: Leaving directory `/var/yp/vserver'

server2 has been set up as a NIS master server.

Now you can run ypinit -s server2 on all slave server.
```

如果用户密码发生过变化,需要运行上面命令重新制作数据库.



## 客户端设置

客户端也需要安装ypbind和yp-tools软件,yp-tools中已经包含了ypbind软件:

```sh
[root@server2 ~]# yum install -y yp-tools
```

客户端同服务器端一样先修改/etc/sysconfig/network文件:

```sh
[root@server2 ~]# echo "NISDOMAIN=vserver" >> /etc/sysconfig/network
```

修改客户端主要配置文件/etc/yp.conf文件,将服务端地址写进去:

```sh
[root@server1 ~]# echo "domain vserver server 192.168.2.254" >> /etc/yp.conf 
```

修改/etc/nsswitch.conf和/etc/sysconfig/authconfig文件:

```sh
[root@server1 ~]# vi /etc/nsswitch.conf
passwd:     files nis
shadow:     files nis
group:      files nis
hosts:      files nis dns
[root@server1 ~]# vi /etc/sysconfig/authconfig
USENIS=yes
```



## 客户端管理

启动服务,并用id查询存在服务端的账号nisuser:

```sh
[root@server1 ~]# systemctl start ypbind
[root@server1 ~]# id nisuser
uid=1000(user1) gid=1001(user1) groups=1001(user1),1000(usergroup)
```

利用yptest验证数据库:

```sh
[root@server1 ~]# yptest
Test 1: domainname
Configured domainname is "vserver"

Test 2: ypbind
Used NIS server: server2

Test 3: yp_match
WARNING: No such key in map (Map passwd.byname, key nobody)

Test 9: yp_all
nisuser nisuser:password:1000:1000::/home/nisuser:/bin/bash
1 tests failed
```

在Test 3报错说没有该数据库,可以忽略.

使用ypwhich检查数据库数量:

```sh
[root@server1 ~]# ypwhich -x
Use "ethers"    for map "ethers.byname"
Use "aliases"   for map "mail.aliases"
Use "services"  for map "services.byname"
```

从结果可以很清楚看到相关文件,这些数据库文件放置在服务端/var/yp/vserver/*中

使用ypcat能直接读取数据库内容:

```sh
[root@server2 ~]# ypcat -h 192.168.2.254 passwd.byname
nisuser:password:1000:1000::/home/nisuser:/bin/bash
```



## 客户端操作

启动好ypbind服务后,服务端和客户端的账号已经同步了.可以在客户端直接切换到nisuser用户:

```sh
[root@server1 ~]# su - nisuser
Last login: Sat Oct  2 01:18:14 CST 2021 on pts/0
su: warning: cannot change directory to /home/nisuser: No such file or directory
-bash-4.2$ id
uid=1000(user1) gid=1000(usergroup) groups=1000(usergroup) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

提示说没有用户主目录,可以使用yppasswd来修改这个账号的密码:

```sh
-bash-4.2$ yppasswd
Changing NIS account information for nisuser on server2.
Please enter old password:
Sorry.
```

此外还有ypchfn修改个人信息和ypchsh修改使用shell命令可用.

