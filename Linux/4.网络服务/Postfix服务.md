# Postfix服务

## 邮件服务器

架设邮件服务器(Mail Server)首先需要有一个合法注册的域名.然后需要申请IP反解策略.最后是设置MX和A类型记录,来对应到邮件服务器的IP地址上.

邮件传输过程中所需要的组件和协议如下:

- MUA(Mail User Agent)

  MUA顾名思义就是邮件用户代理人,用来将用户信件送到邮件服务器上.最常见的MUA有Thunderbid, Kmail, Outlook Express等所谓邮件客户端.

- MTA(Mail Transfer Agent)

  MTA是邮件发送代理人,它可以使用简单邮件传输协议(SMTP, Simple Mail Transfer Protocol)来接受用户邮件,并具有中继转发(Relay)功能.常见的MTA有sendmail, postfix等所谓邮件服务端.

- MDA(Mail Delivery Agent)

  MDA是邮件传送代理人,其实是MTA下面的组件.主要功能是分析由MTA所收到的邮件表头或内容等数据,来决定这份邮件的去向.此外MDA还有分析与过滤邮件和自动回复功能.

用户收信过程中所需的组件和协议如下:

- MRA(Mail Retrieval Agent)

  MRA也是MTA下面的组件,用户可以通过邮政服务协议(POP, Post Office Protocol)来接收自己的邮件,也可以通过IMAP(Internet Message Access Protocol)将邮件保留在邮件主机上面

- Mailbox

  电子邮箱就是某个账号专用的邮件收取文件.Linux下默认放在/var/spool/mail/下面.

  

## 邮件收发流程

电子邮件的内容分为标题(header)和内容(body)两部分.下面是一份测试邮件内容:

```sh
[user1@server1 ~]$ echo "testmail" | mail -s "mailheader1" user1
[user1@server1 ~]$ cat /var/spool/mail/user1 
From root@server1  Sun Oct 17 20:53:33 2021
Return-Path: <root@server1>
Received: from server1 (localhost [127.0.0.1])
        by server1 (8.14.7/8.14.7) with ESMTP id 19HCrXob053690
        for <user1@server1>; Sun, 17 Oct 2021 20:53:33 +0800
Received: (from root@localhost)
        by server1 (8.14.7/8.14.7/Submit) id 19HCrX7P053605
        for user1; Sun, 17 Oct 2021 20:53:33 +0800
From: root <root@server1>
Message-Id: <202110171253.19HCrX7P053605@server1>
Date: Sun, 17 Oct 2021 20:52:33 +0800
To: user1@server1
Subject: mailheader1
User-Agent: Heirloom mailx 12.5 7/5/10
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit

testmail
```

邮件传输流程的流程如下:

1. 本地MUA想要使用MTA来发送邮件时,首先需要取得MTA的权限\.一般是注册对应的Email账号.
2. 用户在MUA上编写邮件后发送到MTA上.邮件包括标题和内容.
3. 如果MTA收到邮件的目标是自己的用户,直接通过MDA将邮件送往对应Mailbox.如果目标是其他MTA,则MDA会开始进行邮件转发,发往下一台MTA的SMTP端口(25).邮件顺利发送出去后,该邮件会从队列当中删除掉.
4. 对方MTA收到转发信件后,发往对应的用户Mailbox中.

通过POP3来收取邮件流程如下:

1. MUA通过POP3协议连接到MRA的110端口,并输入账号和密码来取得正确认证与授权.
2. MRA确认用户密码正确后,会前往用户的Mailbox取得邮件,并发送到用户MUA上面.
3. 当所有邮件传送完毕后,用户的Mailbox内数据将被删除.

对于MTA来说邮件传送的流程如下:

1. 发信端与收信端主机会先经过一个握手(ehlo)阶段,此时发信端被记录为发信来源,将邮件标题进行传送.
2. 收信端主机会分析标题信息,若邮件Mail to地址为收信端主机,且名称符合mydestination的值,则邮件会开始被接受到队列,并送到Mailbox中.若不符合mydestination的值,则终止连接不接受后续的正文内容传送.
3. 若Mail to地址不是收信端本身,继续进行中继转发(relay)分析.先判断邮件来源是否是信任客户端(mynetworks的值),若不是则继续分析邮件来源或目标是否符合relay_domains的设置,以上任意符合的话开始接收邮件至队列中,并等待MDA将邮件转发出去.否则会终止连接不接受后续的正文内容传送.



## Postfix邮件服务

Postfix为完全兼容于SendMail所设计出来的一个邮件服务器软件,在外部配置文件的支持度上与SendMail没有区别.所以在CentOS中作为内建功能提供.

其主要的配置文件有:

- /etc/postfix/main.cf

  最主要的Postfix配置文件,修改配置文件后需要重启Postfix.

- /etc/postfix/master.cf

  规定Postfix每个程序的工作参数.通常不需要改动.

