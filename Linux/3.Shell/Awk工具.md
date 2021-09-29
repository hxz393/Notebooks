# Awk工具

## 命令语法

awk主要是处理每一行的字段内的数据,默认字段的分隔符为空格或[Tab].

awk的命令语法如下:

`awk '条件类型1{动作1} 条件类型2{动作2} ...' 文件名`

awk后面接两个单引号''并加上大括号{}来设置想要对数据进行的处理操作.

\$0代表整个输入行,$n表示第n个字段.在应用脚本之前,awk会先拆分输入记录.

每次从一个或多个文件中读入一行或从标准输入中读入一行,指令包含在小括号对中.

awk将每个输入行解释为一条记录,行中的每个单词由空格或制表符分割解释为一个字段.



## awk处理流程

整个awk的处理流程是:

1. 读入第一行,并将第一行的数据填入\$0,\$1,\$2等变量中;
2. 依据条件类型的限制,判断是否需要进行后面的动作;
3. 完成所有动作与条件类型;
4. 重复以上流程步骤,直到所有数据读完为止.



## awk内置变量

| **变量名**  | **代表意义**                   |
| ----------- | ------------------------------ |
| NF          | 每一行($0)拥有的字段总数(列数) |
| NR          | 目前awk所正在处理数据的行号    |
| FS          | 输入字段分隔字符               |
| OFS         | 输出字段分隔符                 |
| RS          | 输入记录分隔符                 |
| ORS         | 输出记录分隔符                 |
| FIELDWIDTHS | 定义分割数据字段依据的宽度     |



## 格式化打印

printf可以帮我们将数据输出的结果格式化,并且支持一些特殊字符.

命令格式:

`printf '打印样式' 实际内容`

特殊样式:

| 参数  | 样式                           |
| ----- | ------------------------------ |
| \a    | 警告声音;                      |
| \b    | 退格键;                        |
| \f    | 清除屏幕;                      |
| \n    | 输出到新一行;                  |
| \r    | 回车键;                        |
| \t    | 水平[tab]按键;                 |
| \v    | 垂直[tab]按键;                 |
| \xNN  | NN为两位数的数字,可以转为字符; |
| %ns   | n为数字,s表示字符串数量;       |
| %ni   | n为数字,i表示整数位数;         |
| %N.nf | n与N为数字,f代表多少位小数;    |



## 分割数据

例如取出last命令结果中用户名(第一列)与登录IP(第三列)地址:

```sh
[root@101c7 sdb4m]# last -n 5 | awk '{print $1 "--" $3}'
root--Sat
root--192.168.2.101
root--192.168.2.101
root--192.168.2.101
root--192.168.2.101
```

使用print功能键结果打印出来是awk最常用功能.

将内置变量打印出来:

```sh
[root@101c7 sdb4m]# last -n 5 | awk '{print $1 "  lines:" NR "   columes:" NF}'
root  lines:1   columes:9
root  lines:2   columes:10
root  lines:3   columes:10
root  lines:4   columes:10
root  lines:5   columes:10
```

awk可以使用逻辑运算符来写判断条件.例如查找UID小于10的用户:

```sh
[root@101c7 sdb4m]# cat b.cfg | awk 'BEGIN {FS=":"} $3<10{print $1 "\t" $3}'
root    0
bin     1
daemon  2
adm     3
```

命令由两个组成,前面一个命令定义了分隔符FS的值为:(也可以使用-F参数),同时使用BEGIN这个关键字,预先设置awk的变量.第二部分做逻辑判断$3小于10的才运行后面print命令.

同时使用FS和OFS格式化输出:

```sh
[root@server1 bin]# awk 'BEGIN{FS=","; OFS="--"} {print $1,$2}' user.csv 
user6--user6
user7--user7
```

自定义按字符数来分割:

```sh
[root@server1 bin]# awk 'BEGIN{FIELDWIDTHS="5 1"} {print $1,$2,$0}' user.csv 
user6 , user6,user6
user7 , user7,user7
```

如果数据是多行形式,可以指定分隔符为\n,以空行为界定新记录的分隔符:

```sh
[root@server1 bin]# awk 'BEGIN{FS="\n"; RS=""} {print $0 $2}' user.csv 
user6,user6
403403
user7,user7
404404
```

awk还可以使用正则表达式来进行模式匹配.例如只显示$1含有root字符串的用户:

```sh
[root@101c7 sdb4m]# cat b.cfg | awk -F: '/root/{print $1 "\t" $3}'
root    0
operator        11
```



 

 

 