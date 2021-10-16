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

  默认监听80端口,也可以修改端口或绑定网口.

- **User apache**, **Group apache**

  设置httpd进程启动的属主和属组.

- **ServerAdmin root@localhost**

  设置管理员邮箱.

- **ServerName www.example.com:80**

  设置主机名.

### 目录相关设置

默认网页存放目录/var/www/html由DocumentRoot设置.需要注意SELinux相关规则和类型的正确性.

和目录有关的设置写在\<Directory\>段落内,主要有下面一些设置:

- Options(目录参数)

  

