# vsftpd服务

## FTP协议

文件传输协议(FTP, File Transfer Protocol)主要功能是在服务器与客户端之间进行文件传输.它使用明文传输方式,所以建议只在局域网内使用.

FTP服务器提供以下功能:

- 身份验证: 默认情况下分为三种用户: 用户user, 访客guest和匿名用户anonymous.
- 日志记录: FTP可以利用syslogd来进行数据记录,存放于/var/log/下面.
- 限制用户活动目录: change root简称chroot,将用户的工作范围局限在用户主目录下.



## FTP工作流程

FTP协议使用TCP数据包协议,并建立两个连接: 命令通道与数据流通道(ftp-data).

### 主动模式

FTP默认的主动模式(Active)连接流程如下:

1. 客户端随机选取一个大于1024的端口来与服务端的21端口实现连接.实现连接后客户端可以通过这个连接来对服务器执行命令.
2. 当客户端需要传输数据时,会随机启用一个端口,并告诉服务端使用主动模式,通过此端口传输(通过21端口).
3. 服务端接收到消息后,会主动使用20端口与客户端告之的端口建立连接,传输数据.

在防火墙后面的客户端要想连接服务端下载数据时,会因为防火墙端口被拒绝而收不到回应,造成无法建立数据流通道.可以使用modprobe命令来加载ip_conntrack_ftp和ip_nat_ftp等模块,来让iptables分析目标是21端口的连接信息,由此放行数据传输端口.

### 被动模式

FTP的被动模式(Passive)连接流程如下:

1. 客户端同样使用大于1024号的随机端口与服务端21端口建立连接.
2. 当客户端要传输数据时,只发送PASV的被动式连接请求,等待服务器回应.
3. 服务端收到请求后,会启动一个随机的被动监听端口,并告知客户端使用此端口来传输数据.
4. 客户端使用大于1024号的随机端口与主机被动端口连接,传输数据.



## 用户身份

FTP提供的三种身份适用情况如下.

### 用户

此用户为服务器中存在的实体用户,以实体用户作为FTP登录者身份时,系统默认没有针对实体用户来进行限制,用户可以针对整个文件系统进行任何有权限的操作.

因此建议实体用户模式使用sftp服务来代替.

### 访客

通常见于网页空间服务,用户需要管理自己的网页空间,但不需要实体用户的权限.

在服务器的设置中,需要针对不同访客设置不同用户主目录,并且用户主目录与用户的权限设置需要符合.此外还有诸多如容量,时间,命令等需要限制.

### 匿名用户

匿名用户可以被所有人使用,所以开放匿名登录时需要限制上传和最大连接数量.



## vsftpd服务器

vsftp的全名是Very Secure FTP,它针对操作系统的程序权限(privilege)来设计.具有以下特点:

- vsftpd服务的启动者身份为一般用户,对系统的权限较低.此外vsftpd利用chroot来更换根目录,使得系统工具不会被vsftpd服务所误用.
- 任何需要具有较高执行权限的vsftpd命令均已一个特殊的上层程序控制,此上层程序享有的权限已被限制得很低,并且确保不会影响到系统.
- 绝大部分FTP命令都已被整合到vsftpd主程序中了,不需要再调用外部命令.
- 所有客户端想使用上层程序提供的较高执行权限的vsftpd命令(例如chown,login),均被视作不可信任的要求处理,必须经过身份确认.

使用yum来安装vsftpd:

```sh
[root@server2 cgi-bin]# yum install -y vsftpd
```

和vsftpd配置有关的文件有:

- /etc/vsftpd/vsftpd.conf

  vsftpd主要配置文件.以参数=设置值来设置,等号=两边不能有空格.

- /etc/pam.d/vsftpd

  vsftpd使用PAM模块时的相关配置文件,主要可设置拒绝用户文件/etc/vsftpd/ftpusers路径.

- /etc/vsftpd/ftpusers

  由PAM模块指定的拒绝用户列表记录文件,只需要一行一个记录拒绝用户账号即可.默认已经填入了系统账号.

- /etc/vsftpd/user_list

  vsftpd提供的访问控制,由userlist_enable和userlist_deny的设置控制.如userlist_deny=NO为用户白名单.

- /etc/vsftpd/chroot_list

  此文件需手动建立,作用是将用户chroot建立在用户主目录下.由chroot_list_enable和chroot_list_file设置启用.

- /var/ftp

  默认匿名用户登录的根目录.与ftp账号的用户主目录有关.



## vsftpd配置

配置文件/etc/vsftpd/vsftpd.conf中一些常用的选项如下:

- connect_from_port_20=YES

  主动模式使用的数据传输端口.

- listen_port=21

  命令通道端口.如果使用非正规端口号,需要同时开启listen=YES来以stand alone的方式启动.

- listen=YES

  默认为NO,设置为YES表示以stand alone的方式来启动.

- pasv_enable=YES

  设置传输使用被动模式.

- write_enable=YES

  允许用户上传.

- max_clients=0

  设置同时最多连接客户端数量.

- max_per_ip=0

  设置同一个IP同时最多允许的连接数.

- pasv_min_port=0, pasv_max_port=0

  设置被动模式下可使用的传输端口范围.

- guest_enable=YES

  设置为YES时,任何实体账号都会被假设成访客.访客在vsftpd中默认取得ftp用户的权限.

- local_enable=YES

  设置为YES时,实体账号才可以登录vsftpd服务器.

- chroot_local_user=YES

  设置是否将用户限制在主目录内.

- anonymous_enable=YES

  是否允许匿名用户登录,默认为YES.



## vsftpd测试

vsfptd启动使用systemctl控制:

```sh
[root@server2 cgi-bin]# systemctl enable --now vsftpd
```

新建一个账号,并测试ftp连接:

```sh
[root@server2 cgi-bin]# useradd -p user02 user02
[root@server1 ~]# ftp 192.168.2.254
Connected to 192.168.2.254 (192.168.2.254).
220 (vsFTPd 3.0.2)
Name (192.168.2.254:root): user02
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> pwd
257 "/home/user02"
```

将用户user02添加进/etc/vsftpd/ftpusers后再次测试:

```sh
[root@server2 ~]# mkdir /etc/vsftpd/
[root@server2 ~]# vi /etc/vsftpd/ftpusers
user02
[root@server2 home]# systemctl reload sshd
[root@server1 ~]# ftp 192.168.2.254
Connected to 192.168.2.254 (192.168.2.254).
220 (vsFTPd 3.0.2)
Name (192.168.2.254:root): user02
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
```



