# DNS服务

## 域名系统

域名系统主要作用是将计算机主机名转成IP.完整主机名(FQDN, Fully Qualified Domain Name)由主机名与域名(Hostname and Domain Name)组成.

在DNS数据库中针对每个要解析的域(domain)称为一个区域(zone).每个DNS服务器必备称为hint的区域,记录根.服务器地址.

域名系统(DNS, Domain Name System)利用类似树状目录结构,将主机名的管理分配在不同层级的DNS服务器中,并进行分层管理.最顶层的DNS服务器叫.根服务器(root),最早管理的只有com, edu, gov, mil, org, net这种特殊区域以及以国家为分类的第二层的主机名,这两者称为顶级域名(TLDs, Top Level Domains).

DNS采用分层查询流程的好处是:

- 主机名有修改时,只需要改动上一层DNS服务器记录,维护简单.
- DNS服务器可以将主机名解析结果缓存起来,下次查询能快速响应.
- 可持续向下授权(子域名授权),也就是可以设置任意多级域名.

DNS服务器使用的监听端口为53,DNS查询同时使用UDP和TCP数据包.



## 域名查询流程

以向DNS服务器8.8.8.8查询域名www.abc.edu.cn为例,整个流程如下:

1. 首先查询本地/etc/hosts文件中有无记录,没有的话向/etc/resolv.conf中的DNS服务器8.8.8.8查询(由/etc/nsswitch.conf文件配置的查询顺序);
2. DNS服务器8.8.8.8收到查询请求发现没有对应记录,会向最顶层也就是.(root)的服务器查询域名;
3. root服务器中记录了.cn顶级域名服务器的地址,将其回报给8.8.8.8;
4. 顶级域名.cn服务器回报.edu.cn二级域名服务器地址给8.8.8.8;
5. 二级域名.edu.cn服务器发送.abc.edu.cn对应的IP地址给8.8.8.8;
6. 最后通过访问.abc.edu.cn服务器IP地址,得到其www服务对应IP地址;
7. DNS服务器8.8.8.8将查询结果返回给客户端,并将它记录在自己的缓存中(有效时间一般24小时).

可以使用dig +trace命令来观察这一过程:

```sh
[root@server2 ~]# dig +trace mail.dlut.edu.cn

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> +trace mail.dlut.edu.cn
;; global options: +cmd
.                       25330   IN      NS      m.root-servers.net.
.                       25330   IN      NS      c.root-servers.net.
.                       25330   IN      NS      a.root-servers.net.
;; Received 267 bytes from 222.246.129.80#53(222.246.129.80) in 4 ms

cn.                     172800  IN      NS      e.dns.cn.
cn.                     172800  IN      NS      f.dns.cn.
cn.                     172800  IN      NS      ns.cernet.net.
cn.                     86400   IN      DS      57724 8 2 5D0423633EB24A499BE78AA22D1C0C9BA36218FF49FD95A4CDF1A4AD 97C67044
cn.                     86400   IN      RRSIG   DS 8 1 86400 20211025050000 20211012040000 14748 . H7PWSfaMnU5bqTVD4anI0Gkagqbpu3jXuVX6wiGfaBpMYAU06BlzVDSk TcGqHd5qSfztQrT0ztQnWno12NhCfFzOyf64hv54quKOYWss8ilQnmgX AftgAZYD8V/v/cAbo2EKoCwLD8KAWoiUN9VGQpcLeVpb/O4mQ1xMbRbr ENxir09m6iYn+F9Y3MCTekj79c1RWKJ8Qqn+nJlj+bES2OvDNuEmSHlZ 4D/NSqz4WMI3IaaCk/WaKf6NzZHvazs8NsDVXDnqgND3opj6g0B3fQMC NVbtLO/M+FyuYNXmuk/VOettllpYUAOYaSdPZLxmqWXyTgiVbNXTH74S 1cRveA==
;; Received 707 bytes from 2001:500:2f::f#53(f.root-servers.net) in 23 ms

mail.dlut.edu.cn.       59602   IN      A       202.118.66.82
;; Received 106 bytes from 203.119.26.1#53(b.dns.cn) in 4 ms
```



## DNS注册

申请合法的主机名需要注册,注册取得的数据有两种:

- 一种是域名托管.设置FQDN对应的A记录,由上层DNS服务器来解析主机名.
- 一种是申请区域查询权.自架DNS服务器写在NS记录中,通过自己的DNS服务器解析主机名.



## DNS查询

DNS查询命令由bind-utils包提供,需要使用yum安装:

```sh
[root@server2 ~]# yum install -y bind-utils
```

### host

正解查询域名使用host命令.例如查询163.com的IP地址:

```sh
[root@server2 ~]# host 163.com
163.com has address 123.58.180.7
163.com has address 123.58.180.8
163.com mail is handled by 10 163mx02.mxmail.netease.com.
```

查询域名更详细信息使用-a参数:

