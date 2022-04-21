# SAMBA服务

## SAMBA简介

Linux上面使用NFS服务器来共享文件,Windows上使用的是CIFS(Common Internet File System)服务(网上邻居)来共享文件.这两种服务都不能跨系统共享文件,SAMBA就是为了解决这一问题而存在.

SAMBA(Server Message Block)服务的主要功能:

- 共享文件与打印机服务.
- 提供用户登录SAMBA主机时的身份认证.
- 可以进行Windows网络上的主机名解析(NetBIOS Name).
- 可以进行设备的共享(例如CD-ROM).



## NetBIOS协议

SAMBA构架在NetBIOS(Network Basic Input/Output System)通信协议上.

NetBIOS是无法跨路由(Router/Gateway)使用,只能用于局域网.数据传输时使用NetBIOS Name而不是IP来识别对方.后来发展出NetBIOS over TCP/IP技术突破了这一限制.

Windows网上邻居也使用NetBIOS,工作流程如下:

1. 要想在网上邻居访问某台主机时,必须加入对方主机的工作组(Workgroup),并且本机要设置一个主机名.这个主机名是构架在NetBIOS协议上的,也叫NetBIOS Name.在同一个工作组,NetBIOS Name必须唯一.
2. 找到对方主机后,能否登录或使用主机资源,还要看对方主机上的权限配置.

SAMBA通过两个服务来控制上面两步骤:

- **nmdb**: 用来管理工作组,NetBIOS Name的解析,主要利用UDP协议开启137和138端口来做名称解析.
- **smbd**: 用来管理SAMBA主机共享的目录,文件与打印机等.主要利用TCP协议开机139和445端口来传输数据.



## 连接模式

局域网内最常见的连接模式有两种: 对等模式和主控模式.这两种模式SAMBA都可以实现.

### 对等模式

对等模式(Workgroup Model, Peer/Peer)指局域网内两台主机的地位相等,均有各自的账号,可以独立工作.当两台主机之间需要共享数据时,需要使用对方主机内记录的账号来访问对方主机.

这种模式比较适合小型网络内数据共享.

### 主控模式

主控模式(Domain Model)是将所有账号密码都放置在一台主控计算机(PDC, Primary Domain Controller)上面,任何人使用计算机时需要通过PDC服务器认证后,给与对应权限.

这种模式适合企业构架,只需维护好PDC上面的账号资源,再分配给人员使用.



## SAMBA安装

SAMBA所需的基本组件有下面三个:

- **samba**: 提供SMB服务器所需各项服务程序(smbd和nmbd),其他与SAMBA相关的logrotate配置文件及开机默认选项文件等.
- **samba-client**: 提供SAMBA客户端所需工具命令,例如mout.cifs, smbtree等.
- **samba-common**: 提供客户端和服务端通用数据,例如smb.conf,testparm等.
- **cifs-utils**: 提供挂载cifs支持等.

可以使用yum来安装:

```sh
[root@server2 ~]# yum install -y samba
[root@server2 ~]# yum install -y samba-client
[root@server2 ~]# yum -y install cifs-utils
```

samba包内已包含samba-common.



## 配置文件路径

和SAMBA有关的配置文件有下面这些:

- **/etc/samba/smb.conf**

  SAMBA的主要配置文件.主要内容有全局设置和共享目录相关设置.

- **/etc/samba/lmhosts**

  早期NetBIOS Name所需额外设置,用来设置NetBIOS Name对应的IP地址.现在SAMBA默认使用hostname作为NetBIOS Name,因此可以不用设置.

- **/etc/sysconfig/samba**

  设置启动smbd, nmbd时,想要加入的额外服务参数.

- **/etc/samba/smbusers**

  可以用来设置Windows与Linux中管理员与匿名用户名称之间的对应关系.例如administrator对应root用户.

- **/var/lib/samba/private/{passdb.tdb,secrets.tdb}**

  管理SAMBA账号与密码的数据库文件.

- **/usr/share/doc/samba-***

  这个目录包含SAMBA的所有相关技术手册.



## SAMBA命令

和SAMBA有关的命令如下:

| 命令       | 说明                                             |
| ---------- | ------------------------------------------------ |
| smbd       | 服务器权限管理                                   |
| nmbd       | NetBIOS Name查询                                 |
| tdbdump    | 查询SAMBA数据库TDB(Trivial-DataBase)的内容       |
| tdbtool    | 直接修改数据库中账号和密码参数,需要安装tdb-tools |
| smbstatus  | 服务端列出当前SAMBA的连接状态                    |
| smbpasswd  | 早期修改SAMBA用户密码的命令,已被pdbedit取代      |
| pdbedit    | 修改SAMBA用户密码命令                            |
| testparm   | 服务端用来检验配置文件语法错误工具               |
| mount.cifs | 客户端用来挂载远程共享目录的命令                 |
| smbclient  | 客户端用来搜索其他主机共享目录命令               |
| nmblookup  | 客户端查询NetBIOS Name的命令                     |
| smbtree    | 客户端用来查询工作组与计算机名称树形目录分布图   |



## 服务器配置流程

SAMBA服务器配置流程如下:

1. 服务器全局设置: 在smb.conf中设置好工作组,NetBIOS主机名,密码使用状态等;
2. 规划共享目录参数: 在smb.conf中设置好预计要共享的目录或设备以及可供使用的账号数据;
3. 奖励所需文件系统: 在系统中建立好共享出去的目录或设备,设置好权限参数;
4. 建立SAMBA用账号: 在系统中建立好共享时使用的账号,用pdbedit建立SAMBA密码;
5. 启动服务: 启动smbd和nmbd服务.



## 配置文件参数

配置文件smb.conf中全局配置[global]部分主要设置参数如下:

- workgroup: 工作组名称
- netbios name: 主机NetBIOS名称
- server string: 主机的简易说明
- display charset: 终端所使用的编码
- unix charset: 在Linux服务器上使用的编码
- dos charset: 在Windows客户端的编码
- log file: 日志文件名
- max log size: 日志大小限制,达到最大值会被rotate掉
- security: 有三种设置share(不用密码,已废弃), user(用SAMBA服务器认证), domain(指定认证SAMBA服务器)
- encrypt passwords: Yes(密码加密)
- passdb backend: 数据库格式,默认为tdbsam

配置文件smb.conf中共享资源[homes]部分相关设置参数如下:

- comment: 目录的说明
- path: 共享的实际目录
- browseable: 是否让所有用户可见
- writable: 是否可写入
- create mode/directory mode: 设置权限
- writelist: 设置可使用此资源的用户或用户组(@开头)

配置文件smb.conf中可用的变量功能:

- %S: 取代当前的设置项目值.
- %m: 代表客户端的NetBIOS主机名.
- %M: 代表客户端的Hostname.
- %I: 代表客户端的IP
- %L: 代表SAMBA主机的NetBIOS主机名.
- %h: 代表SAMBA主机的Hostname.
- %H: 代表用户的主目录.
- %U: 代表当前登录的用户名.
- %g: 代表登录的用户组名.
- %T: 代表当前的日期和时间.

另外在配置文件内除了#表注释,还可以使用分号;表注释.



## 服务端配置示例

下面列出一些常见的使用场景和对应smb.conf配置文件内容.

### 无密码共享

假设SMB服务器(192.168.2.254)要配置成如下参数:

- 局域网内计算机工作组为WORKGROUP
- 服务器的NetBIOS为smbserver2
- 用户认证等级设置为无密码share
- 取消原本有共享的[homes]目录
- 仅共享/tmp目录,取名也为temp
- Linux服务器的编码格式设置为utf8
- Windows客户端的编码格式设置为GB2312

整个配置文件如下:

```sh
[root@server2 ~]# cat /etc/samba/smb.conf
[global]
        workgroup = WORKGROUP
        netbios name = smbserver2
        server string = This is OPEN SAMBA server
        security = user
        map to guest = Bad User

        unix charset = utf8
        display charset = utf8
        dos charset = cp936

        log file = /var/log/samba/log.%m
        max log size = 50

        passdb backend = tdbsam

        load printers = no

[temp]
        comment = TEMP DIR
        path = /tmp
        writable = yes
        browseable = yes
        guest ok = yes
```

