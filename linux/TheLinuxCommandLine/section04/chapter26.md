# 第二十六章: 自顶向下设计 #

对于任意一个大项目, 一般都会把繁重而复杂的任务分割为细小且简单的任务.

先确定上层步骤, 然后再逐步细化这些步骤的过程被称为自顶向下设计, 这种技巧允许我们把庞大而复杂的任务分割为许多小而简单的任务.
自顶向下设计是一种常见的程序设计方法, 尤其适合shell编程.

## 26.1 shell函数 ##

目前我们的脚本执行以下步骤来产生这个HTML文档:

1. 打开网页
2. 打开网页标头
3. 设置网页标题
4. 关闭网页标头
5. 打开网页主体部分
6. 输出网页标头
7. 输出时间戳
8. 关闭网页主体
9. 关闭网页

我们将在步骤7和步骤8之间添加一些额外的任务:

- 系统正常运行时间和负载: 这是自上次关机或重启系统的运行时间, 以及在几个时间间隔内当前运行在处理中的平均任务量
- 磁盘空间: 系统中存储设备的总使用量
- 家目录空间: 每个用户所使用的存储空间使用量

对于每一个任务, 都有相应的命令, 通过命令替换就可以很容易的把它们添加到我们的脚本中:

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
        $(report_uptime)
        $(report_disk_space)
        $(report_home_space)
    </BODY>
</HTML>
_EOF_
```

我们可以使用两种方法来创建这些额外的命令: 第一种我们可以分别编写三个脚本, 并把它们放置到环境变量PATH所列出的目录下; 第二种我们也可以把这些脚本作为shell函数嵌入到我们的程序中. shell函数有两种语法形式:

```
function name {
    commands
    return
}

and 

name () {
    commands
    return
}

# name是函数名, commands是一系列包含在函数中的命令
```

一个函数实例如下:

```

#!/bin/bash
# Shell function demo

function funct {
    echo "Step 2"
    return
}

# Main program starts here
echo "Step 1"
funct
echo "Step 3"
```

为了使函数调用被识别出是shell函数而不是外部程序的名字, 所以在脚本中shell函数定义必须出现在函数调用之前. 现在修改脚本如下:

```
#!/bin/bash
# Program to output a system information page
TITLE="System Information Report For $HOSTNAME"
CURRENT_TIME=$(date +"%x %r %Z")
TIME_STAMP="Generated $CURRENT_TIME, by $USER"

report_uptime () {
    return
}

report_disk_space () {
    return
}

report_home_space () {
    return
}

cat << _EOF_
<HTML>
    <HEAD>
        <TITLE>$TITLE</TITLE>
    </HEAD>
    <BODY>
        <H1>$TITLE</H1>
        <P>$TIME_STAMP</P>
        $(report_uptime)
        $(report_disk_space)
        $(report_home_space)
    </BODY>
</HTML>
_EOF_
```

shell函数的命名规则和变量一样, 一个函数必须至少包含一条命令, 这条return命令(可选)满足要求.

## 26.2 局部变量 ##

目前我们所写的脚本中, 所有的变量都是全局变量, 全局变量在整个程序中保持存在. 局部变量只能在定义它们的shell函数中使用, 并且一旦shell函数执行完毕, 它们就不存在了.

局部变量名可以与已存在的变量名相同, 这些变量可以是全局变量, 或者是其他shell函数中的局部变量, 而不必担心潜在的名字冲突.

以下是一个使用局部变量的实例:

```
#!/bin/bash
# local-vars: script to demonstrate local variables
foo=0 # 全局变量
funct_1 () {
    local foo # 局部变量
    foo=1
    echo "funct_1: foo = $foo"
}

funct_2 () {
    local foo # 局部变量
    foo=2
    echo "funct_2: foo = $foo"
}

echo "global:  foo = $foo"
funct_1
echo "global: foo = $foo"
funct_2
echo "global: foo = $foo"
```

通过在变量名之前加上单词local来定义局部变量, 在这个shell函数之外这个变量不再存在.

## 26.3 保持脚本的运行 ##

通过添加空函数, 程序员称之为 *stub* , 我们可以在早期阶段证明程序的逻辑流程. 当构建一个stub的时候, 最好包含一些为程序员提供反馈信息的代码, 这些信息可以展示正在执行的逻辑流程.

修改脚本如下:

```
report_uptime () {
    echo "Function report_uptime executed."
    return
}

report_disk_space () {
    echo "Function report_disk_space executed."
    return
}

report_home_space () {
    echo "Function report_home_space executed."
    return
}
```

现在开始实现一些函数代码, 首先是 report_uptime 函数:

```
# 使用here文档来输出标题和uptime命令的输出结果, 使用 PRE 标签来保持命令的输出格式
report_uptime () {
    cat <<- _EOF_
    <H2>System Uptime</H2>
    <PRE>$(uptime)</PRE>
    _EOF_
    return
}
```

然后是 report\_home\_space函数:

```
report_home_space () {
    cat <<- _EOF_
    <H2>Home Space Utilization</H2>
    <PRE>$(du -sh /home/*)</PRE>
    _EOF_
    return
}
```

## 26.4 本章结尾语 ##
