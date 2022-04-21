# iSCSI服务

## NAS与SAN

网络附加储存服务器(NAS, Network Attached Storage)其实就是一台定制好的文件服务器,将NAS连上网络,在网络内的主机就能访问NAS上的数据.

储存局域网(SAN, Storage Area Networks)可以被视为一个外接式的储存设备,可以为局域网内所有主机提供磁盘而不是文件系统访问.



## iSCSI接口

通过TCP/IP技术而不是光纤接口来连接SAN就是iSCSI(Internet SCSI).iSCSI分为两部分:

- iSCSI target: 储存设备端,存放磁盘或RAID的设备.
- iSCSI initiator: 使用target的客户端,通常是服务器.



## iSCSI服务安装

iSCSI所需的软件有下面两个:

- scsi-target-utils: 用来将Linux系统仿真成为iSCSI target的功能.
- iscsi-initiator-utils: 挂载来自target的磁盘到本地.

直接使用yum安装:

```sh
[root@server2 ~]# yum install -y scsi-target-utils
[root@server2 ~]# yum install -y iscsi-initiator-utils
```



## iSCSI target设置

建立一个100MB大小的文件/srv/iscsi/disk.img:

```sh
[root@server2 ~]# mkdir /srv/iscsi
[root@server2 ~]# dd if=/dev/zero of=/srv/iscsi/disk.img bs=1M count=100
[root@server2 ~]# chcon -Rv -t tgtd_var_lib_t /srv/iscsi
```

建立一个200MB大小实际的分区/dev/sda3:

```sh
[root@server2 ~]# fdisk /dev/sda
[root@server2 ~]# partprobe
[root@server2 ~]# fdisk -l
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      411647      204800   83  Linux
/dev/sda2          411648    33187839    16388096   8e  Linux LVM
/dev/sda3        33187840    33579007      195584   83  Linux
/dev/sda4        33579008    41943039     4182016    5  Extended
/dev/sda5        33581056    34195455      307200   8e  Linux LVM
```

建立一个300MB大小的LV设备/dev/centos/iscsi01:

```sh
[root@server2 ~]# pvcreate /dev/sda5
[root@server2 ~]# vgextend centos /dev/sda5
[root@server2 ~]# lvcreate -L 296M  -n iscsi01 centos
[root@server2 ~]# lvscan
  ACTIVE            '/dev/centos/home' [4.88 GiB] inherit
  ACTIVE            '/dev/centos/iscsi01' [296.00 MiB] inherit
```

iSCSI target文件名以iqn(iSCSI Qualified Name)开头,命名格式为`iqn.yyyy-mm.<reversed domain>:id`.这里使用名称iqn.2000-01.server2:mydisk.

每个target能够拥有数个磁盘设备,这些磁盘被叫做逻辑单位编号(LUN, Logical Unit Number).iSCSI initiator就是跟target协调后才取得LUN的访问权.

tgt的配置文件/etc/tgt/targets.conf可以很简单,将上面新建的三个LUN加入到iqn中即可:

```sh
[root@server2 ~]# vi /etc/tgt/targets.conf 
<target iqn.2000-01.server2:mydisk>
        backing-store /srv/iscsi/disk.img
        backing-store /dev/sda3
        backing-store /dev/centos/iscsi01
        initiator-address 10.1.1.0/24
        incominguser iuser ipass
        write-cache off
</target>
```

参数说明如下:

- backing-store/direct-store: 一般用backing-store设置虚拟设备,想整块硬盘都被使用可以用direct-store.
- initiator-address: 限制客户端地址,一般也不用设置而用iptables来规划.
- incominguser: 设置通过账号密码来使用iSCSI target.
- write-cache: 是否使用缓存,默认使用缓存来提高读写速度,有数据丢失风险.

如果没有使用initiator-address参数,使用防火墙来规定只有10.1.1.0/24可以访问target设置如下:

```sh
[root@server2 ~]# iptables -A INPUT -p tcp -s 10.1.1.0/24 --dport 3260 -j ACCEPT
```



## iSCSI target启动

启动tgtd服务:

```sh
[root@server2 ~]# systemctl enable --now tgtd
[root@server2 ~]# netstat -ntulp |grep tgt
tcp        0      0 0.0.0.0:3260            0.0.0.0:*         LISTEN      23981/tgtd   
```

