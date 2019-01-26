# 第三十六章: 其他命令 #

## 36.1 组命令和子shell ##

bash允许将命令组合到一起使用, 有两种方式, 分别是组命令和子shell:

```
# 组命令
{ command1; command2; [command3; ...] }
# 子shell
(command1; command2; [command3; ...])
```

需要注意的是, 在实现组命令时必须使用一个空格将花括号与命令分开, 并且在闭花括号前使用分号或是换行来结束最后的命令.

### 36.1.1 执行重定向 ###

可以使用组命令和子shell来管理重定向:

```
ls -l > output.txt
echo "Listing of foo.txt" >> output.txt
cat foo.txt >> output.txt

# 等价于
{ ls -l; echo "Listing of foo.txt"; cat foo.txt; } > output.txt
# 或是
(ls -l; echo "Listing of foo.txt"; cat foo.txt) > output.txt
```

当创建命令管道时, 通常将多条命令的结果输出到一条流中, 这比较有用:

```
{ ls -l; echo "Listing of foo.txt"; cat foo.txt; } | lpr
```

在下面的脚本中我们将使用组命令, 看几个与关联数组结合使用的编程技巧.
这个脚本称为 array-2, 当给定一个目录名, 打印出目录中的文件列表, 伴随着每个文件的文件所有者和组所有者.
在文件列表的末尾, 脚本打印出属于每个所有者和组的文件数目. 这里我们看到的（为简单起见而缩短的）结果, 是给定脚本的目录为 /usr/bin 的时候:

```
[me@linuxbox ~]$ array-2 /usr/bin
/usr/bin/2to3-2.6                 root        root
/usr/bin/2to3                     root        root
/usr/bin/a2p                      root        root
/usr/bin/abrowser                 root        root
/usr/bin/aconnect                 root        root
/usr/bin/acpi_fakekey             root        root
/usr/bin/acpi_listen              root        root
/usr/bin/add-apt-repository       root        root
.
/usr/bin/zipgrep                  root        root
/usr/bin/zipinfo                  root        root
/usr/bin/zipnote                  root        root
/usr/bin/zip                      root        root
/usr/bin/zipsplit                 root        root
/usr/bin/zjsdecode                root        root
/usr/bin/zsoelim                  root        root

File owners:
daemon  : 1 file(s)
root    : 1394 file(s) File group owners:
crontab : 1 file(s)
daemon  : 1 file(s)
lpadmin : 1 file(s)
mail    : 4 file(s)
mlocate : 1 file(s)
root    : 1380 file(s)
shadow  : 2 file(s)
ssh     : 1 file(s)
tty     : 2 file(s)
utmp    : 2 file(s)
```

脚本代码如下:

```
#!/bin/bash

# array-2: Use arrays to tally file owners

declare -A files file_group file_owner groups owners

if [[ ! -d "$1" ]]; then
   echo "Usage: array-2 dir" >&2
   exit 1
fi

for i in "$1"/*; do
   owner=$(stat -c %U "$i")
   group=$(stat -c %G "$i")
    files["$i"]="$i"
    file_owner["$i"]=$owner
    file_group["$i"]=$group
    ((++owners[$owner]))
    ((++groups[$group]))
done

# List the collected files
{ for i in "${files[@]}"; do
printf "%-40s %-10s %-10s\n" \
"$i" ${file_owner["$i"]} ${file_group["$i"]}
done } | sort
echo

 List owners
echo "File owners:"
{ for i in "${!owners[@]}"; do
printf "%-10s: %5d file(s)\n" "$i" ${owners["$i"]}
done } | sort
echo

# List groups
echo "File group owners:"
{ for i in "${!groups[@]}"; do
printf "%-10s: %5d file(s)\n" "$i" ${groups["$i"]}
done } | sort
```

### 36.1.2 进程替换 ###

组命令和子shell有一个主要的不同: 子shell在当前shell的子拷贝中执行命令, 而组命令在当前shell里面执行命令. 通常情况下组命令更合适, 除非脚本真的需要子shell.

在之前的章节中, 我们接触到了一个子shell环境导致的问题实例:

```
echo "foo" | read
```

以上的命令并不能对REPLY变量进行赋值, 因为read命令是在子shell中执行的. 由于总是在子shell中执行管道中的命令, 所以任何变量赋值都会遇到这个问题. shell提供了一种称为进程替换的外部扩展方式来解决这个问题:

实现进程替换的方式有两种, 一种是产生标准输出的进程:

```
<(list)
```

另一种是吸纳标准输入的进程:

```
>(list)
```

这里的list是一系列命令. 为了解决上述read命令的问题, 可以像如下方式那样来使用进程替换:

```
read < <(echo "foo")
```

进程替换允许把子shell的输出当作一个普通的文件, 目的是为了重定向. 事实上这是一种扩展形式, 我们可以查看它的真实值:

```
[me@linuxbox ~]$ echo <(echo "foo")
/dev/fd/63
```

进程替换通常结合带有read的循环使用, 例如如下代码:

```
#!/bin/bash
# pro-sub : demo of process substitution
while read attr links owner group size date time filename; do
    cat <<- EOF
        Filename:     $filename
        Size:         $size
        Owner:        $owner
        Group:        $group
        Modified:     $date $time
        Links:        $links
        Attributes:   $attr
    EOF
done < <(ls -l | tail -n +2)
```

产生的输出如下:

```
[me@linuxbox ~]$ pro_sub | head -n 20
Filename: addresses.ldif
Size: 14540
Owner: me
Group: me
Modified: 2009-04-02 11:12
Links:
1
Attributes: -rw-r--r--
Filename: bin
Size: 4096
Owner: me
Group: me
Modified: 2009-07-10 07:31
Links: 2
Attributes: drwxr-xr-x
Filename: bookmarks.html
Size: 394213
Owner: me
Group: me
```

## 36.2 trap ##

bash提供了trap机制来让脚本响应信号, 语法如下:

```
trap argument signal [signal...]
```

这里的argument是作为命令被读取的字符串, 而signal是对信号量的说明, 该信号量会触发解释命令的执行. 例如:

```
#!/bin/bash
# trap-demo : simple signal handling demo
trap "echo 'I am ignoring you.'" SIGINT SIGTERM
for i in {1..5}; do
    echo "Iteration $i of 5"
    sleep 5
done
```

当脚本收到 SIGTERM或是SIGINT信号时, 脚本定义的trap将执行echo命令.

通常我们使用shell函数来代替命令, 将代码修改如下:

```
#!/bin/bash
# trap-demo2 : simple signal handling demo
exit_on_signal_SIGINT () {
    echo "Script interrupted." 2>&1
    exit 0
}
exit_on_signal_SIGTERM () {
    echo "Script terminated." 2>&1
    exit 0
}
trap exit_on_signal_SIGINT SIGINT
trap exit_on_signal_SIGTERM SIGTERM
for i in {1..5}; do
    echo "Iteration $i of 5"
    sleep 5
done
```

类UNIX系统通常在/tmp目录下创建临时文件, 为了给临时文件一个不可预知的文件名, 可以使用如下代码:

```
tempfile=/tmp/$(basename $0).$$.$RANDOM
```

这会创建一个包含程序名的文件, 其后是进程ID, 然后是一个1~32767之间的整数.

更好的办法是使用 mktemp程序来命名和创建临时文件:

```
tempfile=$(mktemp /tmp/foobar.$$.XXXXXXXXXX)
```

mktemp使用模板作为参数来创建文件名, 该模板包含一系列的X字符, mktemp用随机的字母和数字来替换这些X字符. mktemp构造了一个临时文件名, 同时也创建了这个文件.

如果是普通用户执行的脚本, 较好的做法是避免使用 /tmp 目录, 而在用户主目录下为临时文件创建一个目录, 代码如下:

```
[[ -d $HOME/tmp ]] || mkdir $HOME/tmp
```

## 36.3 异步执行 ##

bash提供了wait命令让父脚本暂停, 直到指定的进程(例如子脚本)结束.

### 36.3.1 wait命令 ###

父脚本如下:

```
#!/bin/bash
# async-parent : Asynchronous execution demo (parent)
echo "Parent: starting..."
echo "Parent: launching child script..."
async-child &
pid=$!
echo "Parent: child (PID= $pid) launched."
echo "Parent: continuing..."
sleep 2
echo "Parent: pausing to wait for child to finish..."
wait $pid
echo "Parent: child is finished. Continuing..."
echo "Parent: parent is done. Exiting."
```

子脚本如下:

```
#!/bin/bash
# async-child : Asynchronous execution demo (child)
echo "Child: child is running..."
sleep 5
echo "Child: child is done. Exiting."
```

$! 变量的值总是包含后台中最后一次运行的进程ID. 输出如下:

```
[me@linuxbox ~]$ async-parent
Parent: starting...
Parent: launching child script...
Parent: child (PID= 6741) launched.
Parent: continuing...
Child: child is running...
Parent: pausing to wait for child to finish...
Child: child is done. Exiting.
Parent: child is finished. Continuing...
Parent: parent is done. Exiting.
```

## 36.4 命名管道 ##

大多数类UNIX系统支持创建一种叫做命名管道的特殊类型的文件, 使用命名管道可以建立两个进程之间的通信, 并且可以像其他类型的文件一样使用.

命名管道的工作方式和文件相似, 命名管道实际上是两块先进先出的缓冲区, 数据从一端进入, 从另一端出去. 可以通过如下方式使用命名管道:

```
process1 > named_pipe
process2 < named_pipe

# 等价于
process1 | process2
```

### 36.4.1 设置命名管道 ###

可以使用mkfifo命令创建命名管道:

```
mkfifo pipe1
```

命名管道文件的属性字段的第一个字母是p.

### 36.4.2 使用命名管道 ###

打开两个终端, 在第一个终端中输入:

```
ls -l > pipe1
```

这时命令会挂起, 因为命名管道被阻塞. 在第二个终端输入:

```
cat < pipe1
```

可以看到第一个终端中命令的结果出现在第二个终端中, 同时第一个终端中的ls命令结束.

## 36.5 本章结尾语 ##

## 36.6 扩展阅读 ##

- bash 手册页的 *复合命令* 部分包含了对组命令和子 shell 表示法的详尽描述.

- bash 手册也的 *EXPANSION* 部分包含了一小部分进程替换的内容.

- 《高级 Bash 脚本指南》也有对进程替换的讨论:

[http://tldp.org/LDP/abs/html/process-sub.html](http://tldp.org/LDP/abs/html/process-sub.html)

- 《Linux 杂志》有两篇关于命令管道的好文章. 第一篇源于1997年9月:

[http://www.linuxjournal.com/article/2156](http://www.linuxjournal.com/article/2156)

- 和第二篇源于2009年3月:

[http://www.linuxjournal.com/content/using-named-pipes-fifos-bash](http://www.linuxjournal.com/content/using-named-pipes-fifos-bash)