- /etc/postfix/access

  设置开放Relay或拒绝连接的来源与目标地址等信息的外部配置文件.需要以postmap来处理成为数据库文件.

- /etc/aliases

  作为邮件别名的用途,也可以作为邮件组的设置.



## Postfix配置文件

配置文件/etc/postfix/main.cf中设置值格式如myhostname = www.hxz.ass,等号两边必须要有空格.多个设置值之间用逗号分开.此外还可以使用$来使用变量.

常用的设置选项如下:

- myhostname: 主机名(FQDN)

  此设置值会被后续很多参数引用,例如myhostname = www.hxz.ass

- myorigin = $myhostname

  设置寄信地址,如果寄信时没有设置Mail from字段,会以此值作为默认值.

- inet_interfaces = all

  设置监听网口.默认情况下postfix仅针对本机进行监听,可以修改成all来监听所有网口.

- mydestination = $myhostname

  设置本机收信地址,要与MX记录指向相匹配.

- mynetworks = 10.1.1.0/24

  设置信任的客户端来源地址.可以对信任区域帮忙进行Relay转发.

整个配置文件内容如下:

```sh
[root@server2 ~]# vi /etc/postfix/main.cf
myhostname = www.hxz.ass
myorigin = $myhostname
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks = 10.1.1.0/24, 127.0.0.0/8
relay_domains = $mydestination
```



## Postfix服务启动

先使用postmap和postalias来重建和转换数据库:

```sh
[root@server2 ~]# postmap hash:/etc/postfix/access 
[root@server2 ~]# postalias hash:/etc/aliases
```

然后使用postfix check命令检查配置文件是否有错误.之后就可以启动服务了:

```sh
[root@server2 ~]# postfix check
[root@server2 ~]# postfix stop
postfix/postfix-script: stopping the Postfix mail system
[root@server2 ~]# postfix start
postfix/postfix-script: starting the Postfix mail system
[root@server2 ~]# netstat -ntulp |grep 25
tcp        0      0 0.0.0.0:25              0.0.0.0:*       LISTEN      117183/master 
```



## 邮件主机过滤

基本上制定了mynetworks的信任来源就能够让用户Relay了.此外还可以利用access文件来额外管理邮件过滤.

例如想让10.1.1.0/24和.hxz.ass使用这台MTA来转发邮件,不允许192.168.1.0/24使用:

```sh
[root@server2 ~]# vi /etc/postfix/access
10.1.1          OK
.hxz.ass        OK
192.168.1       REJECT
[root@server2 ~]# postmap hash:/etc/postfix/access
[root@server2 ~]# ll /etc/postfix/acces*
-rw-r--r--. 1 root root 20920 Oct 17 23:03 /etc/postfix/access
-rw-r--r--. 1 root root 12288 Oct 17 23:04 /etc/postfix/access.db
```

使用这个数据库文件的好处是,修改以后不必重启Postfix.



## 设定邮件别名

当一些系统账号运行的服务出现问题时,会将错误通过邮件发送给root用户,这里面就使用了aliases邮件别名配置文件来处理.

查看文件的内容,可以看到系统账号都对应到实体的root账号:

```sh
[root@server2 ~]# cat /etc/aliases
bin:            root
daemon:         root
adm:            root
```

可以设置普通账号hxz来接收root的邮件:

```sh
[root@server2 ~]# vi /etc/aliases
root:           root,hxz
```

可以设置一个用户组别名ass来代替user01,user02,hxz@ass.com,这样发信时可通过发给ass来群发邮件:

```sh
[root@server2 ~]# vi /etc/aliases
ass:            user01,user02,hxz@ass.com
[root@server2 ~]# postalias hash:/etc/aliases
```

普通用户可以在用户主目录下新建.forward文件,来定义转发自己的邮件地址:

```sh
[user01@server2 ~]$ vi .forward
user02
user01@mail.com
[user01@server2 ~]$ chmod 644 .forward 
```

注意.forward文件的权限一定得644.



## 邮件队列信息

有时候因为网络问题,导致某些邮件无法送出而被暂存在队列中.可以通过postquere -p命令来查看:

```sh
[user01@server2 ~]$ postquere -p
```

要查看邮件内容可以使用postcat命令来读取:

```sh
[root@server2 ~]# postcat 5CF324312
```

要立即将队列中得邮件发出去可以使用postfix flush命令操作:

```sh
[root@server2 ~]# postfix flush
```



## 收件设置

CentOS中使用dovecot来实现MRA的相关通信协议.使用yum安装后修改配置文件/etc/dovecot/dovecot.conf,设置只使用POP3/IMAP协议:

```sh
[root@server2 ~]# yum install -y dovecot
[root@server2 ~]# vi /etc/dovecot/dovecot.conf 
protocols = imap pop3 
[root@server2 ~]# vi /etc/dovecot/conf.d/10-ssl.conf 
#ssl = required
ssl = no
```