使用tgt-admin命令查看运行状态:

```sh
[root@server2 ~]# tgt-admin --show
Target 1: iqn.2000-01.server2:mydisk
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
        I_T nexus: 2
            Initiator: iqn.1994-05.com.redhat:bf38cf4ccf5 alias: server3
            Connection: 0
                IP Address: 10.1.1.2
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 310 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/centos/iscsi01
            Backing store flags: 
        LUN: 2
            Type: disk
            SCSI ID: IET     00010002
            SCSI SN: beaf12
            Size: 200 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/sda3
            Backing store flags: 
        LUN: 3
            Type: disk
            SCSI ID: IET     00010003
            SCSI SN: beaf13
            Size: 105 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /srv/iscsi/disk.img
            Backing store flags: 
    Account information:
        iuser
    ACL information:
        10.1.1.0/24
```

可以看到和配置文件设定的一致,并且有一个已经连接的主机10.1.1.2.



## iSCSI initiator设置

安装好iscsi-initiator-utils之后,修改配置文件/etc/iscsi/iscsid.conf内容,主要是设置访问账号密码:

```sh
[root@server3 ~]# vi /etc/iscsi/iscsid.conf
node.session.auth.username = iuser
node.session.auth.password = ipass
discovery.sendtargets.auth.username = iuser
discovery.sendtargets.auth.password = ipass
```



## iSCSI initiator操作

设置好之后,使用iscsiadm命令来检测10.1.1.1中的iSCSI设备:

```sh
[root@server3 ~]# iscsiadm -m discovery -t sendtargets -p 10.1.1.1
10.1.1.1:3260,1 iqn.2000-01.server2:mydisk
```

启动iscsi程序:

```sh
[root@server3 ~]# systemctl start iscsi
```

登录目标target:

```sh
[root@server3 ~]# iscsiadm -m node
10.1.1.1:3260,1 iqn.2000-01.server2:mydisk
[root@server3 ~]# iscsiadm -m node -T iqn.2000-01.server2:mydisk --login
Logging in to [iface: default, target: iqn.2000-01.server2:mydisk, portal: 10.1.1.1,3260] (multiple)
Login to [iface: default, target: iqn.2000-01.server2:mydisk, portal: 10.1.1.1,3260] successful.
```

查询iSCSI磁盘:

```sh
[root@server3 ~]# fdisk -l

Disk /dev/sdb: 310 MB, 310378496 bytes, 606208 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 200 MB, 200278016 bytes, 391168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdd: 104 MB, 104857600 bytes, 204800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

注销target,但并不删除连接:

```sh
[root@server3 ~]# iscsiadm -m node -T iqn.2000-01.server2:mydisk --logout
Logging out of session [sid: 1, target: iqn.2000-01.server2:mydisk, portal: 10.1.1.1,3260]
Logout of [sid: 1, target: iqn.2000-01.server2:mydisk, portal: 10.1.1.1,3260] successful.
```

删除target连接:

```sh
[root@server3 ~]# iscsiadm -m node -o delete -T iqn.2000-01.server2:mydisk
[root@server3 ~]# iscsiadm -m node
iscsiadm: No records found
```

直接将target中的磁盘组成LV并格式化:

```sh
[root@server3 ~]# pvcreate /dev/sd{b,c,d}
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
[root@server3 ~]# vgcreate iscsi /dev/sd{b,c,d}
  Volume group "iscsi" successfully created
[root@server3 ~]# vgdisplay
[root@server3 ~]# lvcreate -l 144 -n disk iscsi
  Logical volume "disk" created.
[root@server3 ~]# lvdisplay
[root@server3 ~]# mkfs -t ext4 /dev/iscsi/disk
[root@server3 ~]# mkdir -p /data/iscsi
[root@server3 ~]# echo "/dev/iscsi/disk /data/iscsi ext4 defaults,_netdev 1 2" >> /etc/fstab
[root@server3 ~]# mount -a
[root@server3 ~]# df -Th
/dev/mapper/iscsi-disk  ext4      551M  876K  510M   1% /data/iscsi
```

挂载时使用_netdev参数的意思是,开机后需要网络启动完毕后再挂载.

