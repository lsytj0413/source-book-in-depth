# 第三十章: 故障诊断 #

本章讲解一些脚本中常见的错误类型以及几种用于追踪和解除错误的有用技巧.

## 30.1 语法错误 ##

语法错误是一种常见的错误类型, 其中就包括shell语句中一些元素的拼写错误. 在大多数情况下shell会拒绝执行含有此种类型错误的脚本.

我们将使用以下脚本来演示常见的错误类型:

```
#!/bin/bash

# trouble: script to demonstrate common errors

number=1

if [ $number = 1 ]; then
    echo "Number is euqal to 1."
else
    echo "Number is not euqal to 1."
fi
```

### 30.1.1 引号缺失 ###

修改上述脚本, 删除第一个echo命令实参后的双引号:

```
number=1

if [ $number = 1 ]; then
    echo "Number is euqal to 1.
else
    echo "Number is not euqal to 1."
fi
```

运行脚本, 出现的现象如下:

```
./test.sh: 行 269: 寻找匹配的 `"' 是遇到了未预期的文件结束符
./test.sh: 行 272: 语法错误: 未预期的文件结尾
```

脚本产生了两个错误, 错误报告指出的行不是删除所在的行, 而是之后的代码. 这是因为shell读取到删除双引号的位置之后, 继续向下寻找对应的双引号, 直到第二个echo命令的双引号处, 导致if命令的语法结构被破坏.

使用带语法结构突出显示的编辑器能够帮助寻找这类错误, 如果是vim可以使用以下命令启动vim的语法结构突出显示功能:

```
:syntax on
```

### 30.1.2 符号缺失冗余 ###

if或while命令结构不完整也是常见的错误, 例如删除if命令中test部分后的分号如下:

```
if [ $number = 1 ] then
    echo "Number is euqal to 1."
else
    echo "Number is not euqal to 1."
fi
```

运行脚本, 得到如下的输出:

```
./test.sh: 行 268: 未预期的符号 `else' 附近有语法错误
./test.sh: 行 268: `else'
```

这个错误的原因如下: if接收一系列的命令, 并评估最后一个命令的退出码. 在本例中, 这个命令系列只由一条命令组成, [ 为test的同义表示, 之后的部分视为参数——即 $number, =, 1 和 ], 在分号被删除后, then也被添加到参数列表,
接下来的echo命令被翻译为if接收的命令列表中的第二条命令, 并成为if的退出码. 最后遇到else, 但是else 是shell的保留词, 所以出现了上述错误信息.

### 30.1.3 非预期的展开 ###

有些错误是间歇出现的, 现在将number的值修改为空, 代码如下:

```
number=
if [ $number = 1 ]; then
    echo "Number is euqal to 1."
else
    echo "Number is not euqal to 1."
