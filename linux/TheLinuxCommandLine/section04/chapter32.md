# 第三十二章: 位置参数 #

## 32.1 访问命令行 ##

shell提供了一组称为位置参数的变量, 用于存储命令行中的关键字, 这些变量分别命名为0-9, 使用方法如下:

```
#!/bin/bash
# posit-param: script to view command line parameters
echo "
\$0 = $0
\$1 = $1
\$2 = $2
\$3 = $3
\$4 = $4
\$5 = $5
\$6 = $6
\$7 = $7
\$8 = $8
\$9 = $9
"
```

即便没有提供任何实参, 变量 $0 总是会存储命令行显示的第一行数据, 也就是所执行程序的路径名.

使用参数扩展技术, 可以获取多余9个的参数, 为标明一个大于9的数字, 将数字用大括号括起来即可, 例如 ${211} .

### 32.1.1 确定实参的数目 ###

shell还提供了变量 $# 以给出命令行参数的数目. 例如:

```
#!/bin/bash
# posit-param: script to view command line parameters
echo "
Number of arguments: $#
\$0 = $0
\$1 = $1
\$2 = $2
\$3 = $3
\$4 = $4
\$5 = $5
\$6 = $6
\$7 = $7
\$8 = $8
\$9 = $9
"
```

### 32.1.2 shift-处理大量的实参 ###

为了处理大量的实参, shell提供了shift命令, 每次执行shift命令之后所有的参数均下移一位, 这样就可以只处理一个参数($0) 而完成全部的任务.

```
#!/bin/bash
# posit-param2: script to display all arguments
count=1
while [[ $# -gt 0 ]]; do
    echo "Argument $count = $1"
    count=$((count + 1))
    shift
done
```

### 32.1.3 简单的应用程序 ###

一个简单的使用位置参数的程序如下:

```
#!/bin/bash
# file_info: simple file information program
PROGNAME=$(basename $0)
if [[ -e $1 ]]; then
    echo -e "\nFile Type:"
    file $1
    echo -e "\nFile Status:"
    stat $1
else
    echo "$PROGNAME: usage: $PROGNAME file" >&2
    exit 1
fi
```

这个程序输出了单个特定文件的文件类型和文件状态, 并且使用basename命令移除路径名的起始部分, 只留下基本的文件名.

### 32.1.4 在shell函数中使用位置参数 ###

位置参数也可用于shell函数实参的传递, 例子如下:

```
file_info () {
  # file_info: function to display file information
  if [[ -e $1 ]]; then
      echo -e "\nFile Type:"
      file $1
      echo -e "\nFile Status:"
      stat $1
  else
      echo "$FUNCNAME: usage: $FUNCNAME file" >&2
      return 1
  fi
}
```

shell会自动更新FUNCNAME变量以追踪当前执行的shell函数, 但是变量$0包含的总是命令行第一项的路径全名.

## 32.2 处理多个位置参数 ##

shell提供了两种特殊的参数, 它们都能扩展为一个完整的位置参数列, 如下表:

| 参数 | 描述 |
|:--|:--|
| $* | 扩展为从1开始的位置参数列, 当包括在双引号内时, 扩展为双引号引用的由全部位置参数构成的字符串, 位置参数由IFS变量的第一个字符间隔开 |
| $@ | 扩展为从1开始的位置参数列, 当包括在双引号内时, 将每个位置参数扩展为双引号引用的单独单词 |

一个演示程序如下:

```
#!/bin/bash
# posit-params3 : script to demonstrate $* and $@
print_params () {
    echo "\$1 = $1"
    echo "\$2 = $2"
    echo "\$3 = $3"
    echo "\$4 = $4"
}
pass_params () {
    echo -e "\n" '$* :';      print_params   $*
    echo -e "\n" '"$*" :';    print_params   "$*"
    echo -e "\n" '$@ :';      print_params   $@
    echo -e "\n" '"$@" :';    print_params   "$@"
}
pass_params "word" "words with spaces"
```

输出如下:

```
[me@linuxbox ~]$ posit-param3
 $* :
$1 = word
$2 = words
$3 = with
$4 = spaces
 "$*" :
$1 = word words with spaces
$2 =
$3 =
$4 =
 $@ :
$1 = word
$2 = words
$3 = with
$4 = spaces
 "$@" :
$1 = word
$2 = words with spaces
$3 =
$4 =
```

$@保持了每个位置参数的完整性, 是大多数情况下最令人满意的方法.

## 32.3 更完整的应用程序 ##

现在回到 sys\_info\_page 程序, 添加如下所示的命令行选项:

- 输出文件: 指定输出文件的选项, 形式为 -f file 或者 --file file
- 交互模式: 这个选项会提示用户输出文件的名称, 并校验此文件是否已经存在. 若存在则覆盖之前提示用户. 形式为 -i 或者 --interactive
- 帮助: 输出帮助性质的使用说明, 形式为 -h或者 --help

以下是完善命令行处理功能所需的代码:

```
usage () {
    echo "$PROGNAME: usage: $PROGNAME [-f file | -i]"
    return
}
# process command line options
interactive=
filename=
while [[ -n $1 ]]; do
    case $1 in
    -f | --file)            shift
                            filename=$1
                            ;;
    -i | --interactive)     interactive=1
                            ;;
    -h | --help)            usage
                            exit
                            ;;
    *)                      usage >&2
                            exit 1
                            ;;
    esac
    shift
done
```

以下是完善交互模式的代码:

```
# interactive mode
if [[ -n $interactive ]]; then
    while true; do
        read -p "Enter name of output file: " filename
        if [[ -e $filename ]]; then
            read -p "'$filename' exists. Overwrite? [y/n/q] > "
            case $REPLY in
            Y|y)    break
                    ;;
            Q|q)    echo "Program terminated."
                    exit
                    ;;
            *)      continue
                    ;;
            esac
        elif [[ -z $filename ]]; then
            continue
        else
            break
        fi
    done
fi
```

将写页面的代码改写为shell函数:

```
write_html_page () {
    cat <<- _EOF_
        <HTML>
            <HEAD>
                <TITLE>$TITLE</TITLE>
            </HEAD>
            <BODY>
                <H1>$TITLE</H1>
                <P>$TIMESTAMP</P>
                $(report_uptime)
                $(report_disk_space)
                $(report_home_space)
            </BODY>
        </HTML>
    _EOF_
    return
}
# output html page
if [[ -n $filename ]]; then
    if touch $filename && [[ -f $filename ]]; then
        write_html_page > $filename
    else
        echo "$PROGNAME: Cannot write file '$filename'" >&2
        exit 1
    fi
else
    write_html_page
fi
```

## 32.4 本章结尾语 ##

在位置参数的帮助下, 用户可以写出功能性更强的脚本. 对于重复性的任务, 位置参数使得用户可以写出很有帮助的shell函数, 并放置在用户的 .bashrc 文件中.

下面是完整的 sys\_info\_page 程序:

```
#!/bin/bash
# sys_info_page: program to output a system information page
PROGNAME=$(basename $0)
TITLE="System Information Report For $HOSTNAME"
CURRENT_TIME=$(date +"%x %r %Z")
TIMESTAMP="Generated $CURRENT_TIME, by $USER"
report_uptime () {
    cat <<- _EOF_
        <H2>System Uptime</H2>
        <PRE>$(uptime)</PRE>
    _EOF_
    return
}
report_disk_space () {
    cat <<- _EOF_
        <H2>Disk Space Utilization</H2>
        <PRE>$(df -h)</PRE>
    _EOF_
    return
}
report_home_space () {
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
usage () {
    echo "$PROGNAME: usage: $PROGNAME [-f file | -i]"
    return
}
write_html_page () {
    cat <<- _EOF_
        <HTML>
            <HEAD>
                <TITLE>$TITLE</TITLE>
            </HEAD>
            <BODY>
                <H1>$TITLE</H1>
                <P>$TIMESTAMP</P>
                $(report_uptime)
                $(report_disk_space)
                $(report_home_space)
            </BODY>
        </HTML>
    _EOF_
    return
}
# process command line options
interactive=
filename=
while [[ -n $1 ]]; do
    case $1 in
        -f | --file)          shift
                              filename=$1
                              ;;
        -i | --interactive)   interactive=1
                              ;;
        -h | --help)          usage
                              exit
                              ;;
        *)                    usage >&2
                              exit 1
                              ;;
    esac
    shift
done
# interactive mode
if [[ -n $interactive ]]; then
    while true; do
        read -p "Enter name of output file: " filename
        if [[ -e $filename ]]; then
            read -p "'$filename' exists. Overwrite? [y/n/q] > "
            case $REPLY in
                Y|y)    break
                        ;;
                Q|q)    echo "Program terminated."
                        exit
                        ;;
                *)      continue
                        ;;
            esac
        fi
    done
fi
# output html page
if [[ -n $filename ]]; then
    if touch $filename && [[ -f $filename ]]; then
        write_html_page > $filename
    else
        echo "$PROGNAME: Cannot write file '$filename'" >&2
        exit 1
    fi
else
    write_html_page
fi
```

- *Bash Hackers Wiki* 上有一篇不错的关于位置参数的文章:

[http://wiki.bash-hackers.org/scripting/posparams](http://wiki.bash-hackers.org/scripting/posparams)

- Bash 的参考手册有一篇关于特殊参数的文章, 包括 $* 和 $@:

[http://www.gnu.org/software/bash/manual/bashref.html#Special-Parameters](http://www.gnu.org/software/bash/manual/bashref.html#Special-Parameters)

- 除了本章讨论的技术之外, bash 还包含一个叫做 getopts 的内部命令, 此命令也可以用来处理命令行参数. bash 参考页面的 *SHELL BUILTIN COMMANDS* 一节介绍了这个命令, *Bash Hackers Wiki* 上也有对它的描述:

[http://wiki.bash-hackers.org/howto/getopts_tutorial](http://wiki.bash-hackers.org/howto/getopts_tutorial)
