# 第三十三章: 流控制: for循环 #

在新版bash中, for命令存在两种形式.

## 33.1 for: 传统shell形式 ##

最原始的for命令语法如下:

```
for variable [in words]; do
    commands
done
```

其中variable是一个在循环时会增值的变量名, words是一列将按顺序赋值给变量variable的可选项, commands是每次循环时都会执行的命令. 例如:

```
for i in A B C D; do echo $i; done
```

for命令创建字符列表的方式有多种, 简要介绍如下:

```
# 花括号扩展
for i in {A..D}; do echo $i; done
# 路径名扩展
for i in distros*.txt; do echo $i; done
```

也可以使用命令形式:

```
#!/bin/bash
# longest-word : find longest string in a file
while [[ -n $1 ]]; do
    if [[ -r $1 ]]; then
        max_word=
        max_len=0
        for i in $(strings $1); do
            len=$(echo $i | wc -c)
            if (( len > max_len )); then
                max_len=$len
                max_word=$i
            fi
        done
        echo "$1: '$max_word' ($max_len characters)"
    fi
    shift
done
```

以上程序中使用strings命令读取文件并产生一个可读文本的字符, 然后使用for命令循环处理每个字符以找到最长的字符.

如果for命令中字符部分的选项被忽略, 则for循环默认处理该位置参数, 使用这种方式修改上述脚本如下:

```
#!/bin/bash
# longest-word2 : find longest string in a file
for i; do
    if [[ -r $i ]]; then
        max_word=
        max_len=0
        for j in $(strings $i); do
            len=$(echo $j | wc -c)
            if (( len > max_len )); then
                max_len=$len
                max_word=$j
            fi
        done
        echo "$i: '$max_word' ($max_len characters)"
    fi
done
```

## 33.2 for: C语言形式 ##

第二种for命令语法类似于C语言形式, 语法如下:

```
for (( expression1; expression2; expression3 )); do
    commands
done
```

等价与如下结构:

```
(( expression1 ))
while (( expression2 )); do
    commands
    (( expression3 ))
done
```

其中expression1 用来初始化循环条件, expression2 用来决定循环何时结束, expression3 在每次循环末尾执行.

一个典型的应用如下:

```
#!/bin/bash
# simple_counter : demo of C style for command
for (( i=0; i<5; i=i+1 )); do
    echo $i
done
```

## 33.3 本章结尾语 ##

重写report\_home\_space函数, 为每个用户的主目录提供更多的细节, 并且包含用户的文件总数和子目录总数, 代码如下:

```
report_home_space () {
    local format="%8s%10s%10s\n"
    local i dir_list total_files total_dirs total_size user_name
    if [[ $(id -u) -eq 0 ]]; then
        dir_list=/home/*
        user_name="All Users"
    else
        dir_list=$HOME
        user_name=$USER
    fi
    echo "<H2>Home Space Utilization ($user_name)</H2>"
    for i in $dir_list; do
        total_files=$(find $i -type f | wc -l)
        total_dirs=$(find $i -type d | wc -l)
        total_size=$(du -sh $i | cut -f 1)
        echo "<H3>$i</H3>"
        echo "<PRE>"
        printf "$format" "Dirs" "Files" "Size"
        printf "$format" "----" "-----" "----"
        printf "$format" $total_dirs $total_files $total_size
        echo "</PRE>"
    done
    return
}
```
