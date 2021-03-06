# NFS服务

## NFS服务器

NFS为Network File System的简称,目的是想让不同机器不同操作系统可以彼此共享数据文件.NFS服务器可以让主机将网络中NFS服务器共享的目录挂载到本地端的文件系统中,用起来就像本地分区一样.

NFS服务端口是2049,此外用来传输的端口是小于1024号的随机端口,由RPC协议来调用.

远程过程调用(RPC, Remote Procedure Call)服务主要功能是指定每个NFS功能所对应的端口号,并通知给客户端.端口号由NFS启动时随机选取并向RPC注册,因此RPC服务需要先于NFS启动.

RPC服务使用固定的111端口来监听客户端需求.使用NFS时,客户端和服务端都需要启动RPC服务,实际上称NFS为RPC服务的一种.

NFS所依赖的RPC服务如下:

- rpc.nfsd: 最主要的NFS服务提供程序,管理客户端是否能够使用服务器文件系统挂载信息等.
- rpc.mountd: 用于管理NFS文件系统的权限,它会去读NFS配置文件/etc/exports来对比客户端权限.
- rpc.lockd: 用于管理文件锁定功能.需要服务端和客户端同时开启,与rpc.statd有依赖关系,非必须启动.
- rpc.statd: 用来检查文件的一致性,如果文件被多客户端同时使用造成损坏时,它可以检测并尝试恢复文件.



## NFS服务配置

要想启动NFS服务器,需要具备下面两个程序:

- rpcbind: RPC的主程序,用来做好端口对应的工作.
- nfs-utils: 提供rpc.nfsd及rpc.mountd服务的程序

默认情况下没有安装,使用yum来安装:

```sh
[root@server2 ~]# yum install -y rpcbind
[root@server2 ~]# yum install -y nfs-utils
```

NFS的配置文件存在/etc/exports,日志文件放在/var/lib/nfs/路径下.

例如将/tmp目录共享给主机192.168.2.234和10.1.1.0/24,将/root/nf目录分享给所有人:

```sh
[root@server2 ~]# vi /etc/exports
/tmp 192.168.2.234(ro) 10.1.1.0/24(rw)
/root/nf *(rw,no_root_squash)
```

括号内的权限设置有下面几种组合,可以用逗号,分隔指定多个权限:

- rw, ro: 目录共享权限为可读写(read-write)或只读(read-only).还需要参考文件系统权限和身份.
- sync, async: 数据同步或异步写入到硬盘.
- no_root_squash, root_squash: root_squash表示将root身份替换成nfsnobody.否则开放root身份权限.
- all_squash: 将所有用户都压缩成匿名用户(nfsnobody).
- anonuid=, anongid=: 设置匿名用户UID或GID值.

一般NFS服务不会对互联网开放,如果遇到特殊要求,可以调整配置/etc/sysconfig/nfs让除了111和2049号外的端口固定,以方便防火墙设置规则.



## NFS服务管理

改好配置文件后就可以启动NFS服务:

```sh
[root@server2 ~]# systemctl start rpcbind
[root@server2 ~]# systemctl start nfs
```

可以使用rpcinfo来查询本机RPC状态:

```sh
[root@server2 ~]# rpcinfo -p
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
```

修改配置文件后,可以使用exportfs命令重新挂载NFS文件系统:

```sh
[root@server2 ~]# exportfs -arv
exporting 192.168.2.234:/tmp
exporting 10.1.1.0/24:/tmp
exporting *:/root/nf
```

卸载已经共享的NFS目录也使用exportfs命令操作:

```sh
[root@server2 ~]# exportfs -auv
```



## NFS目录挂载

可以使用showmount来查询NFS服务器共享目录情况:

```sh
[root@server2 ~]# showmount -e 10.1.1.1
Export list for 127.0.0.1:
/root/nf *
/tmp     10.1.1.0/24,192.168.2.234
```

客户端可以使用mount命令直接挂载使用共享目录:

```sh
[root@server1 ~]# mount 192.168.2.254:/tmp 111
[root@server1 ~]# ll
total 8992
drwxrwxrwt. 8 root root     172 Oct 10 14:52 111
[root@server1 ~]# df
Filesystem              1K-blocks    Used Available Use% Mounted on
192.168.2.254:/tmp        1020928   33280    987648   4% /root/111
```

卸载NFS目录使用umount命令:

```sh
[root@server1 ~]# umount 111
```

另外客户端在挂载NFS文件系统时,可以使用一些特殊的挂载参数:

- fg, bg: 执行挂载时,挂载行为在前台或后台执行.默认是前台,在后台执行挂载不会影响到前台工作.
- soft, hard: 默认hard参数,在主机脱机时RPC会持续尝试连接,使用soft参数则会等待超时时间后再尝试.
- intr: 如果采用hard方式挂载,使用intr参数可以中断尝试连接.
- rsize, wsize: 读写区块大小,默认1024bytes.调大可以加速网络传输速度.

例如采用后台软挂载,调整读写区块大小为32K:

```sh
[root@server1 ~]# mount -t nfs -o bg,soft,rsize=32768,wsize=32768 10.1.1.1:/root/nf 111
```

想要开机挂载可以将上面的命令写入到/etc/rc.d/rc.local文件中,并加入运行权限.

