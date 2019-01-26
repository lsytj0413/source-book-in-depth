# 第三十五章: 数组 #

本章介绍一种包含多个值的数据结构--数组.

## 35.1 什么是数组 ##

数组是可以一次存放多个值的变量, 组织形式如同表格. 数组中的单元叫做元素, 并且每个元素含有数据, 使用一种叫做索引或是下标的地址就可以访问一个独立的数组元素.

bash的数组是一维的, bash的第二个版本开始提供对数组的支持, 而最初的UNIX shell程序sh是不支持数组的.

## 35.2 创建一个数组 ##

命名数组变量同命名其他shell变量一样, 当访问数组变量时可以自动创建它们. 例如:

```
a[1]=foo
echo ${a[1]}
```

在输出变量时使用花括号是为了阻止shell在数组元素名里试图扩展路径名. 也可以使用 declare 命令创建数组, 如下:

```
declare -a a
```

## 35.3 数组赋值 ##

使用下面的语法可以对数组的单一元素赋值:

```
name[subscript]=value
```

这里name是数组名, subscript是大于等于0的整数, value是赋值给数组元素的字符串或整数.

使用下面的语法可以对整个数组赋值:

```
name=(value1 value2...)
```

这里name是数组名, 并且将value1, value2...等值依次赋予从索引0开始的数组元素. 也可以通过为每个值指定下标来对特定的元素赋值:

```
days=(Sun Mon Tue Wed Thu Fri Sat)
# 等价于
days=([0]=Sun [1]=Mon [2]=Tue [3]=Wed [4]=Thu [5]=Fri [6]=Sat)
```

## 35.4 访问数组元素 ##

创建一个脚本, 用于输出特定目录中文件的修改次数, 示例输出如下:

```
[me@linuxbox ~]$ hours .
Hour Files Hour Files
---- ----- ---- ----
00   0     12   11
01   1     13   7
02   0     14   1
03   0     15   7
04   1     16   6
04   1     17   5
06   6     18   4
07   3     19   4
08   1     20   1
09   14    21   0
10   2     22   0
11   5     23   0
Total files = 80
```

hours程序输出在指定目录中, 一天中的每个小时有多少文件经过最后一次修改. 该脚本代码如下:

```
#!/bin/bash
# hours : script to count files by modification time
usage () {
    echo "usage: $(basename $0) directory" >&2
}
# Check that argument is a directory
if [[ ! -d $1 ]]; then
    usage
    exit 1
fi
# Initialize array
for i in {0..23}; do hours[i]=0; done
# Collect data
for i in $(stat -c %y "$1"/* | cut -c 12-13); do
    j=${i/#0}
    ((++hours[j]))
    ((++count))
done
# Display data
echo -e "Hour\tFiles\tHour\tFiles"
echo -e "----\t-----\t----\t-----"
for i in {0..11}; do
    j=$((i + 12))
    printf "%02d\t%d\t%02d\t%d\n" $i ${hours[i]} $j ${hours[j]}
done
printf "\nTotal files = %d\n" $count
```

在第三部分代码中, 我们使用stat程序遍历目录中的每个文件来采集数据, 使用cut选项中结果中提取两位小时数, 并且清除hour域中的前导0.

## 35.5 数组操作 ##

### 35.5.1 输出数组的所有内容 ###

可以使用下标 * 和 @ 来访问数组中的每个元素, 对于定位参数来说, @ 更为有用.

```
[me@linuxbox ~]$ animals=("a dog" "a cat" "a fish")
[me@linuxbox ~]$ for i in ${animals[*]}; do echo $i; done
a
dog
a
cat
a
fish
[me@linuxbox ~]$ for i in ${animals[@]}; do echo $i; done
a
dog
a
cat
a
fish
[me@linuxbox ~]$ for i in "${animals[*]}"; do echo $i; done
a dog a cat a fish
[me@linuxbox ~]$ for i in "${animals[@]}"; do echo $i; done
a dog
a cat
a fish
```

对符号 ${animals[*]} 和 ${animals[@]} 加以引用会得到不同的结果, 符号 * 将数组的所有内容放在一个字符串中, 符哈 @ 使用3个字符串来显示数组的真实内容.

