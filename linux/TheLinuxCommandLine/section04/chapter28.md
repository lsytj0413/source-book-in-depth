# 第二十八章: 读取键盘输入 #

## 28.1 read-从标准输入读取输入值 ##

内嵌命令read的作用是读取一行标准输入, 可用于读取键盘输入值或应用重定向读取文件中的一行, 语法结构如下:

```
read [-options] [variable...]
```

其中variable是一个或多个用于存放输入值的变量, 若没有提供任何变量则由shell变量REPLY来存储数据行.

options为一个或多个可用的选项, 常用的选项如下表:

| 选项 | 描述 |
|:--|:--|
| -a array | 将输入值从索引为0的位置开始赋值给array |
| -d delimiter | 用字符串delimiter的第一个字符标志输入的结束, 而不是新的一行的开始 |
| -e | 使用Readline处理输入, 此命令使用户能使用命令行模式的相同方式编辑输入 |
| -n num | 从输入中读取num个字符, 而不是一整行 |
| -p prompt | 使用prompt字符串提示用户进行输入 |
| -r | 原始模式, 不能将反斜杠字符翻译为转义码 |
| -s | 保密模式, 不在屏幕中显示输入的字符 |
| -t seconds | 超时, 若输入超时则返回一个非0的退出状态 |
| -u fd | 从文件说明符fd读取输入, 而不是从标准输入中读取 |

使用read命令改写整数验证脚本如下:

```
#!/bin/bash

# read-integer: evaluate the value of an integer.

echo -n "Please enter an integer -> "
read int

if [ -z "$int" ]; then
    echo "int is empty." >&2
    exit 1
fi

if [[ "$int" =~ ^-?[0-9]+$ ]]; then
    if [ $int -eq 0 ]; then
        echo "int is zero."
    else
        if [ $int -lt 0 ]; then
            echo "int is negative."
        else
            echo "int is positive."
        fi
        if [ $((int % 2)) -eq 0 ]; then
            echo "int is even."
        else
            echo "int is odd."
        fi
    fi
else
    echo "int is not an integer." >&2
    exit 1
fi
```

或者读取输入值到多个变量:

```
echo -n "Enter one or more values > "
read var1 var2 var3 var4 var5
```

若read命令读取的值少于预期的数目, 则多余的变量值为空, 而输入值多余预期的数目时, 最后的变量则包含了所有的多余值.

### 28.1.1 选项 ###

可以使用 -p 选项来显示提示符:

```
read -p "Enter one or more values > " var1 var2
```

也可以使用 -t 选项加上 -s 选项来提示用户输入密码, 并限制等待时间:

```
read -t 10 -sp "ENter secret passphrase > " secret_pass
```

### 28.1.2 使用IFS间隔输入字段 ###

通常shell会间隔提供给read命令的内容, 在输入行由一个或多个空格将多个单词分隔成为分离的单项, 再由read命令将这些单项赋值给不同的变量.
此行为是由shell变量IFS(Internal Field Separator) 设定的, IFS的默认值包含了空格, 制表符和换行符, 每一种都可以将字符彼此分隔开.

可以通过改变IFS变量来控制read命令输入的间隔方式, 例如将IFS变量修改为单个冒号即可使用read命令读取 /etc/passwd 文件中的内容, 并将各个字段分隔为不同的变量. 脚本如下:

```
#!/bin/bash

# read-ifs: read fields from a file

FILE=/etc/passwd

read -p "Enter a username > " user_name

file_info=$(grep "^$user_name:" $FILE)

if [ -n "$file_info" ]; then
    IFS=":" read user pw uid gid name home shell <<< "$file_info"
    echo "User =                '$user'"
    echo "UID =                  '$uid'"
    echo "GID =                  '$gid'"
    echo "Full Name =           '$name'"
    echo "Home Dir =            '$home'"
    echo "Shell =              '$shell'"
else
    echo "No such user '$user_name'" >&2
    exit 1
fi
```

在read命令那一行, shell允许在命令执行前对一到多个变量进行赋值, 并且这些赋值操作只在该行后续命令执行期间内有效.
<<< 操作符是一条嵌入字符串, 它包含的是一条字符串, 在本例中这个字符串被输送给read命令. 也许有人认为以下的代码也是等价的:

```
echo "$file_info" | IFS=":" read user pw uid gid name home shell
```

但是上面的命令不会生效, 因为read命令不可重定向. 更深层的原因是, 在bash或其他shell中, 管道会创造子shell, 子shell复制了shell及pipline执行命令过程中使用到的shell环境. 所以上例中的read命令是在子shell中执行的.

## 28.2 验证输入 ##

一个好的程序应该能够处理错误输入, 以下是一个例子:

```
#!/bin/bash

# read-validate: validate input

invalid_input () {
    echo "Invalid input '$REPLY'" >&2
    exit 1
}

read -p "Enter a single item > "

# input is empty (invalid)
[[ -z $REPLY ]] && invalid_input

# inpus is multiple items (invalid)
(( $(echo $REPLY | wc -w) > 1 )) && invalid_input

# is input a valid filename?
if [[ $REPLY =~ ^[-[:alnum:]\._]+$ ]]; then
    echo "'$REPLY' is a valid filename"
    if [[ -e $REPLY ]]; then
        echo "And file '$REPLY' exists."
    else
        echo "However, file '$REPLY' does not exist."
    fi

    # is input a floating point number?
    if [[ $REPLY =~ ^-?[[:digit:]]*\.[[:digit:]]+$ ]]; then
        echo "'$REPLY' is a floating point number."
    else
        echo "'$REPLY' is not a floating point number."
    fi

    # is input an integer?
    if [[ $REPLY =~ ^-?[[:digit:]]+$ ]]; then
        echo "'$REPLY' is an integer."
    else
        echo "'$REPLY' is not an integer."
    fi
else
    echo "The string '$REPLY' is not a valid filename."
fi
```

## 28.3 菜单 ##

菜单驱动是一种常见的交互方式, 在菜单驱动的程序会呈现给用户一系列的选项, 并请求用户进行选择.

```
#!/bin/bash

# read-menu: a menu driven system information program

clear
echo "
Please Select:

1. Display System Information
2. Display Disk Space
3. Display Home Space Utilization
0. Quit
"

read -p "Enter selection [0-3] > "

if [[ $REPLY =~ ^[0-3]$ ]]; then
    if [[ $REPLY == 0 ]]; then
        echo "Program terminated."
        exit
    fi
    if [[ $REPLY == 1 ]]; then
        echo "Hostname: $HOSTNAME"
        uptime
        exit
    fi
    if [[ $REPLY == 2 ]]; then
        df -h
        exit
    fi
    if [[ $REPLY == 3 ]]; then
        if [[ $(id -u) -eq 0 ]]; then
            echo "Home Space Utilization (All Users)"
            du -sh /home/*
        else
            echo "Home Space Utilization ($USER)"
            du -sh $HOME
        fi
        exit
    fi
else
    echo "Invalid entry." >&2
    exit 1
fi
```

## 28.4 本章结尾语 ##

学习到本章, 用户已经能够写出很多不同功效的程序, 例如专门的计算程序和易用的命令行工具前端.

## 28.5 附加项 ##

作为练习, 用户可以使用 test命令替代 [[]] 复合命令来改写本章的程序, 并且使用grep命令处理正则表达式, 然后评估其退出状态.

Bash 参考手册有一章关于内部命令的内容, 其包括了read命令:

[http://www.gnu.org/software/bash/manual/bashref.html#Bash-Builtins](http://www.gnu.org/software/bash/manual/bashref.html#Bash-Builtins)
