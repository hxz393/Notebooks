# NTP服务

## NTP通信协议

NTP是网络校时协议(Network Time Protocol).为了修正因为BIOS内部芯片问题导致与标准时间(UTC, Coordinated Universal Time)存在的偏差,而通过网络进行时间同步(Synchronize).此外DTSS(Digital Time Synchronization Protocol)也可实现同样的功能.

Linux系统中有两个时间:

- 软件时钟: Linux自己的系统时间,从1970.1.1开始记录的时间参数.
- 硬件时钟: 计算机在BIOS记录的实际时间,通过硬件记录.写入硬件时钟使用hwclock命令.

NTP服务器使用的端口为123,通过UDP数据包传输.

我国授时中心服务器的IP地址为210.72.145.44.



## NTP服务器层次

NTP时间服务器采用类似分级构架(Stratum)来处理时间的同步化,并且采用Server/Client的主从构架.网络上会提供一些主要与次要时间服务器,均属于第一级(stratum-1)与第二级(stratum-2)的时间服务器.

如果我们的NTP服务器向二级时间服务器要求时间同步,那么我们的NTP服务器即为三级(stratum-3)时间服务器.依此传递下去,最多可达15个阶层.

一般在进行NTP主机的设置时,会选择多台上层的Time Server来作为我们的NTP服务器校时用,这样可以避免某台时间服务器下线造成无法更新.



## NTP服务器设置

启动NTP服务器需要安装ntp软件:

```sh
[root@server2 ~]# yum -y install ntp ntpdate
```

修改时区可以使用timedatectl命令.例如修改成上海时间:

```sh
[root@server2 ~]# timedatectl set-timezone "Asia/Shanghai"
[root@server2 ~]# timedatectl
      Local time: Mon 2021-10-11 05:07:51 CST
  Universal time: Sun 2021-10-10 21:07:51 UTC
        RTC time: Sun 2021-10-10 21:07:51
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```

服务器配置文件位于/etc/ntp.conf,其内容修改部分如下:

```sh
[root@server2 ~]# vi /etc/ntp.conf
restrict 210.72.145.44
restrict 192.168.2.0 mask 255.255.255.0 nomodify
server 210.72.145.44
server cn.pool.ntp.org
driftfile /var/lib/ntp/drift
keys /etc/ntp/keys
```

其中restrict行用来控制权限,上面运行局域网内主机通过这部主机来进行网络校时.其他可用参数有:

- ignore: 拒绝所有类型的NTP连接.
- nomodify: 客户端不能使用ntpc和ntpq来修改服务器的时间参数,但可以通过这部主机来进行网络校时.
- noquery: 客户端不能使用ntpq和ntpc等命令来查询时间服务器,不提供NTP的网络校时.
- notrap: 不提供trap这个远程时间登录(remote event logging)的功能.
- notrust: 拒绝没有认证的客户端.

server行用来设置上层NTP服务器地址,可以设置多个.

driftfile行指定的文件用来记录本机与上层Time Server之间振荡周期频率误差.

keys行指定用来认证的密钥文件.



## NTP服务器管理

配置修改完毕后,可以通过systemctl来启动ntpd服务:

```sh
[root@server2 ~]# systemctl start ntpd
[root@server2 ~]# systemctl enable ntpd
[root@server2 ~]# netstat -ntulp | grep ntpd
udp     0      0 10.1.1.1:123            0.0.0.0:*                           30097/ntpd 
udp     0      0 192.168.2.254:123       0.0.0.0:*                           30097/ntpd  
```

通常NTP启动后需要十五分钟内才会和上层NTP服务器连接上.可以使用ntpstat命令查询:

```sh
[root@server2 ~]# ntpstat
synchronised to NTP server (111.230.189.174) at stratum 3
   time correct to within 212 ms
   polling server every 64 s
```

上面结果显示时间矫正了212ms,并且每隔64秒会主动去更新时间.

使用ntpq可以查询我们的NTP与相关上层NTP的状态:

```sh
[root@server2 ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 210.72.145.44   .INIT.          16 u    -   64    0    0.000    0.000   0.000
 ntp.wdc1.us.lea 130.133.1.10     2 u    9   64   37  251.796   -4.729   0.252
*111.230.189.174 100.122.36.4     2 u   65   64   17   21.948   -6.164   0.297
+ntp6.flashdance 194.58.202.20    2 u   66   64   17  237.552   -6.099   5.726
+2402:f000:1:416 186.195.4.16     2 u   62   64   17   33.758   -3.033   0.306
-sv1.ggsrv.de    205.46.178.169   2 u   65   64   17  280.605   10.958   2.962
```

各字段意义如下:

- remote: NTP主机的IP或主机名.最左边符号*代表目前使用的NTP,符号+表示已连接的备选NTP.
- refid: 参考的上层NTP主机地址.
- st: stratum阶层等级.
- when: 上次时间同步操作后的时间间隔.
- poll: 下次时间同步更新操作的时间间隔.
- reach: 已经向上层NTP服务器要求更新的次数.
- delay: 网络传输过程当中的延迟,单位为微秒.
- offset: 时间补偿结果,单位为毫秒.
- jitter: Linux系统时间与BIOS硬件时间的差异时间,单位为微秒.



## NTP客户端设置

客户端可以使用ntpdate命令来与服务器同步时间:

```sh
[root@server1 ~]# ntpdate 192.168.2.254
11 Oct 15:54:26 ntpdate[125764]: adjust time server 192.168.2.254 offset -0.002236 sec
[root@server1 ~]# date;hwclock -r
```

同步之后使用hwclock写入到硬件.ntpd服务端和客户端之间时间误差不允许超过1000秒.

服务端配置也是修改/etc/ntp.conf文件:

```sh
[root@server1 ~]# vi /etc/ntp.conf 
restrict 192.168.2.254
server 192.168.2.254
```

之后启动ntpd服务,客户端就会自动到服务端同步时间:

```sh
[root@server1 ~]# systemctl start ntpd
```