之后就可以启动dovecot了.检查下POP3端口110和IMAP端口143:

```sh
[root@server2 ~]# systemctl enable --now dovecot
[root@server2 ~]# netstat -ntulp |grep dovecot
tcp        0      0 0.0.0.0:110             0.0.0.0:*         LISTEN      122037/dovecot 
tcp        0      0 0.0.0.0:143             0.0.0.0:*         LISTEN      122037/dovecot 
```



## 防火墙设置

整个MTA主要通过SMTP端口进行邮件发送,因此放行25端口就可以:

```sh
[root@server2 ~]# iptables -A INPUT -p TCP -i ens33 --dport 25 --sport 1024:65534 -j ACCEPT
```

如果启动了dovecot,还要将110和143端口放行:

```sh
[root@server2 ~]# iptables -A INPUT -p TCP -i ens33 --dport 110 --sport 1024:65534 -j ACCEPT
[root@server2 ~]# iptables -A INPUT -p TCP -i ens33 --dport 143 --sport 1024:65534 -j ACCEPT
```



## 邮件信箱

每个账号都有一个mailbox,用来收取信件.可以使用mail命令来收发信件.

发送邮件的命令格式: `mail 用户名 -s "邮件标题"`

例如给user1发送一份邮件:

```sh
[root@101c7 ~]# mail user1 -s "Hello user1"
Subject: 1
Hello dear, 
        welcome to my world.
see u~    
.
EOT
```

邮件内容输入完毕,最后一行以.结束按回车,信件便发出去了.也可以用重定向 < 文件名 来将信件内容传给mail.

收取信件直接输入mail进入邮件程序,会有交互式界面.输入h查看所有标题:

```sh
[user1@101c7 ~]$ mail
Heirloom Mail version 12.5 7/5/10.  Type ? for help.
"/var/spool/mail/user1": 1 message 1 new
>N  1 root                  Mon Sep 13 12:05  21/684   "1"
& 
```

输入?可以查询所有命令,符号>指向的是当前邮件位置,可以按回车查看邮件内容,也可输入对应邮件编号来查看:

```sh
& 1
Message  1:
From root@101c7.localdomain  Mon Sep 13 12:05:27 2021
Return-Path: <root@101c7.localdomain>
X-Original-To: user1
Delivered-To: user1@101c7.localdomain
Date: Mon, 13 Sep 2021 12:05:27 -0400
To: user1@101c7.localdomain, Hello@101c7.localdomain,
        -s@101c7.localdomain, user1@101c7.localdomain
Subject: 1
User-Agent: Heirloom mailx 12.5 7/5/10
Content-Type: text/plain; charset=us-ascii
From: root@101c7.localdomain (root)
Status: R

Hello dear,
        welcome to my world.
see u~
```

输入R来回复>指向的邮件:

```sh
& R
To: root@server1.localdomain
Subject: Re: Cron <root@server1> /usr/bin/yum -y update
```

输入d 编号来删除对应邮件:

```sh
& d 3 4
& h
 A  1 (Cron Daemon)         Sat Sep 25 02:15  34/1134  "Cron <root@server1> /usr/bin/yum -y update"
 A  2 (Cron Daemon)         Sun Sep 26 02:16  38/1287  "Cron <root@server1> /usr/bin/yum -y update"
>U  5 (Cron Daemon)         Wed Sep 29 02:16  37/1275  "Cron <root@server1> /usr/bin/yum -y update"
```

储存邮件到文件输入s 编号 文件名:

```sh
& s 1 first
"first" [New file] 34/1134
```

输入q退出,刚才读过的信件会存入收件箱~/mbox中.



## 邮件客户端

在命令行下除了mail命令,还可以使用mutt软件来收发邮件:

```sh
[root@server2 ~]# yum install -y mutt
```

发送邮件使用的命令格式为: 

`mutt [-a 附件] [-i 邮件内容文件] [-b 秘密副本地址] [-c 一般副本地址] [-s 邮件标题] 收件地址`

例如编写一封信寄给assassing@163.com:

```sh
[root@server2 ~]# mutt -s 'mutt mail' assassing@163.com
To: assassing@163.com
Subject: mutt mail   
a mail from mutt!
y:Send  q:Abort  t:To  c:CC  s:Subj  a:Attach file  d:Descrip  ?:Help                                                              
    From: root <root@server2>
      To: assassing@163.com
      Cc:
     Bcc:
 Subject: mutt mail
Reply-To:
     Fcc: ~/sent
Security: None


-- Attachments                                                                                                                     
- I     1 /var/tmp/mutt-server2-0-123262-123667749   [text/plain, 7bit, us-ascii, 0.1K] 


-- Mutt: Compose  [Approx. msg size: 0.1K   Atts: 1]-----------------------------------
```

使用mutt收取外部邮件:

```sh
[root@server2 ~]# mutt -f imaps://mail.163.com
```



