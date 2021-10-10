# DHCP服务

## DHCP服务器

DHCP(Dynamic Host Configuration Protocol)服务器主要工作是自动地将网路参数分配给网络中的计算机,这些网络参数包括IP,网关,DNS地址等.

一般DHCP服务器用在计算机(或移动设备)数量众多的局域网内.如果局域网内计算机数量较少,使用手动配置IP能节省开机后从DHCP服务器获取IP的时间.



## DHCP协议

DHCP通常是用于局域网内的一个通信协议,它主要通过客户端发送广播数据包给整个物理网段内的所有主机,如果有DHCP服务器则会响应客户端的IP参数要求.

整个请求过程如下:

1. 若客户端网卡设置为自动获得IP,则当客户端开机或重启网卡时,网卡会发送搜索DHCP服务器的UDP数据包给整个物理网段.广播数据包发送目标为255.255.255.255,因此一般主机会直接忽略此包,只有DHCP服务器会响应.

2. DHCP服务器端在接收到客户端请求后,会针对客户端的MAC地址与本身的设置数据来进行下面工作:

   - 到服务器日志查询该用户是否曾租用过某个IP,若有记录且该IP当前闲置,则提供此IP给客户端.
   - 若配置文件针对此MAC地址设置了固定IP,则将此IP分配给客户端.
   - 如果不符合上面两个条件,则随机选取当前没有被使用的IP参数给客户端,并记录下来.

   由于客户端此时没有IP地址,因此针对客户端MAC来给与回应,此时服务端会保留这个租约然后等待客户端回应.

3. 有可能局域网内不止一台DHCP服务器,客户端需要选择确认DHCP服务器提供的相关网路参数租约.当决定好要使用的参数后后,客户端开始使用这组网络参数来配置自己的网络环境.此外客户端也会发送一个广播数据包告知已接受该服务器的租约.没有被接受的DHCP服务器会回收对应IP租约.

4. DHCP服务器端收到客户端确认选择后,会回送确认的响应数据包,并告知客户端这个租约的期限,并开始倒计时.当租约到期且没重新收到申请(renew)时,IP将会被收回,用户可以向DHCP服务器再次要求分配IP.此外客户端如果脱机,DHCP服务器也会将IP收回.

一般来说DHCP客户端程序大多会主动依据租约时间去重新申请IP,时间点为租约时间过半和过掉九成.服务端使用67端口监听客户请求,而客户端使用68端口向DHCP服务器请求.



## DHCP服务器配置

使用Linux假设DHCP服务器需要安装dhcp服务:

```sh
[root@server2 ~]# yum -y install dhcp
```

安装完毕后需要修改配置文件/etc/dhcp/dhcpd.conf后才能启动.作为参考可以查看示例文件/usr/share/doc/dhcp-4.2.5/dhcpd.conf.example.

服务器配置文件分为两大块,全局配置和IP分配设置.

### 全局配置

假设整个局域网只有一个子网,那除了IP分配之外的配置参数可以放在全局设置区域中.主要参数如下:

- default-lease-time: 默认租约时间,单位是秒.如果用户没有特别要求祖约时间,那么使用此默认值.
- max-lease-time: 最大租约时间,设置用户能请求的最大租约时间限制.
- option domain-name: 在用户查找的主机名后自动加上此域名后缀.
- **option domain-name-servers**: 设置客户端的DNS服务器.
- ddns-update-style: 通过ddns来更新主机名与IP的对应关系.
- **option routers**: 设定网关的IP地址.

### IP分配设置

根据分配类型动态IP使用range参数指定.例如分配范围10.1.1.101~10.1.1.200:

```sh
subnet 10.1.1.0 netmask 255.255.255.0 {
  range 10.1.1.101 10.1.1.200;
}
```

静态IP分配指定MAC地址与对应的固定IP.例如设置主机名server1,绑定MAC地址10.1.1.103:

```sh
host server1 {
  hardware ethernet 00:0c:29:ab:18:72;
  fixed-address 10.1.1.103;
}
```

### 配置文件

整个配置文件内容如下:

```sh
[root@server2 ~]# vi /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
option domain-name-servers 8.8.8.8, 114.114.114.114;
option routers 10.1.1.1;

subnet 10.1.1.0 netmask 255.255.255.0 {
  range 10.1.1.101 10.1.1.200;
}

host server1 {
  hardware ethernet 00:0c:29:ab:18:72;
  fixed-address 10.1.1.103;
}
```

之后就可以启动dhcp服务器了:

```sh
[root@server2 ~]# systemctl start dhcpd
```

在客户端中获取到的IP参数信息保存在/var/lib/dhclient/dhclient.leases文件中:

```sh
[root@server3 ~]# cat /var/lib/dhclient/dhclient.leases 
lease {
  interface "ens37";
  fixed-address 10.1.1.129;
  option subnet-mask 255.255.255.0;
  option dhcp-lease-time 1800;
  option dhcp-message-type 5;
  option domain-name-servers 10.1.1.1;
  option dhcp-server-identifier 10.1.1.254;
  option broadcast-address 10.1.1.255;
  option domain-name "localdomain";
  renew 5 2021/09/24 03:02:47;
  rebind 5 2021/09/24 03:17:31;
  expire 5 2021/09/24 03:21:16;
}
```

服务端的记录保存在/var/lib/dhcpd/dhcpd.leases



## 通过网络唤醒

在局域网内的计算机只要支持且开启了网络唤醒功能,就可以使用ether-wake命令来让目标从关机状态开机.

例如唤醒MAC地址为AA:BB:CC:DD:EE:FF的主机开机:

```sh
[root@server2 ~]# ether-wake -i ens33 AA:BB:CC:DD:EE:FF
```

