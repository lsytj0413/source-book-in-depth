# 第二十七章: 流控制: IF分支语句 #

## 27.1 使用if ##

在shell中一个简单的分支语句如下:

```
x=5
if [ $x = 5 ]; then
    echo "x equals 5."
else
    echo "x does not euqal 5."
fi
```

或者直接在命令行中如下输入:

```
x=5
if [ $x = 5 ]; then echo "equals 5."; else echo "does not equal 5"; fi
```

if语句的语法格式如下:

```
if commands; then
    commands
[elif commands; then
    commands...]
[else
    commands]
fi
```

## 27.2 退出状态 ##

命令(包括shell脚本) 在执行完毕后, 会向操作系统返回一个值, 称之为退出状态. 这个值是一个0～255的整数, 用来表示命令执行成功还是失败.
按照惯例使用0来表示执行成功, 其他数值表示执行失败. shell提供了一个可以用来检测退出状态的参数, 如下:

```
ls -d /usr/bin
echo $?
ls -d /bin/usr
echo $?
```

有些命令使用不同的退出值来诊断错误, 而许多命令在执行失败时, 只是简单的退出并发送数字1. 数字0总是表示执行成功.

shell提供了两个简单的命令, 以一个0或1退出状态来终止执行. true命令表示执行成功, false命令表示执行失败.

```
true
echo $?
false
echo $?
```

if命令根据后面的命令的执行结果来选择是否执行后续的commands. 如果if后面有一系列的命令, 那么根据最后一个命令的执行结果来进行评估.

## 27.3 使用test命令 ##

test命令经常和if命令一起使用, test命令会执行各种检查和比较, 有以下两种等价方式:

```
test expression
[ expression ]
```

### 27.3.1 文件表达式 ###

下表中的表达式用来评估文件的状态:

| 表达式 | 为true的条件 |
|:--|:--|
| file1 -ef file2 | file1和file2拥有相同的信息节点编号, 即通过硬链接指向同一个文件 |
| file1 -nt file2 | file1比file2新 |
| file1 -ot file2 | file1比file2旧 |
| -b file | file存在并且是一个块设备文件 |
| -c file | file存在并且是一个字符设备文件 |
| -d file | file存在并且是一个目录 |
| -e file | file存在 |
| -f file | file存在并且是一个普通文件 |
| -g file | file存在并且设置了组ID |
| -G file | file存在并且属于有效组ID |
| -k file | file存在并且有 sticky bit属性 |
| -L file | file存在并且是一个符号链接 |
| -O file | file存在并且属于有效用户ID |
| -p file | file存在并且是一个命名管道 |
| -r file | file存在并且可读(有效用户拥有可读权限) |
| -s file | file存在并且长度大于0 |
| -S file | file存在并且是一个网络套接字 |
| -t fd | fd是一个定向到终端/从终端定向的文件描述符, 可以用来确定标准输入/输出/错误是否被重定向 |
| -u file | file存在并且设置了setuid位 |
| -w file | file存在并且可写(有效用户拥有可写权限) |
| -x file | file存在并且可执行(有效用户拥有执行/搜索权限) |

以下的脚本演示了一些表达式的使用:

```
#!/bin/bash

# test-file: Evaluate the status of a file

FILE=~/.bashrc

if [ -e "$FILE" ]; then

if [ -e "$FILE" ]; then
    if [ -f "$FILE" ]; then
        echo "$FILE is a regular file."
    fi
    if [ -d "$FILE" ]; then
        echo "$FILE is a directory."
    fi
    if [ -r "$FILE" ]; then
        echo "$FILE is readable."
    fi
    if [ -w "$FILE" ]; then
        echo "$FILE is writable."
    fi
    if [ -x "$FILE" ]; then
        echo "$FILE is executable/searchable."
    fi
else
    echo "$FILE does not exist"
    exit 1
fi

exit
```

该脚本中有两个需要注意的点:

1. 我们在引号中使用 $FILE 表达式, 尽管引号不是必须的, 但是这可以防止FILE为空的情况. 如果 $FILE 扩展产生一个空值, 将导致一个错误. 使用引号包含则可以确保操作符后面紧跟一个空字符串.

2. 脚本末尾使用exit退出脚本, 该命令接受一个可选参数, 作为脚本的退出状态值.

