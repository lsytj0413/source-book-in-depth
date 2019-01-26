# 第二十五章: 启动一个项目 #

从本章开始, 以编写一个报告生成器程序为示例, 该程序会显示系统的各种统计数据和它的状态, 并将产生HTML格式的报告.

## 25.1 第一阶段: 最小的文档 ##

首先编写一个直接输出HTML文件的程序, 编辑 ~/bin/sys\_info\_page 文件内容如下:

```
#!/bin/bash
# Program to output a system information page
echo "<HTML>
    <HEAD>
        <TITLE>Page Title</TITLE>
    </HEAD>
    <BODY>
        Page Body.
    </BODY>
</HTML> "
```

然后使用以下命令生成结果文件:

```
sys_info_page > sys_info_page.html
```

## 25.2 第二阶段: 加入一点数据 ##

修改脚本, 增加一个网页标题和报告正文部分:

```
#!/bin/bash
# Program to output a system information page
echo "<HTML>
    <HEAD>
        <TITLE>System Information Report</TITLE>
    </HEAD>
    <BODY>
        <H1>System Information Report</H1>
    </BODY>
</HTML> "
```

## 25.3 变量和常量 ##

### 25.3.1 创建变量和常量 ###

我们可以创建一个名为title的变量, 并把 *System Information Report* 字符串赋值给它, 以简化脚本的编写和维护工作:

```
#!/bin/bash
# Program to output a system information page
title="System Information Report"
echo "<HTML>
    <HEAD>
        <TITLE>$title</TITLE>
    </HEAD>
    <BODY>
        <H1>$title</H1>
    </BODY>
</HTML> "
```

当shell碰到一个变量的时候, 它会自动创建该变量. 变量的命名规则如下:

1. 变量名可以由字母数字字符和下划线组成
2. 变量名的第一个字符必须是一个字母或一个下划线
3. 变量名中不允许出现空格和标点符号

在shell中不能辨别变量和常量, 一个惯例是指定大写字母来表示常量, 小写字母表示变量:

```
#!/bin/bash
# Program to output a system information page
TITLE="System Information Report For $HOSTNAME"
echo "<HTML>
    <HEAD>
        <TITLE>$TITLE</TITLE>
    </HEAD>
    <BODY>
        <H1>$TITLE</H1>
    </BODY>
</HTML> "
```

我们在标题中添加了HOSTNAME, 使标题与机器的网络名称相关联起来.

shell提供了一种方法, 通过使用带有 -r 选项的内部命令 declare来强制常量的不变性.

```
declare -r TITLE="System Information Report For $HOSTNAME"
```

但是这个功能极少被使用, 但为了很早之前的脚本, 它仍然存在.

### 25.3.2 为变量和常量赋值 ###

可以使用如下的方式给变量赋值:

```
variable=value
```

这里的 *variable* 是变量的名字, *value* 是一个字符串. shell不会在乎变量值的类型, 它把它们都看作是字符串. 通过使用带有 -i 选项的 declare 命令, 可以强制shell把赋值限定为整数, 但是也极少这样做.

在赋值过程中, 变量名和等号以及变量值之间必须没有空格, 这些值是可以展开成字符串的任意值, 以下是一些例子:

```
# 将变量赋值为字符串 "z"
a=z
# 空格需要使用引号包含
b="a string"
# 变量扩展
c="a string and $b"
# 命令扩展
d=$(ls -l foo.txt)
# 算术扩展
e=$((5 * 7))
# 转义
f="\t\ta string\n"
# 同一行多个赋值
a=5 b="a string"
```

在参数展开的过程中, 变量名可能被花括号包含.

```
# 将文件名从 myfile 修改为 myfile1
filename="myfile"
touch $filename
mv $filename $filename1
```

以上的命令会出错, 因为shell会把mv命令的第二个参数解释为一个新的变量, 可以使用以下方式来避免这个问题:

```
# 通过花括号避免shell把末尾的1解释为变量名的一部分
mv $filename ${filename}1
```

修改脚本, 将创建日期和时间, 以及创建者的用户名添加到输出的数据中:

```
#!/bin/bash
# Program to output a system information page
TITLE="System Information Report For $HOSTNAME"
CURRENT_TIME=$(date +"%x %r %Z")
TIME_STAMP="Generated $CURRENT_TIME, by $USER"
echo "<HTML>
    <HEAD>
        <TITLE>$TITLE</TITLE>
    </HEAD>
    <BODY>
        <H1>$TITLE</H1>
        <P>$TIME_STAMP</P>
    </BODY>
</HTML> "
```

## 25.4 here文档 ##

here文档是n另外一种I/O重定向方式, 我们在脚本文件中嵌入正文文本, 然后把它发送给一个命令的标准输入:

```
command << token
text
token
```

这里的command是一个可以接受标准输入的命令名, token是一个用来指示嵌入文本结束的字符串, 将脚本修改如下:

```
#!/bin/bash
# Program to output a system information page
TITLE="System Information Report For $HOSTNAME"
CURRENT_TIME=$(date +"%x %r %Z")
TIME_STAMP="Generated $CURRENT_TIME, by $USER"
cat << _EOF_
<HTML>
    <HEAD>
        <TITLE>$TITLE</TITLE>
    </HEAD>
    <BODY>
        <H1>$TITLE</H1>
        <P>$TIME_STAMP</P>
    </BODY>
</HTML>
_EOF_
```

我们在脚本中使用cat命令和一个here文档, 这个字符串 \_EOF\_ 被选做token, 标志着嵌入文本的结尾, 注意这个token必须在一行中单独出现, 并且文本行中没有末尾的空格.

在很大程度上, 和echo命令一样, 除了默认情况下 here文档中的单引号和双引号会失去它们在shell中的特殊含义, 这就允许我们在here文档中睡衣的嵌入引号.

```
#!/bin/bash
# Script to retrieve a file via FTP
FTP_SERVER=ftp.n1.debian.org
FTP_PATH=/debian/dists/lenny/main/installer-i386/current/images/cdrom
REMOTE_FILE=debian-cd_info.tar.gz
ftp -n << _EOF_
open $FTP_SERVER
user annoymous me@linuxbox
cd $FTP_PATH
hash
get $REMOTE_FILE
bye
_EOF_
ls -l $REMOTE_FILE
```

如果我们把重定向操作符从 [<<] 修改为 [<<-], shell会忽略在此 here文档中开头的tab字符, 从而提高脚本的可读性:

```
#!/bin/bash
# Script to retrieve a file via FTP
FTP_SERVER=ftp.n1.debian.org
FTP_PATH=/debian/dists/lenny/main/installer-i386/current/images/cdrom
REMOTE_FILE=debian-cd_info.tar.gz
ftp -n <<- _EOF_
    open $FTP_SERVER
    user annoymous me@linuxbox
    cd $FTP_PATH
    hash
    get $REMOTE_FILE
    bye
_EOF_
ls -l $REMOTE_FILE
```

## 25.5 本章结尾语 ##
