# Sed工具

## 字符流编辑器

sed是一个"非交互式的"面向字符流的编辑器(Strem Editor),输入流通过程序并输出直接送到标准输出端,**输入文件本身不会发生改变**,是操作,过滤和转换文本内容的强大工具.

sed可以把输出重定位到文件,但不允许直接送到输入文件

命令语法:

`sed [-nefr] [动作]`

主要参数:

| 参数 | 说明                                             |
| ---- | ------------------------------------------------ |
| -n   | 使用安静模式.只有被处理后的数据才会被列出来.     |
| -e   | 直接在命令行模式上进行sed的动作编辑.             |
| -f   | 指定加载的sed脚本.                               |
| -r   | 设置成支持扩展型正则表达式,默认是基础正则表达式. |
| -i   | 直接修改读取的文件被容,不在屏幕输出.             |

动作说明:

`[n1 [,n2]] function`

function参数:

| 参数 | 说明                                          |
| ---- | --------------------------------------------- |
| a    | 新增,  在当前行后添加一行或多行;              |
| c    | 替换行,  c后面接字符串,替换n1,n2之间的行;     |
| d    | 删除行,只会删除输出中的内容,必须提供地址访问; |
| i    | 插入, 在当前行之前插入文本;                   |
| s    | 替换数据,可以直接进行替换的工作;              |
| p    | 打印,和-n一起合用来屏蔽sed的默认输出;         |
| q    | 第一个模式匹配完成后立即退出;                 |
| r    | 从文件中读取输入行,类似重定向<;               |
| =    | 显示文件行号;                                 |
| !    | 对所选以外的所有行应用命令                    |

替换标志:

| **数值** | 表示替换行内第几次匹配的值.                                |
| -------- | ---------------------------------------------------------- |
| i        | 忽略大小写.                                                |
| g        | 在行内进行全局替换,不设置只会替换每行第一次出现的原始字段. |
| p        | 只打印替换过的行.                                          |
| w        | 将行写入文件.                                              |



## 寻址模式

- **addr1,add2**

  定址 addr1,add2 决定用于对哪些行进行编辑.地址的形式可以是数字,正则表达式或二者结合.

  如果没有指定地址, sed 将处理输入文件中的所有行.

  如果定址是一个数字,则这个数字代表行号.

  如果是逗号分隔的两个行号,那么需要处理的定址就是两行之间的范围(包括两行在内).

- **addr1,+N**

  从addr1这行到往下N行匹配,总共匹配N+1行

- **first~step**

  first 指起始匹配行,step指步长,例如sed -n 2~5p表示从第2行开始匹配,隔5行匹配一次,即 2,7,12...

- **$**

  表示匹配最后一行

