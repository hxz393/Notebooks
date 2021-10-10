# SSH服务

## SSH服务器

SSH是安全的壳程序协议(Secure Shell Protocol),它可以通过数据包加密技术将等待传输的数据包加密后在网上传输.默认状态下,SSH协议提供服务端和客户端,还带有一个类似FTP服务的Sftp-Server,它们都使用22端口.

目前常见网络数据包加密技术通过非对称密钥系统来处理,它通过两把不一样的公钥和私钥来进行数据加密与解密:

- 公钥(Public Key): 提供给远程主机进行数据加密的行为.
- 私钥(Private Key): 在本地端使用私钥来解密通过公钥加密的数据.

SSH服务端与客户端的连接步骤如下:

1. 服务端第一次启动sshd时自动产生公钥和私钥,存放于/etc/ssh/ssh_host_*;
2. 客户端向服务端请求连接;
3. 服务端给客户端发送服务端的公钥;
4. 客户端将公钥记录在~/.ssh/known_hosts中,并发送自己的公钥给服务端;
5. 服务端和客户端建立连接,并各自通过对方公钥加密数据后传输.

在第4步客户端生成的公钥和私钥是一次性的,每次建立连接会重新生成.而服务器的公钥不变,因此客户端也会比较每次连接收到的公钥是否和known_hosts中的记录一致.

可以直接使用systemctl来控制ssh服务器:

```sh
[root@server1 ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2021-10-09 03:22:48 CST; 19h ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 1242 (sshd)
   CGroup: /system.slice/sshd.service
           └─1242 /usr/sbin/sshd -D
```



## SSH客户端

默认ssh客户端使用ssh命令来运行,命令格式: `ssh [-f] [-o 参数项目] [-p 端口号] [账号@]IP [命令]`

选项说明:

| 选项 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| -f   | 配合后面的命令,不登陆主机直接发送命令过去执行.               |
| -o   | 参数项目,常用的有:<br />ConnectTimeout=秒数: 控制超时时间<br />StrictHostKeyChecking=yes\|no\|ask: 加入新公钥时的操作,默认ask询问 |
| -p   | 指定非标准ssh连接端口.                                       |

如果连接时不指定账号,使用的是本地端账号尝试登录.例如以user1账号登录远程10.1.1.2:10022服务器:

```sh
[root@server1 ~]# ssh -p 10022 user1@10.1.1.2
ssh: connect to host 10.1.1.2 port 10022: Connection refused
```

例如通过ssh运行指定命令:

```sh
[root@server1 ~]# ssh -f 127.0.0.1 find / &> find.log
The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
```

假设服务器重新安装过系统,密钥更新后会造成客户端认证失败.客户端清空掉known_hosts文件内容即可:

```sh
[root@server1 ~]# rm -f /etc/ssh/ssh_host*
[root@server1 ~]# systemctl restart sshd
[root@server1 ~]# ssh 127.0.0.1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:jNdAaJuldv+NT40ULeK7lEre4ro8OE/teafAM/bBibg.
Please contact your system administrator.
Add correct host key in /root/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /root/.ssh/known_hosts:2
ECDSA host key for 127.0.0.1 has changed and you have requested strict checking.
Host key verification failed.
[root@server1 ~]# echo "" > /root/.ssh/known_hosts 
[root@server1 ~]# ssh 127.0.0.1
The authenticity of host '127.0.0.1 (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:jNdAaJuldv+NT40ULeK7lEre4ro8OE/teafAM/bBibg.
ECDSA key fingerprint is MD5:8f:86:05:da:66:c3:d8:fd:cf:09:d4:21:bc:84:80:be.
Are you sure you want to continue connecting (yes/no)? 
```



## SFTP文件传输

可以通过sftp或scp命令向服务器上传或下载文件.

用sftp命令连接和ssh类似,连接以后可以使用一些如cd,ls,pwd,mkdir之类的命令来在远程服务器活动.而针对本机的命令在针对服务器的命令前加上l,比如lcd,lls,lpwd等.

上传文件使用put命令.例如将本地find.log文件传到服务器端:

