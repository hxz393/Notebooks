# Bash基础

## Shell的定义

Shell的功能只是提供用户操作的一个接口,因此shell需要可以调用其他软件,用户通过shell来操作这些应用程序.也就是只要能够操作应用程序的接口都能够称为shell.



## Shell的分类

可以通过检查/etc/shells来得知当前系统可用的shell:

```sh
[root@101c7 4]# cat /etc/shells 
/bin/sh
/bin/bash
/usr/bin/sh
/usr/bin/bash
```

passwd文件里记录了用户能使用的shell:

```sh
[root@101c7 4]# cat /etc/passwd | grep root
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
```



## bash特点

bash(Bourne Again Shell)的主要特点有:

- **命令记忆能力**: 在命令行中按上下键就能找到输入过的命令,默认记录最大1000条

- **命令补全功能**: 通过[Tab]按键能自动补全命令或文件名

- **命令别名功能**: 可以使用alias命名一些自定义命令

- **作业后台控制**: 可以将工作丢到后台执行

- **使用程序脚本**: 可以将很多命令写成一个文件执行

- **可使用通配符**: 支持常用的通配符比如*和?




## 命令类型

bash已经内置了很多命令,可以使用type命令来查询命令类型.内建命令和外部命令的区别在于前者不需要使用子程序来进行,它们已经和shell编译成一体,作为shell工具的组成部分存在.因此内建命令执行速度更快,效率越高.

有些命令两种实现都有,which命令只显示出外部命令文件,这种情况下要使用外部命令文件直接使用绝对路径运行.

例如查询man和cd命令:

```sh
[root@101c7 4]# type ls
ls is aliased to `ls --color=auto'
[root@101c7 4]# type man
man is /usr/bin/man
[root@101c7 4]# type cd
cd is a shell builtin
```

结果显示man是外部命令,cd是内部命令,ls有别名.

对于别名命令可以用-a参数进一步查询:

```sh
[root@101c7 4]# type -a ls
ls is aliased to `ls --color=auto'
ls is /usr/bin/ls
```



## bash环境中特殊符号

理论上文件名中不要使用以下已被特殊定义的字符:

| **符号** | **内容**                                      |
| -------- | --------------------------------------------- |
| #        | 批注符号,一般在脚本中拿来注释说明行           |
| **       | 转义符号,将特殊字符还原成一般字符             |
| \|       | 管道(pipe),分隔两个管道命令的界定             |
| ;        | 连续命令执行分隔符                            |
| ~        | 用户的主文件夹                                |
| $        | 使用变量前导符,即是变量之前需要加的变量替代值 |
| &        | 作业控制(job control),将命令放置后台工作      |
| !        | 逻辑运算意义的非(not)                         |
| /        | 目录符号,路径分隔的符号                       |
| >, >>    | 数据流重定向,输出导向,代表替换与追加          |
| <, <<    | 数据流重定向,输入导向                         |
| ' '      | 单引号,不具有变量置换功能                     |
| " "      | 双引号,具有变量置换功能                       |
| \` \`    | 套在可以先执行的命令两边,等同于$()            |
| ( )      | 在中间为子shell的起始与结束                   |
| { }      | 在中间为命令块的组合                          |



## 命令回传码

可以通过命令回传码来判断后续命令是否执行,在两条命令间用&&或||连起来.具体意义如下:

| **执行的命令** | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| cmd1 && cmd2   | 若cmd1执行完毕且正确执行($?=0),继续执行cmd2.否则(\$?≠0)不执行cmd2 |
| cmd1 \|\| cmd2 | 若cmd1执行完毕且正确执行($?=0),不执行cmd2.否则(\$?≠0)继续执行cmd2 |

例如判断目录/tmp/abc存在时执行cd命令:

 ```sh
 [root@101c7 ~]# ls /tmp/abc && cd /tmp/abc
 ls: cannot access /tmp/abc: No such file or directory
 ```

结果只有ls报错,说明并没有执行后面的cd命令.

例如判断目录/tmp/abc不存在时自动新建目录并进入:

```sh
[root@101c7 ~]# ls /tmp/abc || mkdir /tmp/abc && cd /tmp/abc ; pwd
ls: cannot access /tmp/abc: No such file or directory
/tmp/abc
```

假如目录/tmp/abc存在时,回传为0,不执行mkdir命令,&?=0继续向后传,顺利执行cd命令.

如果将||和&&后面的命令调转一下位置,先进行&&判断,那么在文件夹不存在的情况下,cd命令不会运行,但会执行mkdir命令,没有达到目标.因此命令执行的顺序很重要.



## 标准输入输出

标准输出(stdout,Standard Output)与标准错误输出(stderr, Standard Error Output)指的是命令执行所回传的正常与报错信息.默认情况下都是输出到屏幕上.

标准输入(stdin, Standard Input)一般接受的是用户键盘输入数据.



## 管道命令

管道命令使用|符号界定用途时,将前一个命令的标准输出(stdout,不能处理stderr)转给后一命令做标准输入.

每个管道后面接的第一个数据必须是命令,并且这个命令必须能接受stdin的数据.例如less,tail,grep等.



## 减号-的用途

在管道命令中,经常会使用到前一个命令的stdout作为这次的stdin,某些命令需要用到文件名(例如tar)来进行处理时,该stdin与stdout可以利用减号-来代替.



## 子shell

把命令用括号括起来(可以是连续命令)后,命令列表就成为了进程列表.其作用表示使用一个子shell来执行对应命令.

使用$BASH_SUBSHELL变量可以检测是否在子shell中运行:

```sh
[root@server1 ~]# pwd ; echo $BASH_SUBSHELL 
/root
0
[root@server1 ~]# (pwd ; echo $BASH_SUBSHELL)
/root
1
[root@server1 ~]# (pwd ; (echo $BASH_SUBSHELL))
/root
2
```

创建子shell可以嵌套运行,一般用来进行多进程处理,例如将程序置入后台运行.但是采用子shell的代价太高,必须为子shell创建出一个全新的环境,会明显拖慢处理速度.

另一个可以调用子shell的方式是使用协程,它能在后台生成一个子shell并在这个子shell中执行命令:

```sh
[root@server1 ~]# coproc my_sleep { sleep 100; }
[1] 41326
[root@server1 ~]# jobs
[1]+  Running                 coproc my_sleep { sleep 100; } &
```



## 数据流重定向

数据流重定向可以将stdout和stderr分别传送到其他的文件或设备中去,将由stdin的数据改为文件内容来替代输入:

| 类型                 | 表示               |
| -------------------- | ------------------ |
| 标准输入(stdin)      | 代码0, 使用<或<<;  |
| 标准输出(stdout)     | 代码1, 使用>或>>;  |
| 标准错误输出(stderr) | 代码2, 使用2>或2>> |

例如将根目录文件列表输出到root.txt文件中:

```sh
[root@101c7 ~]# ls -la / > root.txt
[root@101c7 ~]# tail root.txt 
drwxr-xr-x.   2 root root    6 Apr 11  2018 opt
dr-xr-xr-x. 227 root root    0 Sep  9 15:08 proc
```

将/root目录信息追加到root.txt文件中:

```sh
[root@101c7 ~]# ls -la ~ 1>> root.txt
[root@101c7 ~]# ll | grep root.txt 
-rw-r--r--. 1 root root      2640 Sep 11 10:05 root.txt
```

也可以使用1>来定义标准输出,因为默认就是代码1,要输出标准错误,使用2>:

```sh
[root@101c7 ~]# cat xx 2>> root.txt ; tail -3 root.txt
cat: xx: No such file or directory
-rw-r--r--.  1 root root       129 Dec 28  2013 .tcshrc
-rw-r--r--.  1 root root  20971520 Jul  7 04:07 TinyCore-current.iso
```

将标准输出和标准错误输出分别输出到不同文件:

```sh
[root@101c7 ~]# find / -name .bashrc > list.txt 2> list_error.txt
```

将标准输出和标准错误输出到同一个文件可以用2>&1,也可以用&>来操作:

 ```sh
 [root@101c7 ~]# find / -name .bashrc > list.txt 2>&1
 [root@101c7 ~]# find / -name .bashrc &> list.txt 
 ```

也可以将标准错误输出直接丢弃:

```sh
[root@101c7 ~]# find / -name .bashrc > list.txt 2> /dev/null
```

将list.txt文件的内容作为cat命令的输入来生成文件catfile:

```sh
[root@101c7 ~]# cat > catfile < /root/list.txt ; head catfile
/etc/skel/.bashrc
/root/.bashrc
```

<<代表的含义是从标准输入接收数据,遇到了关键词则结束输入.例如下面设的结束关键词"end",输入end再敲击回车后,输入立马结束,不同于Ctrl+d结束输入,end这个关键词不会被记录到文件中,仅仅作为结束标记使用:

```sh
[root@101c7 ~]# cat > catfile << "end"
> yes it/'s begin
> line 2
> end
[root@101c7 ~]# cat catfile 
yes it/'s begin
line 2
```



## 双向重定向

tee命令可以将单向传输的数据流半路截取同时移作他用.

例如将ls命令的结果追加到root.txt文件末尾并排序打印出来:

```sh
[root@101c7 ~]# ls -la | tee -a root.txt | sort | head -3; tail -3 root.txt 
drwx------.  2 root root        52 Sep 11 04:07 audit
drwxr-----.  3 root root        19 Sep  7 05:51 .pki
drwxr-xr-x.  2 root root        23 Sep 10 13:03 etc
-rw-r--r--.  1 root root       129 Dec 28  2013 .tcshrc
-rw-r--r--.  1 root root  20971520 Jul  7 04:07 TinyCore-current.iso
-rw-------.  1 root root      3358 Sep 11 04:33 .viminfo
```



## 参数代换

xargs用来产生某个命令的参数.xargs可以读入stdin的数据,并以空格符或断行字符进行分辨,将stdin的数据分隔成为参数.主要是因为很多命令并不支持管道命令,才需要使用xargs.

基本用法:`xargs [-0epn] command`

参数:

| 参数 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| -0   | 如果输入的stdin含有特殊字符比如`,\,空格等,这个参数可以将它还原成一般字符; |
| -e   | 这个是EOF的意思,后面可以接一个字符串,当解析到这个字符串时就会停止继续工作; |
| -p   | 在执行每个命令参数时,都会询问用户;                           |
| -n   | 后面接次数,每次command命令执行时,要使用几个参数.             |
|      | 当xargs后面没有接任何的命令时,默认以echo来进行输出           |

例如将/etc/passwd第一列取出,只取三行,使用finger命令将账号内容显示出来:

```sh
[root@101c7 sdb4m]# cut -d ':' -f 1 /etc/passwd | head -3 | xargs finger
Login: root                             Name: root
Directory: /root                        Shell: /bin/bash
On since Sat Sep 11 09:35 (EDT) on tty1    3 hours 14 minutes idle
On since Sat Sep 11 07:08 (EDT) on pts/0 from 192.168.2.101
```

finger命令用来查询账户说明信息,xargs将三个账号名称传给finger作为参数

将passwd中所有账号都用finger查询,但一次只查询5个:

```sh
[root@101c7 sdb4m]# cut -d ':' -f 1 /etc/passwd | xargs -p -n 5 finger
finger root bin daemon adm lp ?...
finger sync shutdown halt mail operator ?...
```

同上,当参数分析到lp就结束命令:

```sh
[root@101c7 sdb4m]# cut -d ':' -f 1 /etc/passwd | xargs -p -e'lp' finger
finger root bin daemon adm ?...
```

注意-e与'lp'之间没有空格.

 