可以使用testparm检查下smb.conf配置文件语法设置:

```sh
[root@server2 ~]# testparm
Load smb config files from /etc/samba/smb.conf
Unknown parameter encountered: "display charset"
Ignoring unknown parameter "display charset"
Loaded services file OK.
Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions
```

### 有密码共享

假设要设置基于用户密码访问的SAMBA服务器:

- 用户认证等级设置为user
- 用户密码文件使用TDB数据库格式,存放于/var/lib/samba/private/内
- 密码需要加密
- 每隔SAMBA的用户拥有自己的用户主目录
- 设置一个用户smbuser登录密码为1111,SAMBA密码为4444
- 共享/home/sharedir目录,共享资源名称为sharedir
- 用户组smbuser对/home/sharedir具有写入权限

先建立相关用户和目录:

```sh
[root@server2 ~]# useradd smbuser -p 1111
[root@server2 ~]# mkdir /home/sharedir
[root@server2 ~]# chgrp smbuser /home/sharedir/
[root@server2 ~]# chmod 2770 /home/sharedir/
[root@server2 ~]# ll -d /home/sharedir/
drwxrws---. 2 root smbuser 6 Oct 11 23:25 /home/sharedir/
```

需要使用pdbedit来处理SAMBA用户,命令常用选项如下:

| 参数 | 说明                                      |
| ---- | ----------------------------------------- |
| -L   | 列出数据库中对应账号与UID等信息           |
| -v   | 搭配-L参数显示更多详细信息                |
| -w   | 搭配-L参数显示smbpasswd格式数据           |
| -a   | 设置一个可使用SAMBA的用户账号             |
| -r   | 修改一个账号相关信息                      |
| -x   | 删除一个可使用SAMBA的用户账号             |
| -m   | 后接机器的代码(machine account),与PDC有关 |

例如将用户smbuser加入到SAMBA的数据库:

```sh
[root@server2 ~]# pdbedit -a -u smbuser
new password:
retype new password:
Unix username:        smbuser
NT username:          
Account Flags:        [U          ]
User SID:             S-1-5-21-4293618527-3344052101-3349617220-1000
Primary Group SID:    S-1-5-21-4293618527-3344052101-3349617220-513
Full Name:            
Home Directory:       \\smbserver2\smbuser
HomeDir Drive:        
Logon Script:         
Profile Path:         \\smbserver2\smbuser\profile
Domain:               SMBSERVER2
Account desc:         
Workstations:         
Munged dial:          
Logon time:           0
Logoff time:          Wed, 06 Feb 2036 23:06:39 CST
Kickoff time:         Wed, 06 Feb 2036 23:06:39 CST
Password last set:    Mon, 11 Oct 2021 23:36:23 CST
Password can change:  Mon, 11 Oct 2021 23:36:23 CST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
[root@server2 ~]# pdbedit -L
smbuser:1001:
```

修改密码与删除用户示例:

```sh
[root@server2 ~]# smbpasswd smbuser
New SMB password:
Retype new SMB password:
[root@server2 ~]# pdbedit -x -u smbuser
```

共享用户目录需要在SELinunx中放行:

```sh
[root@server2 ~]# setsebool -P samba_enable_home_dirs on
```

局域网内可完全放开SAMBA对文件系统的读写:

```sh
[root@server2 ~]# setsebool -P samba_export_all_rw 1
```

否则需要修改目录的类型:

```sh
[root@server2 ~]# chcon -t samba_share_t /home/sharedir
```

smb.conf配置文件如下:

```sh
[root@server2 ~]# cat /etc/samba/smb.conf
[global]
        workgroup = WORKGROUP
        netbios name = smbserver2
        server string = This is OPEN SAMBA server
        security = user

        unix charset = utf8
        display charset = utf8
        dos charset = cp936

        log file = /var/log/samba/log.%m
        max log size = 50

        passdb backend = tdbsam

        load printers = no

[homes]
        comment = Home DIR
        writable = yes
        browseable = no
        create mode = 0664
        directory mode = 0775
[sharedir]
        comment = sharedir(server2)
        path = /home/sharedir
        browseable = yes
        writable = yes
        write list = @smbuser
```