可以在return命令中包含一个整数, 使shell函数返回一个退出状态. 例如将以上脚本的功能修改为一个shell函数如下:

```
test_file () {

    FILE=~/.bashrc

    if [ -e "$FILE" ]; then
        if [ -f "$FILE" ]; then
            echo "$FILE is a regular file."
        fi
        if [ -d "$FILE" ]; then
            echo "$FILE is a directory."
        fi
        if [ -r "$FILE" ]; then
            echo "$FILE is readable."
        fi
        if [ -w "$FILE" ]; then
            echo "$FILE is writable."
        fi
        if [ -x "$FILE" ]; then
            echo "$FILE is executable/searchable."
        fi
    else
        echo "$FILE does not exist"
        return 1
    fi
}
```

### 27.3.2 字符串表达式 ###

下表中的表达式用来测试字符串的操作:

| 表达式 | 为true的条件 |
|:--|:--|
| string | 字符串不为空 |
| -n string | 字符串长度大于 0 |
| -z string | 字符串长度等于 0 |
| string1=string2 或 string1==string2 | 字符串是否相等, 双等号使用更多 | 
| string1!=string2 | 字符串不相等 |
| string1>string2 | 在排序时, string1在string2之后 |
| string1<string2 | 在排序时, string1在string2之前 |

在test命令中, "<" 和 ">" 符号需要使用引号包含(或使用反斜杠转义), 以避免shell将其解释为重定向操作符. 同时, 排序遵从当前语系的排序规则, 但在bash4.0版本及之前是使用的ASCII(POSIX) 排序方式.

下面是一个合并字符串表达式的脚本:

```
#!/bin/bash

# test-string: evaluate the value of a string

ANSWER=maybe

if [ -z "$ANSWER" ]; then
    echo "There is no answer." >&2
    exit 1
fi

if [ "$ANSWER" == "yes" ]; then
    echo "The answer is yes."
elif [ "$ANSWER" = "no" ]; then
    echo "The answer is no."
elif [ "$ANSWER" == "maybe" ]; then
    echo "The answer is maybe."
else
    echo "The answer is unknown."
fi
```

### 27.3.3 整数表达式 ###

下表中的表达式用来进行整数判断的操作:

| 表达式 | 为true的条件 |
|:--|:--|
| integer1 -eq integer2 | 相等 |
| integer1 -ne integer2 | 不相等 |
| integer1 -le integer2 | integer1小于等于 integer2 |
| integer1 -lt integer2 | integer1小于integer2 |
| integer1 -ge integer2 | integer1大于等于integer2 |
| integer1 -gt integer2 | integer1大于 integer2 |

下面是一个演示脚本:

```
#!/bin/bash

# test-integer: evaluate the value of an integer.

INT=-5

if [ -z "$INT" ]; then
    echo "INT is empty." >&2
    exit 1
fi

if [ $INT -eq 0 ]; then
    echo "INT is zero."
else
    if [ $INT -lt 0 ]; then
        echo "INT is negative."
    else
        echo "INT is positive."
    fi
    if [ $((INT % 2)) -eq 0 ]; then
        echo "INT is even."
    else
        echo "INT is odd."
    fi
fi
```

## 27.4 更现代的test命令版本 ##

bash的最近版本包含了一个增强的test命令, 语法如下:

```
[[ expression ]]
```

expression是一个表达式, [[]] 命令和test命令类似, 只是增加了一个很重要的字符串表达式, 该表达式形式如下:

```
string1=~regex
```

如果string1与扩展的正则表达式匹配, 则返回true. 修改前面的整数表达式脚本, 加入验证是否为整数的步骤如下:

```
#!/bin/bash

# test-integer: evaluate the value of an integer.

INT=-5

if [ -z "$INT" ]; then
    echo "INT is empty." >&2
    exit 1
fi

if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
    if [ $INT -eq 0 ]; then
        echo "INT is zero."
    else
        if [ $INT -lt 0 ]; then
            echo "INT is negative."
        else
            echo "INT is positive."
        fi
        if [ $((INT % 2)) -eq 0 ]; then
            echo "INT is even."
        else
            echo "INT is odd."
        fi
    fi
else
    echo "INT is not an integer." >&2
    exit 1
fi
```

[[]] 增加的另外一个特性是 == 操作符支持模式匹配, 就像路径名扩展那样, 例如:

```
FILE=foo.bar
if [[ $FILE == foo.* ]]; then
    echo "$FILE matches pattern 'foo.*'"
fi
```

## 27.5 (())-为整数设计 ##

bash提供了 (()) 复合命令, 可用于操作整数, 该命令支持一套完整的算术计算, 它用于执行算术真值测试, 当算术计算的结果是非零值时, 则算术真值测试为true.

```
if ((1)); then echo "It is true."; fi
```

使用 (()) 命令, 简化 test-integer 脚本如下:

```
#!/bin/bash

# test-integer2a: evaluate the value of an integer.

INT=-5

if [ -z "$INT" ]; then
    echo "INT is empty." >&2
    exit 1
fi

if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
    if (( $INT == 0 )); then
        echo "INT is zero."
    else
        if (( INT < 0 )); then
            echo "INT is negative."
        else
            echo "INT is positive."
        fi
        if (( ((INT % 2)) == 0 )); then
            echo "INT is even."
        else
            echo "INT is odd."
        fi
    fi
else
    echo "INT is not an integer." >&2
    exit 1
fi
```

(()) 命令只是shell语法的一部分, 而且只能处理整数, 它可以通过名字来识别变量, 并且不需要执行扩展操作.

## 27.6 组合表达式 ##

表达式可以使用逻辑运算符组合起来, test以及 [[]] 命令使用不同的操作符, 如下表:

| Operation | test | [[]]and(()) |
|:--|:--|:--|
| AND | -a | && |
| OR | -o | \|\| |
| NOT | ! | ! |

以下是一个AND运算的例子.

```
#!/bin/bash

# test-integer3: determine if an integer is within a

# specified range of values.

MIN_VAL=1
MAX_VAL=100

INT=50

if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
    if [[ INT -ge MIN_VAL && INT -le MAX_VAL ]]; then
    # 等价与 if [ $INT -ge $MIN_VAL -a $INT -le $MAX_VAL]; then
        echo "$INT is within $MIN_VAL to $MAX_VAL."
    else
        echo "$INT is out of range."
    fi
else
    echo "INT is not an integer." >&2
    exit 1
fi
```

也可以使用以下命令来判断INT的值是否在指定的范围之外:

```
if [[ ! (INT -ge MIN_VAL && INT -le MAX_VAL) ]]; then
```

在test命令中, 所有的表达式和操作符都被shell看做命令参数(除了 (()) 和 [[]]), 因此在bash中有特殊含义的字符, 如 <, >, (, ) 必须用引号括起来或者进行转义.

```
if [ ! \( $INT -ge $MIN_VAL && $INT -le $MAX_VAL \) ]; then
```

test和 [[]] 命令基本上完成相同的功能, test更为传统, 而 [[]] 是bash特有的.

## 27.7 控制运算符: 另一种方式的分支 ##

bash还提供了两种可以执行分支的控制运算符, && 和 || 运算符与 [[]] 符合命令中的逻辑运算符类似, 语法如下:

```
command1 && command2
command1 || command2
```

对于AND运算来说, 先执行command1, 只有在它执行成功时才会执行command2.
对于OR运算来说, 先执行command1, 只有在它执行失败时才会执行command2.

## 27.8 本章结尾语 ##

回到 sys\_info\_page脚本, 我们可以检测用户是否具有读取所有家目录的权限, 实现如下:

```
report_home_page () {
    if [[ $(id -u) -eq 0 ]]; then
        cat <<- _EOF_
    <H2>Home Space Utilization (All Users)</H2>
    <PRE>$(du -sh /home/*)</PRE>
_EOF_
    else
        cat <<- _EOF_
    <H2>Home Space Utilization ($USER)</H2>
    <PRE>$(du -sh $HOME)</PRE>
_EOF_
    fi
    return
}
```

## 27.9 扩展阅读 ##

bash 手册页中有几部分对本章中涵盖的主题提供了更详细的内容:

- Lists ( 讨论控制操作符 || 和 && )
- Compound Commands ( 讨论 [[ ]], (( )) 和 if )
- CONDITIONAL EXPRESSIONS （条件表达式）
- SHELL BUILTIN COMMANDS ( 讨论 test )

进一步, Wikipedia 中有一篇关于伪代码概念的好文章:

[http://en.wikipedia.org/wiki/Pseudocode](http://en.wikipedia.org/wiki/Pseudocode)
