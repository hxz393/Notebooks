# WEB服务器

## URL网址

浏览器中使用网址(URL, Uniform Resource Locator)来定位服务器中数据存放的位置并访问.URL格式如下:`<协议>://<主机地址或主机名>[:port]/<目录资源>`.以斜线作为分段,各段解释如下:

- **协议**

  浏览器支持比较常见的协议有http, https, ftp, telnet等,还有不太常见的news, gopher等.如果服务器启用的是非标准端口,必须在主机地址后面写明.

- **主机地址或主机名**

  主机地址是服务器的IP地址,主机名是域名,通过DNS来解析成IP.

- **目录资源**

  在Apache中默认将html文件存放在/var/www/html/目录内.



## 传输数据方法

在http的服务端和客户端之间数据传递的基本方法有下面几种:

- **GET**

  最常用的方法,浏览器直接向服务器请求网址上面的资源.使用GET方式可以直接在URL中输入变量.

- **POST**

  客户端向服务端发送请求的方法.

- **HEAD**

  服务端响应给客户端的一些数据头文件.

- **OPTIONS**

  服务端响应给客户端一些允许的功能与方法.

- **DELETE**

  删除某些资源的方法.



## SSL证书

SSL(Secure Socket Layer)是HTTPS使用的加密方式之一,和SSH一样使用非对称的密钥对,用对方的公钥加密数据后传输,以此保证数据在传输过程中的安全.

为了加强网络安全,提高网站信任等级,网站可以把生成的密钥通过第三方公证单位(CA, Certificate Authorities)注册,客户端的浏览器访问时可以自动比对CA单位注册时信息,确认证书是否正确有效.



## LAMP服务器安装

由于Linux上面Apache仅能提供最基本的静态网站支持,要想实现动态网站需要PHP和MySQL的支持.整个LAMP环境总共需要安装以下软件与模块: 

- httpd: 提供Apache主程序.
- mysql: MySQL客户端程序.
- mysql-server: MySQL服务端程序.
- php: PHP主程序,包含给Apache使用的模块.
- php-devel: PHP开发工具,与PHP外挂的加速软件有关.
- php-mysql: 提供PHP程序读取MySQL数据库的模块.

整合安装如下:

```sh
[root@server2 ~]# yum install -y httpd mysql mysql-server php php-mysql
```



## 相关配置目录

### Apache

在Apache中主要的配置文件或目录如下:

- **/etc/httpd/conf/httpd.conf**

  httpd最主要的配置文件,有时也会被拆成数个小文件分别管理不同参数.

- **/etc/httpd/conf.d/*.conf**

  可以将自定义额外参数单独写成配置文件,放到/etc/httpd/conf.d目录下.在httpd启动时,这个文件内容就会被读入主要配置文件当中.

- **/usr/lib64/httpd/modules/, /etc/httpd/modules/**

  Apache支持很多外挂模块,PHP和SSL都是Apache外挂的一种.所有要使用的模块文件默认放到以上目录中.

- **/var/www/html/**

  默认存放html数据文件位置.

- **/var/www/error/**

  当服务器设置错误或客户端请求的数据错误时,在浏览器上显示此目录下对应的错误页面.

- **/var/www/icons/**

  存放一些Apache提供的小icon文件.

- **/var/www/cgi-bin/**

  给一些可执行的CGI程序放置的目录.

- **/var/log/httpd/**

  默认的Apache日志文件存放目录,需要注意用量大小.

### MySQL

MySQL相关的配置文件和目录有:

- **/etc/my.cnf**

  MySQL的配置文件,可以对MySQL数据库进行优化或指定额外的运行参数.

- **/var/lib/mysql/**

  MySQL数据库文件存放位置.

### PHP

在LAMP环境中和PHP有关的配置和目录有:

- **/etc/httpd/conf.d/php.conf**

  PHP设置参数存放此文件中,在Apache重新启动时会被读取,因此不需要手动将php模块写入httpd.conf中.

- **/etc/php.ini**

  PHP的主要配置文件,包括是否允许用户上传,能否允许某些低安全性的标志等.

- **/usr/lib64/httpd/modules/libphp5.so**

  PHP提供给Apache使用的模块.

- **/etc/php.d/mysql.ini, /usr/lib64/php/modules/mysql.so**

  PHP操作MySQL使用的模块.