```sh
[root@server1 ~]# sftp 10.1.1.2
root@10.1.1.2's password: 
Connected to 10.1.1.2.
sftp> put find.log /tmp
Uploading find.log to /tmp/find.log
find.log                                                   100% 8958KB  56.2MB/s   00:00 
```

下载文件或目录使用get命令:

```sh
[root@server1 ~]# sftp 10.1.1.2
root@10.1.1.2's password: 
Connected to 10.1.1.2.
sftp> get html/
Fetching /root/html/ to html
Cannot download non-regular file: /root/html/
sftp> get html/*
Fetching /root/html/index.html to index.html
/root/html/index.html                                      100%   81    46.8KB/s   00:00 
Fetching /root/html/index.html.1 to index.html.1
/root/html/index.html.1                                    100%   81    48.0KB/s   00:00 
Fetching /root/html/index.html.2 to index.html.2
/root/html/index.html.2                                    100%   81    42.1KB/s   00:00 
sftp> exit
```

注意,上传或下载目录时,要想保持文件夹结构,需要使用-r参数.



## SCP文件传输

如果知道服务器上存在的文件信息,可以使用scp命令来上传下载.

上传命令: `scp [-pr] [-l 速率] 本地文件路径 [账号@]主机:远程目录名`

下载命令: `scp [-pr] [-l 速率] [账号@]主机:远程文件路径 本地目录名`

选项说明:

| 选项 | 说明                                             |
| ---- | ------------------------------------------------ |
| -p   | 保留文件权限信息                                 |
| -r   | 递归目录                                         |
| -l   | 限制传输速度,单位为Kbits/s.例如-l 800代表100KB/s |

例如将本地/etc/hosts*上传到10.1.1.2的/root/html目录中:

```sh
[root@server1 ~]# scp /etc/hosts* root@10.1.1.2:/root/html/
root@10.1.1.2's password: 
hosts                                                      100%  801   533.0KB/s   00:00 
hosts.allow                                                100%  370   281.4KB/s   00:00 
hosts.deny                                                 100%  460   391.2KB/s   00:00 
```

将远程主机10.1.1.2中的/boot/目录下载到本地/tmp目录下:

```sh
[root@server1 ~]# scp -r root@10.1.1.2:/boot /tmp
root@10.1.1.2's password: 
device.map                                                 100%   84    16.4KB/s   00:00 
gcry_rmd160.mod                                            100% 8068     2.4MB/s   00:00 
acpi.mod                                                   100% 9932     5.0MB/s   00:00 
gcry_rsa.mod                                               100% 2068     1.3MB/s   00:00
```



## SSH服务器配置文件

sshd服务器配置文件存放在/etc/ssh/sshd_config.一些重要的配置选项如下:

- **Port** 22: 默认端口22,也可以使用多个端口,在配置中多定义一个Port选项即可.
- Protocol 2: SSH协议版本.可以指定为Protocol 2, 1来同时支持v1和v2版本.
- **ListenAddress** 0.0.0.0: 默认监听所有端口.
- PidFile /var/run/sshd.pid: 放置PID文件的路径.
- LoginGraceTime 2m: 连接超时时间,包括密码输入等待时间.
- HostKey /etc/ssh/ssh_host_key: SSH v1使用的私钥.
- HostKey /etc/ssh/ssh_host_rsa_key: SSH v2使用的RSA私钥.
- HostKey /etc/ssh/ssh_host_dsa_key: SSH v2使用的DSA私钥.
- LogLevel INFO: 日志的等级.
- **PermitRootLogin** yes: 是否允许root登录.
- StrictModes yes: 是否让sshd检查用户主目录或相关文件的权限.
- **PubkeyAuthentication** yes: 是否允许通过密钥登录.
- **AuthorizedKeysFile** .ssh/authorized_keys: 自定义公钥数据存放位置.
- **PasswordAuthentication** yes: 是否允许通过密码登录.
- PermitEmptyPasswords no: 是否允许以空密码登录.
- IgnoreUserKnowHosts no: 是否忽略known_hosts文件.
- UsePAM yes: 利用PAM管理用户认证.
- X11Forwarding yes: 让窗口数据通过SSH连接来传送.
- PrintLastLog yes: 是否显示上次登录信息.
- TCPKeepAlive yes: 是否一直发送TCP数据包给客户端,来检测连接状态.
- UseDNS yes: 是否使用DNS去反查客户端的主机名.在内网可以设为no加速连接.



