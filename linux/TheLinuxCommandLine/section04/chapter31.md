# 第三十一章: 流控制: case分支 #

## 31.1 case ##

shell的多项选择复合命令称为case, 语法如下:

```
case word in
    [pattern [| pattern]...) commands ;;]...
esac
```

使用case命令简化read-menu程序如下:

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

case $REPLY in
    0)         echo "Program terminated."
               exit
               ;;
    1)         echo "Hostname: $HOSTNAME"
               uptime
               ;;
    2)         df -h
               ;;
    3)         if [[ $(id -u) -eq 0 ]]; then
                   echo "Home Space Utilization (All Users)"
                   du -sh /home/*
               else
                   echo "Home Space Utilization ($USER)"
                   du -sh $HOME
               fi
               ;;
    *)         echo "Invalid entry." >&2
               exit 1
               ;;
esac
```

case命令将关键字的值与特定的模式相比较, 若发现吻合的模式则执行与此模式相联系的命令, 并不再对比剩余的模式.

### 31.1.1 模式 ###

同路径名扩展一样, case使用以 ) 字符结尾的模式, 下表列出了一些常见的模式:

| 模式 | 描述 |
|:--|:--|
| a) | 若关键字为a则吻合 |
| [[:alpha:]]) | 若关键字为单个字母则吻合 |
| ???) | 若关键字为三个字符则吻合 |
| *.txt) | 若关键字以.txt结尾则吻合 |
| *) | 匹配所有关键字. 通常放于case命令的最后一个模式, 处理所有与前模式不吻合的关键字 |

一个例子如下:

```
#!/bin/bash

read -p "enter word > "

case $REPLY in
    [[:alpha:]])    echo "is a single alphabetic character." ;;
    [ABC][0-9])     echo "is A, B, or C followed by a digit." ;;
    ???)            echo "is three characters long." ;;
    *.txt)          echo "is a word ending in '.txt'" ;;
    *)              echo "is something else." ;;
esac
```

### 31.1.2 多个模式的组合 ###

可以使用竖线作为分隔符来组合多个模式, 模式之间是或的条件关系. 一个例子如下:

```
#!/bin/bash

read-menu: a menu driven system information program

clear
echo "
Please Select:

A. Display System Information
B. Display Disk Space
C. Display Home Space Utilization
Q. Quit
"

read -p "Enter selection [A, B, C or Q] > "

case $REPLY in
    q|Q)         echo "Program terminated."
                 exit
                 ;;
    a|A)         echo "Hostname: $HOSTNAME"
                 uptime
                 ;;
    b|B)         df -h
                 ;;
    c|C)         if [[ $(id -u) -eq 0 ]]; then
                     echo "Home Space Utilization (All Users)"
                     du -sh /home/*
                 else
                     echo "Home Space Utilization ($USER)"
                     du -sh $HOME
                 fi
                 ;;
    *)           echo "Invalid entry." >&2
                 exit 1
                 ;;
esac
```

在上例中, 用户即可以输入大写字母进行选择, 也可以输入小写字母进行选择.

## 31.2 本章结尾语 ##