- **/REGEXP/**

  表示匹配正则那一行,通过//之间的正则来匹配.

- **\cREGEXPc**

  表示匹配正则那一行,通过\c和c之间的正则来匹配,c可以是任一字符



## 在命令行使用

例如打印并替换list文件中的BOB为Candy:

```sh
[root@localhost ~]# sed 's/BOB/Candy/' list
Alice eats apples.
Bob pets cats.
```

同时多个替换命令可以在''内使用;分割两个指令:

```sh
[root@localhost ~]# sed 's/BOB/Candy/;s/aaa/bbb/' list
```

也可以用-e参数来运行多条指令:

```sh
[root@localhost ~]# sed -e 's/BOB/Candy/' -e 's/aaa/bbb/' list
```

把单引号''内的内容用大括号{}括起来,可以多行输入执行多条命令:

```sh
[root@ ~]# sed -n '{
s/i/@/
s/@/III/p
}' test.txt
102,Jason SmIIIth,IT Manager
103,Raj Reddy,SysadmIIIn
105,Jane MIIIller,Sales Manager
```



## 在脚本中使用

在sed和awk中,每个指令都包括模式和过程.

模式是由斜杠/分割的正则表达式.

过程指定一个或多个将被执行的动作,过程必须用大括号括起.

**Sed 脚本执行流程:**

1. sed每次从input-file中读取一行记录,并执行命令,也就是会在第一行运用脚本所有内容.
2. 处理下一行,直到文件结束.
3. 脚本执行遵从下面简单易记的顺序:Read,Execute,Print,Repeat(读取,执行,打印,重复),简称 REPR

调用时直接在-f参数接脚本名.例如用SED.sed脚本处理list文件:

```sh
[root@localhost ~]# sed -f SED.sed list
```



## 打印显示

使用p动作将文件内容打印出来.例如从第3行打印到最后一行:

```sh
[root@101c7 sdb4m]# sed -n '3,$p' c.cfg 
1.4
1a4
```

打印正则匹配halt开头的行到operator开头的行:

```sh
[root@101c7 sdb4m]# sed -n '/^halt/,/^operator/p' b.cfg 
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
```



## 写入到文件

使用w动作将指定内容写入到文件中,而不是在屏幕显示:

```sh
[root@101c7 sdb4m]# sed -n '1,3 w cnew.txt' c.cfg;cat cnew.txt
abcac
abc\n
1.4
```

如果使用-i参数则直接将修改应用到文件.例如去除文件中的#注释行和空行:

```sh
[root@101c7 sdb4m]# sed -i -r '/^#|^$/d' ac.cfg
```



## 删除指定行

使用d动作定义要删除的范围.例如将文件内容2到5行删除并打印:

```sh
[root@101c7 sdb4m]# nl b.cfg | sed '2,5d'
     1  root:x:0:0:root:/root:/bin/bash
     6  sync:x:5:0:sync:/sbin:/bin/sync
     7  shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
```

删除第一次匹配到的sync行,并删除后面2行:

```sh
[root@101c7 sdb4m]# nl b.cfg | sed '/sync/,+2d'
     4  adm:x:3:4:adm:/var/adm:/sbin/nologin
     5  lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
     9  mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
```



## 插入数据

使用a或i动作将字符串插入到指定行之后或之前:

```sh
[root@101c7 sdb4m]# sed -r '/^root/a 1999' b.cfg 
root:x:0:0:root:/root:/bin/bash
1999
bin:x:1:1:bin:/bin:/sbin/nologin
```

增加多行内容可以用\来断行:

```sh
[root@101c7 sdb4m]# sed -r '/^root/a 1999 \
2000 
' b.cfg
root:x:0:0:root:/root:/bin/bash
1999 
2000
```



## 替换数据

整行替换使用c动作将行内容替换成指定内容:

```sh
[root@101c7 sdb4m]# sed -r '/^root/c 1999 \' b.cfg 
1999 
bin:x:1:1:bin:/bin:/sbin/nologin
```

以行为单位替换行内数据使用s动作.格式为:**'s/要被替换的字符串/新字符串/替换标记'**

例如将配置文件内root行的的notempty替换成emptyok:

```sh
[root@101c7 sdb4m]# sed '/root/s/notempty/emptyok/g' a.cfg | grep 'empty'
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
```

除了使用全局替换的g标志外,还可以使用数字来表示替换行内第几次匹配结果.

例如替换每行第二次出现的a为A:

```sh
[root@101c7 sdb4m]# sed 's/a/A/2' c.cfg 
abcAca
abc\nA
```

还有p标记,用来只打印修改后的行:

```sh
[root@101c7 sdb4m]# sed -n 's/a/A/3p' c.cfg 
abcacA
```

可以自由设置**分界符号**,这样在可以方便使用作为字符串的/符号.

例如使用@作为分界符,替换文件中的/root/etc/为/etc/:

```sh
[root@101c7 sdb4m]# sed 's@/root/etc/@/etc/@' c.cfg 
abcaca
/etc/
```

**替换位置为&**时,代表他会被替换成匹配到的被替换字符串或正则表达式.

例如将行号数字用大括号{}括起来:

```sh
[root@101c7 sdb4m]# nl c.cfg | sed 's/[0-9]/{&}/1'
     {1}        abcaca
     {2}        /root/etc/
     {3}        1.4
     {4}        1a4
```

sed支持**分组替换**,每个分组数据用( )括起来.

例如抽取第1和3组分组数据,并在第3分组数据用[]括起来:

```sh
[root@101c7 sdb4m]# sed -r 's/(^[a-z]+)(\:.)(\:[0-9]+\:).*/\1[\3]/g' b.cfg
root[:0:]
bin[:1:]
daemon[:2:]
```



 

 

 

 