## 通过密钥登录

可以将客户端用户公钥复制到服务端用户目录内的ssh认证文件中,这样可以不用密码使用SSH和SFTP.

首先客户端生成密钥对(id_rsa私钥,id_rsa.pub公钥),可以用-t参数指定rsa或dsa算法:

```sh
[root@server1 ~]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
```

使用ssh-copy-id命令将公钥id_rsa.pub内容传送到root@192.168.2.254:

```sh
[root@server1 ~]# ssh-copy-id root@192.168.2.254
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.2.254's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.2.254'"
and check to make sure that only the key(s) you wanted were added.
```

至此便可以通过ssh免密登录192.168.2.254了:

```sh
[root@server1 ~]# ssh root@192.168.2.254
Last login: Sat Oct  9 20:00:00 2021 from 192.168.2.101
```



## 修改SSH端口

在SELinux开启的情况下,修改ssh配置文件中端口配置,将无法启动ssh.

```sh
[root@server1 ~]# journalctl -xe
Oct 10 01:22:37 server1 sshd[10905]: error: Bind to port 20000 on :: failed: Permission denie
Oct 10 01:22:37 server1 sshd[10905]: fatal: Cannot bind any address.
Oct 10 01:22:37 server1 systemd[1]: sshd.service: main process exited, code=exited, status=25
Oct 10 01:22:37 server1 systemd[1]: Failed to start OpenSSH server daemon.
-- Subject: Unit sshd.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit sshd.service has failed.
-- 
-- The result is failed.
Oct 10 01:22:37 server1 systemd[1]: Unit sshd.service entered failed state.
Oct 10 01:22:37 server1 systemd[1]: sshd.service failed.
Oct 10 01:22:37 server1 polkitd[963]: Unregistered Authentication Agent for unix-process:1088
[root@server1 ~]# cat /var/log/audit/audit.log | grep AVC | grep ssh
type=AVC msg=audit(1633800192.050:404): avc:  denied  { name_bind } for  pid=10954 comm="sshd" src=20000 scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

通过安装好setroubleshoot诊断套件后,查询SELinux日志信息:

```sh
[root@server1 ~]# yum -y install setroubleshoot
[root@server1 ~]# tail /var/log/messages
Oct 10 01:48:40 server1 setroubleshoot: SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket port 20000. For complete SELinux messages run: sealert -l 04603a0d-6723-417a-839a-d0974457f614
[root@server1 ~]# sealert -l 04603a0d-6723-417a-839a-d0974457f614
SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket port 20000.

*****  Plugin bind_ports (92.2 confidence) suggests   ************************

If you want to allow /usr/sbin/sshd to bind to network port 20000
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 20000
    where PORT_TYPE is one of the following: ssh_port_t, vnc_port_t, xserver_port_t.
```

根据提示,有三种解决方式:

- 修改默认绑定端口的设置: `semanage port -a -t ssh_port_t -p tcp 20000`
- 修改nis的设置: `setsebool -P nis_enabled 1`
- 强制允许动作: `ausearch -c 'sshd' --raw | audit2allow -M my-sshd;semodule -i my-sshd.pp`

以上任意一种方式都可以解决端口绑定问题.



## 使用SSH通道

可以通过ssh开启一个加密的通道来供其他不支持加密的应用使用.

例如在本地开启端口8888连接到远端服务器192.168.2.234的22端口:

```sh
[root@server2 ~]# ssh -L 8888:127.0.0.1:22 -N 192.168.2.234
[root@server2 ~]# netstat -ntulp | grep 8888
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      78842/ssh 
[root@server2 ~]# ssh -p 8888 127.0.0.1
Last login: Sun Oct 10 02:20:31 2021 from localhost
```

本地8888端口将一直处于监听中,ssh本地8888端口将直接连接到192.168.2.234的22端口.要中断通道直接关闭响应ssh进程.

