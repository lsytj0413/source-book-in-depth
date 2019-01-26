# 第二十九章: 流控制: WHILE和UNTIL循环 #

## 29.1 循环 ##

循环是一系列重复的步骤, 循环中的动作会一直重复, 直到条件为真.

## 29.2 while ##

while命令的语法结构如下:

```
while commands; do commands; done
```

例如按顺序显示1～5的脚本如下:

```
#!/bin/bash

# while-count: display a series of numbers

count=1

while [ $count -le 5 ]; do
    echo $count
    count=$(( count + 1 ))
done
echo "Finished."
```

while循环会判断一系列指令的退出状态, 如果退出状态为0则执行循环内的命令.

使用while循环改写 read-menu程序如下:

```
#!/bin/bash

# read-menu: a menu driven system information program

DELAY=3 # Number of seconds to display results

while [[ $REPLY != 0 ]]; do

    clear
    cat <<- EOF
Please Select:

1. Display System Information
2. Display Disk Space
3. Display Home Space Utilization
0. Quit
EOF

    read -p "Enter selection [0-3] > "
    if [[ $REPLY =~ ^[0-3]$ ]]; then
        if [[ $REPLY == 1 ]]; then
            echo "Hostname: $HOSTNAME"
            uptime
            sleep $DELAY
        fi
        if [[ $REPLY == 2 ]]; then
            df -h
            sleep $DELAY
        fi
        if [[ $REPLY == 3 ]]; then
            if [[ $(id -u) -eq 0 ]]; then
                echo "Home Space Utilization (All Users)"
                du -sh /home/*
            else
                echo "Home Space Utilization ($USER)"
                du -sh $HOME
            fi
            sleep $DELAY
        fi
    else
        echo "Invalid entry." >&2
        exit 1
    fi
done

echo "Program terminated."
```

## 29.3 跳出循环 ##

bash提供了两种可用于控制循环内部程序流的内建命令: break命令立即终止循环, 程序从循环后的语句恢复执行; continue命令则会导致程序跳过循环剩余部分, 直接从下一次循环迭代.

使用break以及continue命令重写 read-menu程序如下:

```
#!/bin/bash

# read-menu: a menu driven system information program

DELAY=3 # Number of seconds to display results

while true; do

    clear
    cat <<- EOF
Please Select:

1. Display System Information
2. Display Disk Space
3. Display Home Space Utilization
0. Quit
EOF

    read -p "Enter selection [0-3] > "
    if [[ $REPLY =~ ^[0-3]$ ]]; then
        if [[ $REPLY == 0 ]]; then
            break
        fi
        if [[ $REPLY == 1 ]]; then
            echo "Hostname: $HOSTNAME"
            uptime
            sleep $DELAY
            continue
        fi
        if [[ $REPLY == 2 ]]; then
            df -h
            sleep $DELAY
            continue
        fi
        if [[ $REPLY == 3 ]]; then
            if [[ $(id -u) -eq 0 ]]; then
                echo "Home Space Utilization (All Users)"
                du -sh /home/*
            else
                echo "Home Space Utilization ($USER)"
                du -sh $HOME
            fi
            sleep $DELAY
            continue
        fi
    else
        echo "Invalid entry." >&2
        exit 1
    fi
done

echo "Program terminated."
```

## 29.4 until ##

until命令与while命令相反, 它在退出状态不为0时终止循环. 使用until命令改写 while-count 脚本如下:

```
#!/bin/bash

# until-count: display a series of numbers

count=1

while [ $count -gt 5 ]; do
    echo $count
    count=$(( count + 1 ))
done
echo "Finished."
```

## 29.5 使用循环读取文件 ##

while和until可以处理标准输入, 这让使用while和until循环处理文件成为可能. 例如显示 distros.txt文件的内容:

```
#!/bin/bash

# while-read: read lines from a file

while read distro version release; do
    printf "Distro: %s\tVersion: %s\tReleased: %s\n" \
           $distro \
           $version \
           $release
done < distros.txt
```

我们在done语句之后添加重定向操作符, 以便将一份文件重定向到循环中.
我们也可以把标准输入重定向到循环中:

```
#!/bin/bash

# while-read2: read lines from a file

sort -k 1,1 -k 2n distros.txt | while read distro version release; do
        printf "Distro: %s\tVersion: %s\tReleased: %s\n" \
               $distro \
               $version \
               $release
done
```

以上的脚本获取sort命令的输出并显示文本流. 需要注意的是, 管道是在子shell中进行循环操作, 当循环终止时, 所有的循环内部新建的变量以及对变量的赋值效果都会消失.

## 29.6 本章结尾语 ##

在讲解过循环, 分支, 子进程以及队列之后, 编程过程中使用的主要流控制方法已经都介绍完成.

## 29.7 扩展阅读 ##

- Linux 文档工程中的 Bash 初学者指南一书中介绍了更多的 while 循环实例:

[http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_09_02.html](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_09_02.html)

- Wikipedia 中有一篇关于循环的文章, 其是一篇比较长的关于流程控制的文章中的一部分:

[http://en.wikipedia.org/wiki/Control_flow#Loops](http://en.wikipedia.org/wiki/Control_flow#Loops)