### 服务端启动

设置好配置文件和相关账号权限后,就可以启动SAMBA服务了(nmb有可能被SELinux阻拦):

```sh
[root@server2 ~]# systemctl enable --now smb
[root@server2 ~]# systemctl enable --now nmb
[root@server2 ~]# netstat -ntulp | grep mbd
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address     Foreign Address  State       PID/Program name 
tcp   0      0      0.0.0.0:445       0.0.0.0:*        LISTEN      80761/smbd
tcp   0      0      0.0.0.0:139       0.0.0.0:*        LISTEN      80761/smbd
udp   0      0      0.0.0.0:137       0.0.0.0:*                    80786/nmbd
udp   0      0      0.0.0.0:138       0.0.0.0:*                    80786/nmbd
```

### 服务状态查询

在启动后和运行中可以使用smbstatus命令来查看SAMBA服务的状态:

```sh
[root@server2 samba]# smbstatus

Samba version 4.10.16
PID   Username Group    Machine                                   Protocol Version  Encryption           Signing              
-----------------------------------------------------------------------------------------
85520 smbuser  smbuser  192.168.2.101 (ipv4:192.168.2.101:62317)  SMB3_11           -                    partial(AES-128-CMAC)
84217 smbuser  smbuser  192.168.2.234 (ipv4:192.168.2.234:41716)  SMB3_02           -                    partial(AES-128-CMAC)

Service      pid     Machine       Connected at                     Encryption   Signing 
-----------------------------------------------------------------------------------------
smbuser      84217   192.168.2.234 Mon Oct 11 11:48:11 PM 2021 CST  -            -       
IPC$         85520   192.168.2.101 Tue Oct 12 12:10:30 AM 2021 CST  -            -       
sharedir     84217   192.168.2.234 Tue Oct 12 12:02:55 AM 2021 CST  -            -       
IPC$         84217   192.168.2.234 Mon Oct 11 11:48:11 PM 2021 CST  -            -       
sharedir     85520   192.168.2.101 Tue Oct 12 12:40:25 AM 2021 CST  -            -       

Locked files:
Pid    User(ID)  DenyMode   Access    R/W     Oplock  SharePath   Name   Time
-----------------------------------------------------------------------------------------
85520  1001      DENY_NONE  0x100081  RDONLY  NONE    /home/sharedir   . Tue Oct 12 00:40:32 2021
85520  1001      DENY_NONE  0x100081  RDONLY  NONE    /home/sharedir   . Tue Oct 12 01:06:15 2021
```



## 客户端操作

### 查询

可以通过nmblookup命令来查询NetBIOS Name与IP对应信息,例如查询I5-103对应的IP地址:

```sh
[root@server1 sss]# nmblookup -S I5-103
192.168.2.103 I5-103<00>
Looking up status of 192.168.2.103
        I5-103          <00> -         B <ACTIVE> 
        WORKGROUP       <00> - <GROUP> B <ACTIVE> 
        I5-103          <20> -         B <ACTIVE> 
        WORKGROUP       <1e> - <GROUP> B <ACTIVE> 
        WORKGROUP       <1d> -         B <ACTIVE> 
        ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE> 

        MAC Address = 30-85-A9-4C-F1-86
```

可以使用smbtree用树状图列出SAMBA服务器情况:

```sh
[root@server1 sss]# smbtree
Enter SAMBA\root's password: 
WORKGROUP
        \\X9-102         
        \\SMBSERVER2                    This is OPEN SAMBA server
                \\SMBSERVER2\IPC$               IPC Service (This is OPEN SAMBA server)
                \\SMBSERVER2\sharedir           sharedir(server2)
        \\I5-103         
        \\DESKTOP-QU8VM21
```

### 操作

可以通过smbclient程序来查询SAMBA服务是否正常:

```sh
[root@server1 ~]# smbclient -L //192.168.2.254
Enter SAMBA\root's password: 
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        sharedir        Disk      sharedir(server2)
        IPC$            IPC       IPC Service (This is OPEN SAMBA server)
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

        Server               Comment
        ---------            -------
        Workgroup            Master
        ---------            -------
        WORKGROUP     
```