### 35.5.2 确定数组元素的数目 ###

使用参数扩展, 可以采用类似获取字符串长度的方式来确定数组中元素的数目.

```
[me@linuxbox ~]$ a[100]=foo
[me@linuxbox ~]$ echo ${#a[@]} # number of array elements
1
[me@linuxbox ~]$ echo ${#a[100]} # length of element 100
3
```

需要注意的是, 在bash中数组中未初始化的元素不统计到长度中.

### 35.5.3 查找数组中使用的下标 ###

可以通过参数扩展来实现:

```
${!array[*]}
${!array[@]}
```

引用中含有 @ 的形式更为有用, 因为它将数组的内容扩展成独立的单词.

```
[me@linuxbox ~]$ foo=([2]=a [4]=b [6]=c)
[me@linuxbox ~]$ for i in "${foo[@]}"; do echo $i; done
a
b
c
[me@linuxbox ~]$ for i in "${!foo[@]}"; do echo $i; done
2
4
6
```

### 35.5.4 在数组的结尾增加元素 ###

通过使用 += 运算符可以在数组的尾部自动添加元素.

```
[me@linuxbox~]$ foo=(a b c)
[me@linuxbox~]$ echo ${foo[@]}
a b c
[me@linuxbox~]$ foo+=(d e f)
[me@linuxbox~]$ echo ${foo[@]}
a b c d e f
```

### 35.5.5 数组排序操作 ###

shell没有直接的方式来实现数组排序, 但是可以使用一些代码来实现:

```
#!/bin/bash
# array-sort : Sort an array
a=(f e d c b a)
echo "Original array: ${a[@]}"
a_sorted=($(for i in "${a[@]}"; do echo $i; done | sort))
echo "Sorted array: ${a_sorted[@]}"
```

### 35.5.6 数组的删除 ###

可以使用unset命令来删除数组, 例如:

```
[me@linuxbox ~]$ foo=(a b c d e f)
[me@linuxbox ~]$ echo ${foo[@]}
a b c d e f
[me@linuxbox ~]$ unset foo
[me@linuxbox ~]$ echo ${foo[@]}
[me@linuxbox ~]$
```

也可以使用unset来删除单个的数组元素:

```
[me@linuxbox~]$ foo=(a b c d e f)
[me@linuxbox~]$ echo ${foo[@]}
a b c d e f
[me@linuxbox~]$ unset 'foo[2]'
[me@linuxbox~]$ echo ${foo[@]}
a b d e f
```

对数组赋值一个空值并不能清空数组的内容, 而且任何不含下标的数组变量的引用指的是数组中的元素0.

```
[me@linuxbox ~]$ foo=(a b c d e f)
[me@linuxbox ~]$ foo=
[me@linuxbox ~]$ echo ${foo[@]}
b c d e f
[me@linuxbox~]$ foo=(a b c d e f)
[me@linuxbox~]$ echo ${foo[@]}
a b c d e f
[me@linuxbox~]$ foo=A
[me@linuxbox~]$ echo ${foo[@]}
A b c d e f
```

## 35.6 关联数组 ##

关联数组使用字符串而不是整数作为数组索引, 这种功能给出了一种有趣的新方法来管理数据. 例如我们可以创建一个叫做 colors 的数组, 并用颜色名字作为索引:

```
declare -A colors
colors["red"]="#ff0000"
colors["green"]="#00ff00"
colors["blue"]="#0000ff"
```

不同于整数索引的数组仅仅引用它们就能创建数组, 关联数组必须用带有 -A 选项的 declare 命令创建.

访问关联数组元素的方式几乎与整数索引数组相同:

```
echo ${colors["blue"]}
```

## 35.7 本章结尾语 ##

数组和循环有一种天然的姻亲关系, 它们经常被一起使用. 例如:

```
for ((expr; expr; expr))
```

形式的循环尤其适合计算数组下标.

## 35.8 扩展阅读 ##

- Wikipedia 上面有两篇关于在本章提到的数据结构的文章:

[http://en.wikipedia.org/wiki/Scalar_(computing)](http://en.wikipedia.org/wiki/Scalar_(computing))

[http://en.wikipedia.org/wiki/Associative_array](http://en.wikipedia.org/wiki/Associative_array)