- **/usr/bin/phpize, /usr/include/php/**

  安装PHP加速器所需文件.

  

## Apache配置文件

apache配置文件/etc/httpd/conf/httpd.conf中分以下几个部分.

### 服务器环境设置

针对服务器环境设置方面,包括主机名,服务器配置文件顶层目录等:

- **ServerRoot "/etc/httpd"**

  服务器设置的最顶层目录,包括日志,模块等数据都放于此处(链接文件).

- **ServerTokens OS**

  告知客户端服务器的版本和操作系统,一般不需要设置.

- **PidFIle run/httpd.pid**

  设置PID文件路径.

- **Timeout 60**

  发送或接受数据时的等待超时时间,一般不需要设置.

- **KeepAlive On**

  是否允许持续性的连接,也就是一个TCP连接可以有多个文件传送请求,默认为Off.

- **MaxKeepAliveRequests 500**

  设置能够传输的最大传输数量,0代表不限制.

- **Keep Alive Timeout 15**

  在设置保持连接的条件下,连接在最后一次传输后等待延迟的秒数.

- **Listen 80**

  默认监听80端口,也可以修改绑定网口或端口.

- **User apache**, **Group apache**

  设置httpd进程启动的属主和属组.

- **ServerAdmin root@localhost**

  设置管理员邮箱.

- **ServerName www.example.com:80**

  设置主机名.

### 目录相关设置

默认网页存放目录/var/www/html由DocumentRoot设置.需要注意SELinux相关规则和类型的正确性.

首页文件名由<IfModule dir_module>内的DirectoryIndex设置.文件读取由文件名参数顺序决定.

和目录有关的设置写在<Directory\>段落内,主要有下面一些设置:

- **Options**(目录参数)

  此设置值表示这个目录内能让Apache进行的操作,也就是震度Apache的程序权限设置.主要参数有:

  - Indexes: 如果找不到首页文件,便显示整个目录下的文件名.
  - FollowSymLins: 让连接文件可以生效.
  - ExecCGI: 让此目录具有执行CGI程序的权限.
  - Includes: 让一些Server-Side Include程序可以运行.
  - MultiViews: 与多语言支持有关.

- **AllowOverride**(允许覆盖参数)

  表示是否允许额外配置文件.htaccess的某些参数覆盖.常见参数有:

  - ALL: 所有权限均可被覆盖.
  - AuthConfig: 仅有网页认证账号可覆盖.
  - Indexes: 仅允许Indexes方面的覆盖.
  - Limits: 允许用户利用Allow, Deny与Order管理可浏览的权限.
  - None: 不可覆盖.

### PHP模块参数修改

PHP模块配置文件位于/etc/httpd/conf.d/php.conf,其内容不需要有任何修改,主要修改的是PHP配置文件/etc/php.ini:

- log_errors = On, ignore_repeated_errors = Off, ignore_repeated_source = Off

  设置记录日志,忽略重复错误记录默认为Off,可以设置为On.

- display_errors = Off, display_startup_errors = Off

  设置出现错误时,是否在浏览器上显示相关错误信息.debug时可以设置为On.

- post_max_size = 8M, file_uploads = On, upload_max_filesize = 2M, memory_limit = 128M

  设置PHP程序上传文件大小限制和内存占用数值.



## 启动服务

使用systemctl直接启动httpd,或者可以用apache自带的apachectl命令操作:

```sh
[root@server2 ~]# systemctl start httpd
[root@server2 ~]# systemctl enable httpd
[root@server2 ~]# apachectl status
```

测试PHP模块是否驱动,可以建立一个phpinfo.php文件:

```sh
[root@server2 ~]# echo '<?php phpinfo (); ?>' > /var/www/html/phpinfo.php
```

在浏览器中使用http://192.168.2.254/phpinfo.php地址访问,可以看到php版本等信息.

MySQL需要手动启动,使用mysql命令测试连接:

```sh
[root@server2 ~]# systemctl start mysqld
[root@server2 ~]# mysqladmin -u root pass 'mypass'
[root@server2 ~]# mysql -u root -p mypass
```



## 防火墙设置

iptables放行80和443端口设置如下:

```sh
[root@server2 ~]# iptables -A INPUT -p TCP -i ens33 --dport 80 --sport 1024:65534 -j ACCEPT
[root@server2 ~]# iptables -A INPUT -p TCP -i ens33 --dport 443 --sport 1024:65534 -j ACCEPT
```

设置SELinux的放行规则:

```sh
[root@server2 ~]# setsebool -P httpd_can_network_connect=1
```

新建一个网页文件后需要恢复SELinux类型:

```sh
[root@server2 ~]# echo "HOME" > index.html
[root@server2 ~]# mv index.html /var/www/html
[root@server2 ~]# ll -Z /var/www/html/
-rw-r--r--. root root unconfined_u:object_r:admin_home_t:s0 index.html
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 phpinfo.php
[root@server2 ~]# restorecon -Rv /var/www/html/
```



## 基本身份认证

Apache可以通过目录下的.htaccess文件来设置保护目录.首先建立目录和文件:

```sh
[root@server2 ~]# mkdir /var/www/html/pro
[root@server2 ~]# echo 'PROTECT!' > /var/www/html/pro/index.html
```

在主配置文件/etc/httpd/conf/httpd.conf中,增加一段配置:

```sh
[root@server2 ~]# vi /etc/httpd/conf/httpd.conf
<Directory "/var/www/html/pro">
    AllowOverride AuthConfig
    Require all granted
</Directory>
[root@server2 ~]# apachectl restart
```

在/var/www/html/pro目录下建立.htaccess文件:

```sh
[root@server2 httpd]# vi /var/www/html/pro/.htaccess
AuthName        "protect test"
Authtype        Basic
AuthUserFile    /var/www/apache.passwd
require user user11
```

文件配置选项说明如下:

- AuthName: 出现在账号密码提示对话框中的内容.
- AuthType: 认证类型.
- AuthUserFile: 账号密码配置文件.
- require: 设置通行账号.可以设为valid-user,即所有在apache.passwd内定义的用户都可以访问.

下面是建立账号密码文件.使用htpasswd命令加入账号user11和user12,密码为passwd:

```sh
[root@server2 ~]# htpasswd -c /var/www/apache.passwd user11
New password: 
Re-type new password: 
Adding password for user user11
[root@server2 ~]# htpasswd /var/www/apache.passwd user12
New password: 
Re-type new password: 
Adding password for user user12
```

在浏览器中输入地址http://192.168.2.254/pro/测试.



## 虚拟主机设置

虚拟主机就是让多个主机名对应到不同的主网页目录(DocumentRoot 参数).例如ftp.hxz.ass和www.hxz.ass都指向192.168.2.254,需要在DNS服务商网页设置好对应关系.

这里新建一个ftp目录,来对应ftp.hxz.ass域名:

```sh
[root@server2 ~]# mkdir /var/www/ftp
[root@server2 ~]# echo "FTPftp" > /var/www/ftp/index.html
```

配置对应的虚拟主机设置文件,然后重启httpd服务:

```sh
[root@server2 ~]# vi /etc/httpd/conf.d/virtual.conf
<Directory "/var/www/ftp">
    AllowOverride None
    Require all granted
</Directory>

<VirtualHost *:80>
    ServerName www.hxz.ass
    DocumentRoot /var/www/html
</VirtualHost>
<VirtualHost *:80>
    ServerName ftp.hxz.ass
    DocumentRoot /var/www/ftp
    CustomLog /var/log/httpd/ftp.acess_log combined
</VirtualHost>
[root@server2 ~]# systemctl restart httpd
```

上面使用CustomLog来指定写入日志的路径,这样可以更方便管理.

在客户端测试一下:

```sh
[root@server3 ~]# curl www.hxz.ass
HOME
[root@server3 ~]# wget ftp.hxz.ass
--2021-10-16 23:22:07--  http://ftp.hxz.ass/
Resolving ftp.hxz.ass (ftp.hxz.ass)... 10.1.1.1
Connecting to ftp.hxz.ass (ftp.hxz.ass)|10.1.1.1|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7 [text/html]
Saving to: 'index.html'
```



## 服务器测试

Apache提供了一个叫做ab的程序来测试网站效率.例如指定110个同时连接,200个每连接线程测试phpinfo.php:

```sh
[root@server2 ~]# ab -dkS -c110 -n200 http://127.0.0.1/phpinfo.php
This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 127.0.0.1 (be patient)
Completed 100 requests
Completed 200 requests
Finished 200 requests


Server Software:        Apache/2.4.6
Server Hostname:        127.0.0.1
Server Port:            80

Document Path:          /phpinfo.php
Document Length:        47649 bytes

Concurrency Level:      110
Time taken for tests:   0.070 seconds
Complete requests:      200
Failed requests:        17
   (Connect: 0, Receive: 0, Length: 17, Exceptions: 0)
Write errors:           0
Keep-Alive requests:    0
Total transferred:      9566380 bytes
HTML transferred:       9529780 bytes
Requests per second:    2863.57 [#/sec] (mean)
Time per request:       38.414 [ms] (mean)
Time per request:       0.349 [ms] (mean, across all concurrent requests)
Transfer rate:          133759.55 [Kbytes/sec] received

Connection Times (ms)
              min   avg   max
Connect:        0     3    8
Processing:    12    22   25
Total:         12    25   33
```



## 日志分析

Apache日志文件有两个:/var/log/httpd/access_log和/var/log/httpd/error_log.由于日志文件增速很快,所以需要修改logrotate的httpd配置文件,加入压缩功能:

```sh
[root@server2 ~]# vi /etc/logrotate.d/httpd
/var/log/httpd/*log {
    missingok
    notifempty
    sharedscripts
    compress
    delaycompress
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
    endscript
}
```

日志文件分析软件可以使用webalizer,软件配置文件位于/etc/webalizer.conf,生成的图标位于/var/www/usage:

```sh
[root@server2 ~]# yum install -y webalizer
[root@server2 ~]# vi /etc/webalizer.conf
OutputDir      /var/www/html/pro/
[root@server2 ~]# webalizer
Warning: Truncating oversized hostname
Skipping bad record (1)
```

之后就可以通过http://192.168.2.254/pro/来访问了.

另外一个日志文件分析软件是awstats,它采用Perl语言编写,所以需要mod_perl的支持:

```sh
[root@server2 ~]# yum install -y awstats
[root@server2 ~]# vi /etc/httpd/conf.d/awstats.conf 
<Directory "/usr/share/awstats/wwwroot">
    Options None
    AllowOverride AuthConfig
    <IfModule mod_authz_core.c>
        # Apache 2.4
        Require all granted
    </IfModule>
    <IfModule !mod_authz_core.c>
        # Apache 2.2
        Order allow,deny
        Allow from all
        Allow from ::1
    </IfModule>
</Directory>
```

配置文件中修改了CGI运行方式,网页认证,允许访问的客户端地址.另外还需要配置针对域名的配置文件:

```sh
[root@server2 ~]# cp /etc/awstats/awstats.model.conf /etc/awstats/awstats.www.conf
[root@server2 ~]# vi /etc/awstats/awstats.www.conf
SiteDomain="www.hxz.ass"
```

测试生成分析资料与访问账号:

```sh
[root@server2 ~]# cd /usr/share/awstats/wwwroot/cgi-bin/
[root@server2 cgi-bin]# ./awstats.pl now
[root@server2 wwwroot]# vi /usr/share/awstats/wwwroot/.htaccess
AuthName        "awstats test"
Authtype        Basic
AuthUserFile    /var/www/apache.passwd
require         valid-user
```

之后浏览器通过http://192.168.2.254/awstats/awstats.pl?config=www并输入账号密码之后,就可以访问awstats的报表了.



## HTTPS加密

加密需要通过mod_ssl来进行,可以使用yum安装:

```sh
[root@server2 ~]# yum install -y mod_ssl
```

安装好以后就可以通过浏览器访问https://192.168.2.254/了.和SSL加密有关的文件如下:

- /etc/httpd/conf.d/ssl.conf: 软件mode_ssl提供的Apache配置文件.
- /etc/pki/tls/private/localhost.key: 系统私钥文件,可以用来制作证书.
- /etc/pki/tls/certs/localhost.crt: 加密过的证书文件.

如果要自建证书,例如建立一个名为hxz.ass的证书,步骤如下:

```sh
[root@server2 ~]# cd /etc/pki/tls/certs/
[root@server2 certs]# make my.key
[root@server2 certs]# openssl rsa -in my.key -out hxz.ass.key
Enter pass phrase for my.key:
writing RSA key
[root@server2 certs]# rm -rf my.key
[root@server2 certs]# chmod 400 hxz.ass.key 
[root@server2 certs]# make hxz.ass.crt SERIAL=20210101
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:CN
Locality Name (eg, city) [Default City]:CN
Organization Name (eg, company) [Default Company Ltd]:CN
Organizational Unit Name (eg, section) []:CN
Common Name (eg, your name or your server's hostname) []:www.hxz.ass
Email Address []:admin@hxz.ass
[root@server2 certs]# ll hxz*
-rw-------. 1 root root 1379 Oct 17 04:08 hxz.ass.crt
-r--------. 1 root root 1679 Oct 17 04:06 hxz.ass.key
```

最终生成的证书是hxz.ass.crt,对应私钥为hxz.ass.key.其有效期为一年,如果想要十年,可以修改make脚本中的内容.

修改ssl.conf中证书和私钥的位置:

```sh
[root@server2 certs]# vi /etc/httpd/conf.d/ssl.conf 
SSLCertificateFile /etc/pki/tls/certs/hxz.ass.crt
SSLCertificateKeyFile /etc/pki/tls/certs/hxz.ass.key
```

之后重启httpd服务,并在浏览器中刷新查看新的证书.