fi
```

运行脚本, 得到如下的输出:

```
./test.sh: 第 265 行: [: =: 需要一元表达式
Number is not euqal to 1.
```

这个错误的原因是test命令中的number变量展开, 将test命令变为以下形式:

```
[ = 1 ]
```

等式无效, 即产生了错误. = 是一个二元操作符, 在本例中缺少了一个值, 所以test命令要求程序改用一元操作符. 接下来, 因为test命令不成立(因为上述错误), if命令接收到一个非零的退出码, 从而执行了第二个echo命令.

可以通过在test命令中使用双引号引用第一个参数的方式来更在这个错误.

## 30.2 逻辑错误 ##

以下几种逻辑错误是脚本中常见的:

- 条件表达错误: 例如相反的或者不完整的逻辑表达式
- 从1开始错误: 在使用计数器的循环中忽略了程序需要从0开始而不是从1开始
- 非预期的情形: 开发人员没有预期到的数据或环境

### 30.2.1 防御编程 ###

在编程中核实各种假设是非常重要的. 例如需要删除某目录下的文件:

```
cd $dir_name
rm *
```

当在dir_name不存在时会删除当前目录下的文件(因为cd命令会失败, 导致停留在当前工作目录), 这可以肯定基本不是预期的结果. 以下是一种改进方式:

```
cd $dir_name && rm *
```

以上代码可以避免dir\_name不存在时的情况, 但是在dir\_name 为空时存在用户的主目录被删除的可能性, 可以使用以下代码继续改进:

```
[[ -d $dir_name ]] && cd $dir_name && rm *
```

通常来说, 我们需要在上述情形中给出相应的提示, 并可能需要终止脚本的运行:

```
if [[ -d $dir_name ]]; then
    if cd $dir_name; then
        rm *
    else
        echo "cannot cd to '$dir_name'" >&2
        exit 1
    fi
else
    echo "no such directory: '$dir_name'" >&2
    exit 1
fi
```

### 30.2.2 输入值验证 ###

若程序需要获取用户的输入, 那么程序必须能够处理任何输入值, 通常意味着程序必须仔细检查输入值, 以保证有效的输入可用于进一步的处理过程.
例如用以下命令来验证菜单的选择结果是否有效:

```
[[ $REPLY =~ ^[0-3]$ ]]
```

## 30.3 测试 ##

### 30.3.1 桩 ###

在脚本开发的最初阶段, 桩就是可用于查看工作进程的重要手段. 例如:

```
if [[ -d $dir_name ]]; then
    if cd $dir_name; then
        echo rm *    # TESTING
    else
        echo "cannot cd to '$dir_name'" >&2
        exit 1
    fi
else
    echo "no such directory: '$dir_name'" >&2
    exit 1
fi
exit    # TESTING
```

在上述的代码中, 通过echo命令来显示rm命令和扩展的命令参数, 并添加一些测试的注释.

### 30.3.2 测试用例 ###

测试需要选择能够反映边缘测试的输入数据和操作条件. 例如上述代码片段中需要测试以下三种条件下的代码执行情况:

- dir_name包含的是存在的目录名
- dir_name包含的是不存在的目录名
- dir_name为空

## 30.4 调试 ##

### 30.4.1 找到问题域 ###

在一些脚本中, 将问题相关的脚本部分隔离出来是很有必要的. 示例如下:

```
if [[ -d $dir_name ]]; then
    if cd $dir_name; then
        rm *
    else
        echo "cannot cd to '$dir_name'" >&2
        exit 1
    fi
# else
#     echo "no such directory: '$dir_name'" >&2
#     exit 1
fi
```

### 30.4.2 追踪 ###

一种追踪技术是通过在脚本中添加通知信息的方式来展示程序执行之处, 例如:

```
echo "preparing to delete files" >&2
if [[ -d $dir_name ]]; then
    if cd $dir_name; then
echo "deleting files" >&2
        rm *
    else
        echo "cannot cd to '$dir_name'" >&2
        exit 1
    fi
else
    echo "no such directory: '$dir_name'" >&2
    exit 1
fi
echo "file deletion complete" >&2
```

将这些追踪信息发送到标准错误, 从而与一般的程序输出区分开. 这些信息所在的行没有进行缩进, 这能使这些代码在需要删除的时候易于查找.

bash也提供了一种追踪的方法, 即直接使用 -x 选或set命令加 -x 选项.

可以在脚本的第一行添加 -x 选项激活对整个脚本的追踪活动:

```
#!/bin/bash -x
# trouble: script to demonstrate common errors
number=1
if [ $number = 1 ]; then
    echo "Number is equal to 1."
else
    echo "Number is not equal to 1."
fi
```

激活追踪之后, 我们就可以看到变量展开的执行情况. 每个输出行开头的加号代表此行是系统的追踪信息, 是追踪信息的默认特征, 由shell的变量PS4设定, 用户可以修改变量值使追踪信息提供更多的帮助信息. 例如包含脚本行号:

```
export PS4='$LINENO + '
```

对脚本的某一部分进行追踪, 可以使用 set命令加上 -x 选项:

```
#!/bin/bash
# trouble: script to demonstrate common errors
number=1
set -x # Turn on tracing
if [ $number = 1 ]; then
    echo "Number is equal to 1."
else
    echo "Number is not equal to 1."
fi
set +x # Turn off tracing
```

使用 -x 选项来激活追踪, 使用 +x 选项来解除追踪.

### 30.4.3 运行过程中变量的检验 ###

可以使用 echo 命令输出变量的值, 已确认运行过程中的变量如预期那样赋值:

```
#!/bin/bash
# trouble: script to demonstrate common errors
number=1
echo "number=$number" # DEBUG
set -x # Turn on tracing
if [ $number = 1 ]; then
    echo "Number is equal to 1."
else
    echo "Number is not equal to 1."
fi
set +x # Turn off tracing
```

## 30.5 本章结尾语 ##

调试是一门在实践中成长的技术, 既包括避免BUG, 也包括找到BUG.

## 30.6 扩展阅读 ##

- Wikipedia 上面有两篇关于语法和逻辑错误的短文:

[http://en.wikipedia.org/wiki/Syntax_error](http://en.wikipedia.org/wiki/Syntax_error)
[http://en.wikipedia.org/wiki/logic_error](http://en.wikipedia.org/wiki/logic_error)

- 网上有很多关于技术层面的 bash 编程的资源:

[http://mywiki.wooledge.org/BashPitfalls](http://mywiki.wooledge.org/BashPitfalls)

[http://tldp.org/LDP/abs/html/gotchas.html](http://tldp.org/LDP/abs/html/gotchas.html)

[http://www.gnu.org/software/bash/manual/html_node/Reserved-Word-Index.html](http://www.gnu.org/software/bash/manual/html_node/Reserved-Word-Index.html)

- 想要学习从编写良好的 Unix 程序中得知的基本概念, 可以参考 Eric Raymond 的《Unix 编程的艺术》这本 伟大的著作. 书中的许多想法都能适用于 shell 脚本:

[http://www.faqs.org/docs/artu/](http://www.faqs.org/docs/artu/)

[http://www.faqs.org/docs/artu/ch01s06.html](http://www.faqs.org/docs/artu/ch01s06.html)

- 对于真正的高强度的调试, 参考这个 Bash Debugger:

[http://bashdb.sourceforge.net/](http://bashdb.sourceforge.net/)
