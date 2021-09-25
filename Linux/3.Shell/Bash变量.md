# Bash变量

## 变量定义

变量简单定义就是让某一个特定字符串代表不固定内容.



## 显示变量

可以使用echo $变量名 来查看变量.将变量加上{}效果是一样的:

```sh
[root@101c7 4]# echo {$PATH}
{/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin}
```

这个特性可以让字符串扩展变量.例如用echo显示my ${rev}th version.



## 设置变量

直接用等号连接变量和内容就行了:

```sh
[root@101c7 4]# myhome=ext333
[root@101c7 4]# echo $myhome
ext333
```

使用myhome='ext333'的格式也能设置变量.通常变量值有带空格或特殊符号要这样设置.

注意等号两边如果有空格表示等量关系测试.



## 取消变量

使用`unset 变量名`取消变量设置:

```sh
[root@101c7 4]# unset myhome
```

记住删除环境变量时,不用带\$.所有使用变量时用\$,操作变量不使用\$.



## 变量设置规则

- 等号两边不能直接接空格符,变量内容有空格可使用双引号或单引号括起来;

- 变量名只能使用英文字母和数字,不能以数字开头;

- **双引号**内特殊字符可以保持原有特性,例如将``撇号内的命令执行或变量扩展:

```sh
[root@101c7 4]# varlan="lang is $LANG" ; echo $varlan
lang is en_US.UTF-8
```

- **单引号**内特殊字符仅作为一般字符,例如:

```sh
[root@101c7 4]# varlang='lang is $LANG';echo $varlang
lang is $LANG
```

- **转义字符\\**将特殊符号(如$,\,空格,!等)变成一般字符;

- 在一串命令中,还需要通过其他命令提供的信息,可以使用\`命令\`或$(命令)来引用其他命令的stdout:

 ```sh
 [root@101c7 4]# varlan=$(uname -r);echo $varlan
 3.10.0-1160.41.1.el7.x86_64
 ```

- **变量增加内容**可用"变量名称=\$变量名称+新增内容"或"变量=${变量}+新增内容"来给变量添加内容:

```sh
[root@101c7 4]# PATH=${PATH}:/root
[root@101c7 4]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root
```

- 若该变量需要在其他子程序执行,用export来使当前变量变成环境变量:

```sh
[root@101c7 4]# export PATH
```

- 在子程序中修改全局环境变量并不会影响父shell中该变量的值,使用export也无效.
- 通常大写字符为系统默认环境变量,自行设置变量可以使用小写字符;



## 查看变量

要查看已定义的环境变量(全局变量),可以使用env或export命令:

```sh
[root@101c7 4]# env
XDG_SESSION_ID=15
HOSTNAME=101c7
SELINUX_ROLE_REQUESTED=
TERM=vt100
SHELL=/bin/bash
```

要查看所有变量,包括自定义变量(局部变量),使用set命令:

```sh
[root@101c7 ~]# set
XDG_SESSION_ID=15
_=16710
colors=/root/.dircolors
varlan=3.10.0-1160.41.1.el7.x86_64
varlang='lang is $LANG'
```



## 获取变量长度

指取得变量字符串的长度,可以这样表示${#变量名}:

```sh
[root@server1 bin]# echo ${#PATH}
59
```



## $PATH变量

环境变量PATH定义了可执行文件目录.

如果不同目录中有相同名称文件,则按照先查询到的同名命令先被执行.

不同身份用户默认的PATH不同.

查看当前系统中PATH变量,显示结果每个目录中间用冒号:隔开,按照先后顺序排列:

```sh
[root@101c7 a1]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

将/root目录加入到$PATH变量:

```sh
[root@101c7 a1]# PATH="$PATH":/root ; echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root
```



## 数组变量

数组是能够储存多个值的变量,这些值可以单独引用,也可以作为整个数组来引用.

要设置环境变量,可以把值放到括号里,值与值之间用空格分隔:

```sh
[root@server1 ~]# myarray=(one two)
```

使用echo直接变量只能读取第一个值,要引用一个单独的数组元素,必须使用索引值,从0开始索引:

```sh
[root@server1 ~]# echo $myarray 
one
[root@server1 ~]# echo ${myarray[1]}
two
```

要显示整个数组变量,可以用星号作为通配符放在索引位置:

```sh
[root@server1 ~]# echo ${myarray[*]}
one two
```

也可以直接修改某个索引位置的值:

```sh
[root@server1 ~]# myarray[1]=1
[root@server1 ~]# echo ${myarray[1]}
1
```

删除某个索引值位置的值用unset命令,它会清空对应索引值数据,但并不会重建索引:

```sh
[root@server1 ~]# unset myarray[0]
[root@server1 ~]# echo ${myarray[0]}

[root@server1 ~]# echo ${myarray[*]}
1
```

