# Vim基础

## Vim模式

vi分为三种模式,一般模式,编辑模式与命令行模式:

- 一般指令模式(Command Mode)

  用vi打开一个文件就进入一般模式,可以使用方向键来移动光标,删除,复制与粘贴.

- 编辑模式(Insert Mode)

  在一般模式中,按下"i o a r"任何一个按键之后才会进入编辑模式,左下角会显示INSERT或REPLACE等字样.按Esc键可以返回一般模式.

- 命令行模式(Command-line Mode)

  在一般模式下,输入": / ?"三个中的任何一个按键,可以将光标移动到最下面一行,在此模式可以查找数据,读取保存,大量替换等操作.



## 界面介绍

使用vi打开一个已存在的文件后,显示如下所示:

```sh
[root@101c7 ~]# vi anaconda-ks.cfg 
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
cdrom
# Use graphical install
graphical
# Run the Setup Agent on first boot
"anaconda-ks.cfg" 48L, 1260C
```

最下面一行分别展示了:

- "anaconda-ks.cfg": 文件名;
- 48L: 文件有48行;
- 1260C: 文件有1260个字符,也就是1260B大小.

按下i键后,进入到编辑模式,最左下角显示:

```sh
-- INSERT --
```

此时可以自由输入文字.按下Esc键退回到一般模式,再输入":wq"按回车可以保存并离开vi:

```sh
:wq
"anaconda-ks.cfg" 48L, 1260C written
```

如果文件权限不对,可以使用":wq!"来强制写入



## 恢复功能

恢复功能是指,当因为某些原因导致死机的情况下,能通过一些方法将之前未保存的数据找回来.

在使用vim编辑文件时,vim会在被编辑文件同目录下新建一个名为.filename.swp的交换文件.每当键入200个字符或有4秒没有键入内容时,交换文件都会自动地更新.

例如用vi新建打开一个文件,在命令模式下按Ctrl+z中断,再查看.swp文件内容:

```sh
[root@101c7 4]# vi 1.txt
a
b
~

[1]+  Stopped                 vi 1.txt
[root@101c7 4]# cat .1.txt.swp 
U3210#"! Utpad???
```

可以看到swp实时保存了我们输入的内容.

再次打开会提示有swp临时文件存在:

```sh
[root@101c7 4]# vim 1.txt 

E325: ATTENTION
Found a swap file by the name ".1.txt.swp"
          owned by: root   dated: Sat Sep 11 03:44:17 2021
         file name: ~root/4/1.txt
          modified: YES
         user name: root   host name: 101c7
        process ID: 87510 (still running)
While opening file "1.txt"
             dated: Sat Sep 11 03:44:11 2021

(1) Another program may be editing the same file.  If this is the case,
    be careful not to end up with two different instances of the same
    file when making changes.  Quit, or continue with caution.
(2) An edit session for this file crashed.
    If this is the case, use ":recover" or "vim -r 1.txt"
    to recover the changes (see ":help recovery").
    If you did this already, delete the swap file ".1.txt.swp"
    to avoid this message.

Swap file ".1.txt.swp" already exists!
[O]pen Read-Only, (E)dit anyway, (R)ecover, (Q)uit, (A)bort:
```

提示说出现这一原因有两种情况:

- 此文件正被另一个程序所使用;
- 文件在编辑过程中非正常退出.

此时可以选择以哪种模式来打开:

| 按键 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| O    | 以只读模式打开,显示内容为没有保存的内容;                     |
| E    | 以正常模式编辑文件,不会载入暂存文件内容;                     |
| R    | 加载暂存文件内容,恢复之前的作业.修改完毕后需要手动删除.swp文件; |
| Q    | 直接退出不进行操作;                                          |
| A    | 等同于Q,直接退出.                                            |



## 环境设置

在使用vim编辑文件时,文件会继承上一次编辑时的状态,比如光标出现在最后一行.这个记录操作文件保存在~/.viminfo里面.

在命令模式下用set修改的设置值保存在/etc/vimrc这个文件中,不建议动它.需要自定义设置可以新建~/.vimrc这个文件将希望设置的值写入.

例如设置自动缩进和行号功能:

```sh
[root@101c7 ~]# vim ~/.vimrc
set nu          "hanghao
set autoindent  "suojin
```

再次打开vim即可生效:

```sh
[root@101c7 ~]# vim ~/.vimrc
  1 set nu          "hanghao
  2 set autoindent  "suojin
```



 

 

 

 

 