```sh
[root@server2 ~]# host -a douban.com
Trying "douban.com"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61546
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;douban.com.                    IN      ANY

;; ANSWER SECTION:
douban.com.             180     IN      A       81.70.124.99
douban.com.             180     IN      A       140.143.177.206
douban.com.             180     IN      A       49.233.242.15

Received 76 bytes from 240e:50:5000::80#53 in 8 ms
```

### nslookup

nslookup命令可以用来正解与反解查询,正解查询结果基本上与host命令一致:

```sh
^C[root@server2 ~]# nslookup 163.com
Server:         222.246.129.80
Address:        222.246.129.80#53

Non-authoritative answer:
Name:   163.com
Address: 123.58.180.7
Name:   163.com
Address: 123.58.180.8
```

### dig

dig命令综合了上两命令的功能,主要参数有:

| 参数    | 说明                        |
| ------- | --------------------------- |
| +trace  | 从根目录开始追踪            |
| -t type | 查询的数据有MX,NS,SOA等类型 |
| -x      | 查询反解信息                |

例如使用默认参数查询linux.org:

```sh
[root@server2 ~]# dig linux.org

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> linux.org
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30344
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;linux.org.                     IN      A

;; ANSWER SECTION:
linux.org.              300     IN      A       104.21.48.239
linux.org.              300     IN      A       172.67.157.1

;; Query time: 338 msec
;; SERVER: 222.246.129.80#53(222.246.129.80)
;; WHEN: Wed Oct 13 03:21:51 CST 2021
;; MSG SIZE  rcvd: 70
```

结果显示重要部分如下:

- QUESTION: 显示要查询的内容.
- ANSWER: 显示查询的结果.数字300表示查询结果缓存时间.
- AUTHORITY: 显示查询结果来源DNS服务器.

### whois

whois用来查询域名所有者的信息:

```sh
[root@server2 ~]# whois 163.com
   Domain Name: 163.COM
   Registry Domain ID: 473619_DOMAIN_COM-VRSN
   Registrar WHOIS Server: whois.markmonitor.com
   Registrar URL: http://www.markmonitor.com
   Updated Date: 2019-01-25T07:56:57Z
```



## 正解文件记录

从主机名查询到IP的流程称为正解,正解文件资源记录(RR, Resource Record)格式如下:

`[domain]  [ttl]  IN  [[RR type]  [RR data]]`

字段说明:

- domain: 域名记录使用FQDN,也就是在主机名结尾加上小数点.例如`www.abc.com.`.
- ttl: 暂存时间(time to live),单位为秒.指其他DNS服务器缓存此笔记录的时间.
- IN: 固定关键词.
- RR type: 资源类型,如A或NS.
- RR data: 资源内容,如IP地址或NS域名.

资源类型有下面几种:

- A(Address)记录: 记录对应IPv4地址.

  ```sh
  [root@server3 ~]# dig -t a douban.com
  douban.com.             300     IN      A       81.70.124.99
  ```

- AAAA记录: 记录对应IPv6地址.

- NS(NameServer)记录: 查询管理区域名(Zone)的服务器主机名.

  ```sh
  [root@server3 ~]# dig -t ns douban.com
  douban.com.             59880   IN      NS      ns3.dnsv4.com.
  ```

- SOA(Start Of Authority): 开始验证的标志.

  ```sh
  [root@server3 ~]# dig -t soa douban.com
  douban.com.             180     IN      SOA     ns3.dnsv4.com. enterprise2dnsadmin.dnspod.com. 1629873707 3600 180 1209600 180
  ```

  SOA后面会接七个参数,按顺序分别表示:

  - 主DNS服务器主机名.
  - 管理员的Email地址.第一个点需要替换成@才是实际地址.
  - 序号(Serial),代表数据库文件的版本,序号越大表示越新.
  - 更新频率(Refresh),单位为秒,需大于2*Retry时间.指从服务器向主服务器检查更新的频率.
  - 失败重试时间(Retry),单位为秒.指从服务器连不上主服务器时,重试等待时间.
  - 失效时间(Expire),单位为秒,需大于Refresh+Retry或七天以上时间.指失败尝试总时长到这一值后停止重试.
  - 缓存时间(Minumum TTL),单位为秒.当没有设置TTL时,用此值代替.

- MX(Mail eXchanger)记录: 邮件服务器主机名字.

  ```sh
  [root@server3 ~]# dig -t mx dlut.edu.cn 
  dlut.edu.cn.            74898   IN      MX      20 mx2.dlut.edu.cn.
  ```

  邮件服务器前面的数字代表优先级,有多台邮件服务器情况下数字越小优先级越高.

- CNAME(Alias)记录: 主机名的别名.

  ```sh
  [root@server3 ~]# dig www.douban.com
  www.douban.com.         2       IN      CNAME   forward.douban.com.
  ```

  常用于单IP对应多个域名情况下,可以将域名指向别名,而别名指向目标IP,这样当更换IP地址时,只需要更新别名与IP对应的记录就行.不用针对每个域名设置一遍.



## 反解文件记录

从IP反解析到主机名的流程称为反解,需要直属上层ISP授权.使用nslookup进行反解查询:

```sh
[root@server2 ~]# nslookup 123.58.180.7
7.180.58.123.in-addr.arpa.       name = 163.com.

Authoritative answers can be found from:
```

由于IP从左到右解析,与域名相反,所以反解的ZOne就必须要将IP反过来写,在结尾加上.in-addr.arpa.字样.

反解区最主要的类型是PTR(PoinTeR)记录,即查询IP对应的主机名.



## 客户端设置

客户端和DNS有关的配置文件主要有三个:

- /etc/hosts: 主机名与IP对应设置的本地文件.
- /etc/resolv.conf: 记录DNS服务器IP.会被DHCP服务器更新,可以在网卡配置文件中加入PEERDNS=no来禁止.
- /etc/nsswitch.conf: 决定DNS查询顺序,本地解析还是DNS服务器解析优先.

/etc/hosts设置格式如下,一行一个主机名和IP对应:

```sh
[root@server2 ~]# cat /etc/hosts
207.97.227.243 www.github.com 
192.168.2.234  server1 smblinux
```

/etc/resolv.conf设置格式如下,可以设置多个DNS地址,一般不超过3个:

```sh
[root@server2 ~]# cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 222.246.129.80
```

/etc/nsswitch.conf设置选项如下,files代表本地解析,dns代表网络DNS服务器解析:

```sh
[root@server2 ~]# cat /etc/nsswitch.conf
hosts:      files dns myhostname
```



## 主从构架

为了让DNS服务持续可用,通常DNS服务器采取主从构架(Master/Slave).

在主服务器(Master)中,所有主机名相关信息都需要手动设置,从服务器(Slave)则直接拉取主服务器的配置同步.

主从服务器数据同步依据的是数据库版本号,主服务器每次更新数据后,数据库版本号会增加,主服务会主动告知从服务器要更新数据库,从服务器也会定期检测主服务器数据库版本来来决定更新.



## 服务器安装

搭建DNS服务器所需的软件名叫BIND(Berkeley Internet Name Domain).可以使用yum安装:

```sh
[root@server2 ~]# yum install -y bind
```

和服务器设置有关的文件与目录:

- /etc/named.conf: 主配置文件.
- /etc/sysconfig/named: 启动chroot及额外参数控制.
- /var/named/: 数据库默认存放目录.
- /var/run/named/: named程序放置pid文件的目录.



## 缓存DNS服务器

最简单的唯高速缓存(Cache-Only)DNS服务器只需要一个.根目录zone文件,它只有缓存搜寻结果的功能.甚至连根目录指向也可以不需要,丢给上层DNS服务器去转发(Forwarding).

配置文件/etc/named.conf内容如下:

```sh
[root@server2 ~]# vi /etc/named.conf
options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };

        recursion yes;
        forward only;
        forwarders {
                222.246.129.80;
                8.8.8.8;
        };
};
```

配置文件选项说明如下:

- **listen-on port 53 { any; };**

  设置本机监听使用的网口,可以设置多个端口.

- **directory "/var/named";**

  设置正解和反解的zone文件存放目录.

- **dump-file, statistics-file, memstatistics-file**

  与named服务有关的统计信息文件路径.

- **allow-query  { any; };**

  设置可以使用DNS查询的客户端列表.

- **forward only;**

  设置DNS服务器仅进行转发,忽略zone文件设置.

- **forwarders {222.246.129.80; 8.8.8.8;};**

  设置转发到的上层DNS服务器地址.

启动named服务:

```sh
[root@server2 ~]# systemctl start named
[root@server2 ~]# systemctl enable named
[root@server2 ~]# netstat -ntulp |grep named
tcp        0      0 10.1.1.1:53             0.0.0.0:*           LISTEN      43486/named   
tcp        0      0 192.168.2.254:53        0.0.0.0:*           LISTEN      43486/named   
tcp        0      0 127.0.0.1:53            0.0.0.0:*           LISTEN      43486/named   
tcp        0      0 127.0.0.1:953           0.0.0.0:*           LISTEN      43486/named 
```

其中953端口是named的远程控制功能,称为远程名称解析服务控制功能(RNDC, Remote Name Daemon Control).

查看named相关日志:

```sh
[root@server2 ~]# tail -30 /var/log/messages | grep named
Oct 13 05:20:42 server2 named[43486]: configuring command channel from '/etc/rndc.key'
Oct 13 05:20:42 server2 named[43486]: command channel listening on 127.0.0.1#953
Oct 13 05:20:42 server2 named[43486]: configuring command channel from '/etc/rndc.key'
Oct 13 05:20:42 server2 named[43486]: command channel listening on ::1#953
Oct 13 05:20:42 server2 named[43486]: managed-keys-zone: loaded serial 0
Oct 13 05:20:42 server2 named[43486]: all zones loaded
Oct 13 05:20:42 server2 named[43486]: running
```

用客户端测试,使用dig指定DNS服务器为192.168.2.254:

```sh
[root@server3 ~]# dig 163.com @192.168.2.254
;; SERVER: 192.168.2.254#53(192.168.2.254)
```

结果显示解析通过192.168.2.254服务器,表示已经成功.