对其他shell而言,数组变量可移植性不好,所以不常用到.



## 特殊变量

**$$**代表shell的线程代号(PID),可以查询:

```sh
[root@101c7 ~]# echo $$
17701
```

**$?**代表上个执行的命令所回传的值,成功执行返回0,执行错误返回其他代码:

```sh
[root@101c7 ~]# 141=55
-bash: 141=55: command not found
[root@101c7 ~]# echo $?
127
```



## 读取变量

要由用户输入来定义变量可以使用read命令.

例如让用户在30秒内输入来自定义变量myname:

```sh
[root@101c7 ~]# read -p "type your name: " -t 30 myname
type your name: ass
[root@101c7 ~]# echo $myname
ass
```



## 声明变量

declare或typeset可以用来声明变量,并支持定义变量类型.

可用参数为:

| 参数 | 说明                                             |
| ---- | ------------------------------------------------ |
| -a   | 将变量定义为数组(array)类型                      |
| -i   | 将变量定义为整数(integer)类型                    |
| -x   | 用法与export一样,将变量变成环境变量              |
| -r   | 将变量设置readonly类型,不能修改也不能用unset取消 |

例如让变量计算1+2的结果:

```sh
[root@101c7 ~]# declare -i sum=1+2
[root@101c7 ~]# echo $sum
3
```

将变量由环境变量变为非环境变量:

```sh
[root@101c7 ~]# declare -r sum
[root@101c7 ~]# declare -p sum
declare -air sum='([0]="3")'
```

定义数组变量sum:

```sh
[root@101c7 ~]# declare -a sum
[root@101c7 ~]# sum[1]=111
[root@101c7 ~]# sum[2]=222
[root@101c7 ~]# echo ${sum[1]}
111
```

也可以不用declare提前声明.读取时必须${数组}的方式读取.



## 变量内容修改

变量内容修改使用echo设置,设置方式如下表:

| **变量设置方式**           | **说明**                                                    |
| -------------------------- | ----------------------------------------------------------- |
| ${变量#关键字}             | 若变量内容从头开始的数据符合"关键字",则将符合的最少数据删除 |
| ${变量##关键字}            | 若变量内容从头开始的数据符合"关键字",则将符合的最多数据删除 |
| ${变量%关键字}             | 若变量内容从尾向前的数据符合"关键字",则将符合的最少数据删除 |
| ${变量%%关键字}            | 若变量内容从尾向前的数据符合"关键字",则将符合的最多数据删除 |
| ${变量/旧字符串/新字符串}  | 若变量内容符合"旧字符串",则第一个旧字符串会被新字符串替代   |
| ${变量//旧字符串/新字符串} | 若变量内容符合"旧字符串",则全部的旧字符串会被新字符串替代   |

例如删除变量中最后一个"/root/bin":

```sh
[root@101c7 ~]# echo $myvar
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
[root@101c7 ~]# echo ${myvar%:*bin}
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
```

表示删除的是右数从:开始到bin结尾这一段内容

竟可能多地删除变量从左数,/usr开始到local结束之间的内容:

```sh
[root@101c7 ~]# echo ${myvar##/usr*local}
/bin:/usr/sbin:/usr/bin:/root/bin
```

竟可能少地删除结果是这样:

```sh
[root@101c7 ~]# echo ${myvar#/usr*local}
/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

 

## 设置变量默认值

当需要变量不存在时使用一个常用设置时,可以用 变量{变量-默认值} 来设置:

```sh
[root@101c7 ~]# echo $myvar1

[root@101c7 ~]# myvar1=${myvar1-root} ; echo $myvar1
root
[root@101c7 ~]# myvar1=alice ; myvar1=${myvar1-root} ; echo $myvar1
alice
```

如果变量为空字符串分辨不出是否没设置,这时用 变量{变量:-默认值} 来设置:

```sh
[root@101c7 ~]# myvar1="" ; myvar1=${myvar1:-root} ; echo $myvar1
root
```

设置方法对比结果如下表:

| 设置方式         | str没有设置        | str为空字符串      | str为非空字符串   |
| ---------------- | ------------------ | ------------------ | ----------------- |
| var=\${str-expr}       |var=expr|var=|var=\$str|
| var=\${str:-expr}        |var=expr|var=expr|var=$str|
| var=${str+expr}  | var=               | var=expr           | var=expr          |
| var=${str:+expr} | var=               | var=               | var=expr          |
| var=\${str=expr} |str=expr  var=expr|str不变  var=|str不变  var=$str|
| var=\${str:=expr} |str=expr  var=expr|str=expr  var=expr|str不变  var=$str|
| var=${str?expr}  | expr输出至stderr   | var=               | var=str           |
| var=${str:?expr} | expr输出至stderr   | expr输出至stderr   | var=str           |

 