指定使用smbuser账号连接smb服务器192.168.2.254:

```sh
[root@server1 sss]# smbclient //192.168.2.254/sharedir -U smbuser
Enter SAMBA\smbuser's password: 
Try "help" to get a list of possible commands.
smb: \> 
```

### 挂载

可以直接使用mount命令来挂载:

```sh
[root@server1 ~]# mount -t cifs //192.168.2.254/temp /root/111/
Password for root@//192.168.2.254/temp:  
[root@server1 ~]# mount -t cifs //192.168.2.254/smbuser /root/myhome -o username=smbuser
Password for smbuser@//192.168.2.254/smbuser:  ****
[root@server1 ~]# mount -t cifs //192.168.2.254/sharedir /root/sss -o username=smbuser
[root@server1 ~]# df
Filesystem              1K-blocks    Used Available Use% Mounted on
//192.168.2.254/temp      1020580   33092    987488   4% /root/111
//192.168.2.254/smbuser   5109760   33084   5076676   1% /root/myhome
//192.168.2.254/sharedir   5109760   33120   5076640   1% /root/sss
```

针对Windows下文件系统中使用中文字符,可以指定mount挂载参数:

```sh
[root@server1 ~]# mount -t ntfs -o iocharset=utf8,codepage=936 //192.168.2.101/temp 222
```

Windows下也可以直接使用\\\192.168.2.254\来访问SAMBA服务器(不支持匿名模式)



## PDC服务器搭建

SAMBA PDC的作用很简单,就是让PDC成为整个局域网的域管理员(Domain Controller),然后把主机加入到这个域中.域内的用户账户和用户数据都由PDC来管理.

PDC服务器的搭建步骤如下:

### 设置本地解析

将NetBIOS Name与IP的对应填入/etc/samba/lmosts文件中:

```sh
[root@server2 ~]# echo "192.168.2.254 smbserver2" >> /etc/samba/lmhosts
[root@server2 ~]# echo "192.168.2.234 smblinux" >> /etc/samba/lmhosts
[root@server2 ~]# echo "192.168.2.101 DESKTOP-QU8VM21" >> /etc/samba/lmhosts
```

由于NetBIOS Name和Hostname不一样,因此还要修改/etc/hosts文件:

```sh
[root@server2 ~]# sed -i "s/192.168.2.254  server2/192.168.2.254  server2 smbserver2/g" /etc/hosts
[root@server2 ~]# sed -i "s/192.168.2.234  server1/192.168.2.234  server1 smblinux/g" /etc/hosts
[root@server2 ~]# echo "192.168.2.101 DESKTOP-QU8VM21" >> /etc/hosts
```

### 配置smb.conf

需要让PDC客户端登录时可以取得用户主目录.smb.conf文件内容如下:

```sh
[root@server2 ~]# cat /etc/samba/smb.conf
[global]
        workgroup = WORKGROUP
        netbios name = smbserver2
        server string = This is OPEN SAMBA server
        security = user
        unix charset = utf8
        dos charset = cp936
        log file = /var/log/samba/log.%m
        max log size = 50
        passdb backend = tdbsam
        load printers = no

        preferred master = yes
        domain master = yes
        local master = yes
        wins support = yes
        os level = 100
        domain logons = yes
        logon drive = K:
        logon script = startup.bat
        time server = yes
        admin users = root
        logon path = \\%N\%U\profile
        logon home = \\%N\%U
[netlogon]
        comment = Network Logon
        path = /winhome/netlogon
        writable = no
        write list = root
        follow symlinks = yes
        guest ok = yes

[homes]
        comment = Home DIR
        writable = yes
        browseable = no
        create mode = 0664
        directory mode = 0775
[sharedir]
        comment = sharedir(server2)
        path = /home/sharedir
        browseable = yes
        writable = yes
        write list = @smbuser
[root@server2 ~]# testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
idmap range not specified for domain '*'
ERROR: Invalid idmap range for domain *!

Server role: ROLE_DOMAIN_PDC
```

和PDC有关的参数:

- time server: 设置SAMBA与Windows主机的时间同步.
- logon script: 当用户通过Windows客户端登录后运行的批处理文件.文件需要放置到指定目录内.
- logon drive: 用户主目录挂载到的分区盘符.
- admin users: 指定SAMBA PDC的管理员身份.
- [netlogon] : 指定利用网络登录后使用的目录资源.
- logon path: 用户登录后,环境配置文件所在目录.
- logon home: 用户的主目录,默认放置到与Linux的用户主目录相同位置. 

配置文件没问题后重启smb和nmb服务:

```sh
[root@server2 ~]# systemctl restart smb
[root@server2 ~]# systemctl restart nmb
```

### 建立对应目录

预计将所有的PDC数据放到/winhome内:

```sh
[root@server2 ~]# mkdir -p /winhome/netlogon
```

建立用户脚本文件,主要设置时间同步和用户目录挂载:

```sh
[root@server2 ~]# vi /winhome/netlogon/startup.bat
net time \\smbserver2 /set /yes
net use K: /home
[root@server2 ~]# unix2dos /winhome/netlogon/startup.bat 
unix2dos: converting file /winhome/netlogon/startup.bat to DOS format ...
```

### 建立对应用户

建立一个smbwin的用户并指定用户目录:

```sh
[root@server2 ~]# useradd -d /winhome/smbwin -p 3333 smbwin
[root@server2 ~]# mkdir /winhome/smbwin/profile
[root@server2 ~]# chown -R smbwin:smbwin /winhome/smbwin/profile
```

将用户加入到SAMBA数据库,设置密码为2222:

```sh
[root@server2 ~]# smbpasswd -a root
New SMB password:
Retype new SMB password:
Added user root.
[root@server2 ~]# smbpasswd -a smbwin
New SMB password:
Retype new SMB password:
Added user smbwin.
```

### 建立机器码账号

由于PDC会针对Windows客户端的主机名(NetBIOS Name)进行主机账号检查,所以腰围客户端的主机名进行账号的设置.主机账号在主机名后加个$即可:

```sh
[root@server2 ~]# useradd -M -s /sbin/nologin -d /dev/null DESKTOP-QU8VM21$
```

账号被设置为不能登录,将其加入到SAMBA数据库中:

```sh
[root@server2 ~]# smbpasswd -a -m DESKTOP-QU8VM21$
Added user DESKTOP-QU8VM21$.
```

### 修改安全性配置

由于/winhome为自建目录,需要修改目录的SELinux类型为samba_share_t:

```sh
[root@server2 ~]# chcon -R -t samba_share_t /winhome
```



## PDC客户端操作

### 修改注册表

在Win10中需要修改注册表才支持通过SAMBA登录.修改位置为:

`计算机\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters`

添加两个DWORD值:DomainCompatibilityMode=1和DNSNameResolutionRequired=0

### 设置主机名与域名

在"计算机属性--高级系统设置--计算机名"页面中,点击"网络 ID"按钮进入加入域或工作组页面.

在描述网络选项中,选择办公网络,单击下一步.

选择使用域的网络,单击下一步两次.

在键入域账户的用户名,密码和域名页面,填写用户名root,密码为记录在SAMBA中的root密码.单击下一步.

在填写计算机名和域名页面,填写计算机域名WORKGROUP.单击下一步.

在弹出域用户名和密码页面,填入root用户和密码,域名依然是WORKGROUP.单击确定按钮.

在是否要启用域用户账户页面,选择目前不添加,由PDC来管控.单击下一步.

单击完成后重启计算机.

### 用户登录

重启之后便可选择域名,以及使用PDC记录的用户名smbwin来登录,本机账号不受影响.

用户登陆后,会将K盘挂载成用户主目录,可以在里头工作,数据将直接保存在/winhome/smbwin中.

用户注销之后,在桌面上进行的各项个人化设置会被移动到/winhome/smbwin/profile.v6目录中:

```sh
[root@server2 smbwin]# ll profile.V6/Desktop/
total 16
-rw-rw-r--+ 1 smbwin smbwin   282 Oct 12 03:38 desktop.ini
-rw-rw----+ 1 smbwin smbwin 11613 Oct 12 03:44 新建 Microsoft Word 文档.docx
```

