# 第二十章: 文本处理 #

本章介绍的命令如下:

- cat: 连接文件并打印到标准输出
- sort: 对文本行排序
- uniq: 报告并省略重复行
- cut: 从每一行中移除文本区域
- paste: 合并文件文本行
- join: 基于某个共享字段来联合两个文件的文本行
- comm: 逐行比较两个已经排好序的文件
- diff: 逐行比较文件
- patch: 对原文件打补丁
- tr: 转换或删除字符
- sed: 用于过滤和转换文本的流编辑器
- aspel: 交互式拼写检查器

## 20.1 文本应用程序 ##

### 20.1.1 文件 ###

可以用文本格式编辑较大的文档, 一种常用的方法是首先在文本编辑器中编辑大型文档的内容, 然后使用标记语言描述文件格式.

### 20.1.2 网页 ###

网页属于文本文档, 一般使用 html 或 xml等标记语言描述内容的可视化形式.

### 20.1.3 电子邮件 ###

电子邮件本质上是一种基于文本的媒介, 即便是非文本附件, 在传输的时候也会被转成文本格式.

### 20.1.4 打印机输出 ###

在类UNIX系统中, 准备向打印机传送的信息是以纯文本格式传送的.

### 20.1.5 程序源代码 ###

程序员实际所编写的源代码, 也总是使用文本的形式编辑.

## 20.2 温故以求新 ##

### 20.2.1 cat-进行文件之间的拼接并且输出到标准输出 ###

cat命令的 -A 选项用于显示文本中的非打印字符.

```
cat -A foo.txt
```
在输出结果中, 文本中的制表符Tab由符号 [\^I] 表示, 这是一种常用的表示方法, 意味着 Ctrl+I.

cat命令的 -n 选项可以输出行编号, -s 选项可以进制输出多个空白行.

### 20.2.2 sort-对文本行进行排序 ###

sort是一个排序命令, 它的操作对象为标准输入或是命令行中指定的一个或多个文件后将结果送至标准输出.

```
sort > foo.txt
c
b
a
cat foo.txt
```

sort命令允许多个文件作为其输入参数, 所以可以将多个文件融合为一个已经排序的整体文件.

```
sort file1.txt file2.txt file3.txt > final_sorted_list.txt
```

sort的一些选项如下表:

| 选项 | 全局选项表示 | 描述 |
|:--|:--|:--|
| -b | --ignore-leading-blanks | 默认情况下, 整个行都会进行排序操作. 添加该选项后sort会忽略行头的空格, 从第一个非空白字符开始排序 |
| -f | --ignore-case | 排序时不区分大小写 |
| -n | --numeric-sort | 基于字符串的长度进行排序, 该选项使得文件数值顺序而不是按照字母表顺序排序 |
| -r | --reverse | 逆序排序, 结果为降序而不是升序 |
| -k | --key=field1[,field2] | 对filed1和field2之间的字符排序, 而不是整个文本行 |
| -m | --merge | 将每个输入参数当作已排序的文件名, 将多个文件合并为一个排好序的文件, 而不执行额外的排序操作 |
| -o | --output=file | 将排序结果输出到文件而不是标准输出 |
| -t | --field-separator=char | 定义字段分隔符, 默认情况下为空格或制表符 |

例如将du命令输出的结果进行排序, 以确定最大的硬盘空间:

```
du -s /usr/share/* | sort -nr | head
```

上面的命令能够输出正确结果的原因是数值出现在每一行的开头, 如果需要对文本行中的某个数值排序, 例如对ls的输出文件大小进行排序:

```
ls -l /usr/bin | sort -nr -k 5 | head
```

sort的许多用法都与表格数据处理有关. sort的k选项有许多特性, 默认情况下sort将空白字符用作字段之间的界定符, 并且排序时这些界定符是包括在字段中的.

例如对文件distros.txt的以下内容进行排序操作:

```
SUSE 10.2 12/07/2006
# 更多的内容
```

对该文件内容进行排序, 则可能会出现版本10位于第一行而版本9却在后面的情况, 因为这是按照字符排序可能会出现的结果.

为了解决这个问题, 需要对多个键值进行排序. sort支持 -k 选项的多个实例, 一个键值可能是一个字段范围, 如果没有指定则该键值始于指定的字段并到行尾.

```
sort --key=1,1 --key=2n distros.txt
# 等价与 sort -k 1 -k 2n distros.txt
```

第一个k选项指定了使用第一个字段排序, 第二个k选项指定了2n, 即第二个字段是排序的键值, 并且按照数值排序. 一个选项字母可能包含在一个键值说明符的结尾, 用来指定排序的种类.

以上的第三个字段包含的日期形式不利于排序, 但是sort提供了允许在字段中指定偏移的功能, 所以可以使用如下命令排序:

```
sort -k 3.7nbr -k 3.1nbr -k 3.4nbr distros.txt
```

其中指定 -k 3.7 表示从第三个字段的第7个字符开始排序, 同时添加 b 选项用来删除日期字段开头的空格, 以避免不同长度的空白影响排序结果.

有些文件不使用空白作为字段界定符, 例如 /etc/passwd 文件, 这时可使用 -t 选项指定界定符:

```
sort -t ':' -k 7 /etc/passwd | head
```

### 20.2.3 uniq-通知或省略重复行 ###

uniq命令的作用是, 给定一个已经排序好的文件, uniq会删除任何重复的行并将结果输出到标准输出中. 需要注意的是, GUN 版本的sort命令支持 -u 选项, 该选项可以移除sort输出内容中的重复行.

```
sort foo.txt | uniq
```

uniq常用的选项如下表:

| 选项 | 功能描述 |
|:--|:--|
| -c | 输出重复行列表, 并且在重复行前加上其出现次数 |
| -d | 只输出重复行, 不包括单独行 |
| -f n | 忽略每行前n个字段, 字段之间以空格分开, 与sort类似. uniq没有参数可以指定字段分隔符 |
| -i | 忽略大小写 |
| -s n | 忽略每行的前n个字符 |
| -u | 仅输出不重复行, 默认选项 |

## 20.3 切片和切块 ##

### 20.3.1 cut-删除文本行中的部分内容 ###

cut命令用于从文本行中提取一段文字并将其输出至标准输出, 可以接受多个文件和标准输入作为输入参数. 该命令的选择选项如下表:

| 选项 | 功能描述 |
|:--|:--|
| -c char_list | 从文本行中提取char_list定义的部分内容, 此列表可能会包含一个或更多冒号分开的数值范围 |
| -f field_list | 从文本行中提取field_list定义的一个或多个字段, 该列表可能会包含冒号分割的一个, 多个字段或字段范围 |
| -d delim_char | 指定 -f 选项后, 使用 delim_char 作为字段界定符, 默认为单个制表符 |
| --complement | 从文本中提取整行, 除了那些由 -c 或 -f 指定的部分 |

例如提取由制表符分割的 distros.txt 文件的第三个字段:

```
cut -f 3 distros.txt
# 提取年份字段
cut -f 3 distros.txt | cut -c 7-10
```

或者提取 /etc/passwd 文件的每行的第一个字段:

```
cut -d ':' -f 1 /etc/passwd | head
```

有时需要将文件处理为cut借助字符提取的理想的形式, 就需要将Tab符号替换为等长的空格符号, 可以借助GNU的coreutils软件包中的expand命令.

```
expand distros.txt | cut -c 23-
```

也可以使用unexpand命令进行反向替换.

### 20.3.2 paste-合并文本行 ###

paste命令是cut命令的逆向操作, 用于向文本文件中增加一个或更多的文本列, 该命令读取多个文件并将每个文件中提取出的字段组合为一个整体输出到标准输出流.

例如, 将distros.txt 文件按照发行版本的时间顺序排序:

```
# 首先按照时间排序
sort -k 3.7nbr -k 3.1nbr -k3.4nbr distros.txt > distros-by-date.txt
# 提取系统名称和发行版本号到新文件
cut -f 1,2 distros-by-date.txt > distros-versions.txt
# 提取发行时间到新文件
cut -f 3 distros-by-date.txt > distros-dates.txt
# 合并文件, 把发行时间置于开头
paste distros-dates.txt distros-versions.txt
```

### 20.3.3 join-连接两文件中具有相同字段的行 ###

join与paste命令类似, 也是向文件增加列, 是一个基于共享关键字字段将多个文件的数据拼接在一起的操作.

首先创建两个具有共享字段的文件作为演示, 利用 distros-by-date.txt 文件生成两个附属文件:

```
# 一个包含发行时间和系统名的文件
cut -f 1,2 distros-by-date.txt > distros-names.txt
paste distros-dates.txt distros-names.txt > distros-key-name.txt

# 一个包含发行时间和版本号的文件
cut -f 2,2 distros-by-date.txt > distros-vernums.txt
paste distros-dates.txt distros-vernums.txt > distros-key-vernums.txt
```

此时这两个文件具有公共字段发行时间, 然后使用join合并文件:

```
join distros-key-names.txt distros-key-vernums.txt | head
```

默认情况下, join使用空格作为输入字段的界定符, 以单个空格作为输出字段的界定符, 可以通过指定参数选项改变这一默认属性.

## 20.4 文本比较 ##

### 20.4.1 comm-逐行比较两个已排序文件 ###

comm命令用于文本文件之间的比较, 显示两文件中相异的行以及相同的行.

```
comm file1.txt file2.txt
```

comm会输出三列内容, 第一列显示的是第一个文件独有的行, 第二列显示的是第二个文件独有的行, 第三列显示的则是两个文件共有的行. comm命令还支持 -n 选项, 此处的n是1, 2或者3, 表示省略第几行的内容.

```
comm -12 file1.txt file2.txt
```

### 20.4.2 diff-逐行比较文件 ###

diff命令用于检测文件之间的不同, 支持多种输出形式, 并且具备一次性处理大文件集的能力. diff常见用法就是创建diff文件和补丁, 可以为诸如patch命令所用, 从而实现一个版本的文件更新为另一个版本.

```
diff file1.txt file2.txt

1d0
< a
4a4
> e
```

默认输出形式中, 每一组改动的前面都有一个以 "范围 执行操作 范围" 形式表示的改变操作命令, 该命令会告诉程序对第一个文件的某个位置进行某种改变, 便可实现与第二个文件内容一致.

diff 改变命令如下表:

| 改变操作 | 功能描述 |
|:--|:--|
| r1ar2 | 将第二个文件中的r2位置的行添加到第一个文件中的位置r1处 |
| r1cr2 | 用第二个文件r2处的行替代第一个文件的r1处的行 |
| r1dr2 | 删除第一个文件r1处的行, 并且删除的内容作为第二个文件r2行范围的内容 |

此格式中, 然为range一般是由冒号隔开的起始行和末尾行组成. 这种格式没有其他格式应用广泛, 上下文格式和统一格式才是比较普遍使用的格式.

上下文格式的输出结果如下:

```
diff -c file1.txt file2.txt

*** file1.txt	2017-04-16 19:54:54.135886088 +0800
--- file2.txt	2017-04-16 19:55:17.575885056 +0800
***************
*** 1,4 ****
- a
  b
  c
  d
--- 1,4 ----
  b
  c
  d
+ e
```

该结果以两个文件的名字和时间信息开头, 第一个文件用星号表示, 第二个文件用破折号表示, 输出结果的其余部分出现的星号和破折号则分别表示各自所代表的文件. 其他的内容便是两个文件间的差异组, 包括文本的默认行号.
第一组差异以范围开头, 表示第一个文件中的第1行到第4行; 第二组也以范围开头, 表示第二个文件中的第1行到第4行. 每个差异组的行都是以如下四个标识符之一开头, 标识符如下表:

| 标识符 | 含义 |
|:--|:--|
| (无) | 该行表示上下文文本共有的行 |
| - | 缺少的行, 该行只在第一个文件出现 |
| + | 多余的行, 该行只在第二个文件出现 |
| ! | 改变的行, 两个版本的内容都会显示 |

统一格式如下:

```
diff -u file1.txt file2.txt

--- file1.txt	2017-04-16 19:54:54.135886088 +0800
+++ file2.txt	2017-04-16 19:55:17.575885056 +0800
@@ -1,4 +1,4 @@
-a
 b
 c
 d
+e
```

统一格式中没有重复的文本行, 每一行的标识符如下:

| 标识符 | 含义 |
|:--|:--|
| (无) | 该行表示上下文文本共有的行 |
| - | 缺少的行, 该行只在第一个文件出现 |
| + | 多余的行, 该行只在第二个文件出现 |

### 20.4.3 patch-对原文件进行diff操作 ###

patch命令用于更新文本文件, 利用diff命令的输出结果将较旧版本的文件升级成较新版本.

```
diff -Naur file1.txt file2.txt > patchfile.txt
patch < patchfile.txt
```

## 20.5 非交互式文本编辑 ##

### 20.5.1 tr-替换或删除字符 ###

tr是替换字符命令, 可以看作一种基于字符的查找和替换操作.

```
echo "lowercase letters" | tr a-z A-Z
```

tr可以对标准输入进行操作并且将结果输出到标准输出, tr有两个参数: 等待转换的字符集和与之相对应的替换字符集. 字符集的表示方法可以是下面三种方式中的任意一种:

- 枚举列表: 例如 ABCDEFG
- 字符范围: 例如 A-Z, 这种方式可能也会受限与系统排序
- POSIX字符类: 例如 [:upper:]

多数情况下这两个字符集应该是同等长度, 第一个字符集也可能比第二个字符集长, 例如:

```
echo "lowercase letters" | tr [:lower:] A
```

除了替换, tr命令也可以删除字符, 例如删除每行末尾的回车符:

```
tr -d '\r' < dos_file > unix_file
```

tr命令还可以使用 -s 选项来删除重复出现(必须相邻)的字符:

```
echo "aaabbbccc" | tr -s ab
```

### 20.5.2 sed-用于文本过滤和转换的流编辑器 ###

sed是流式编辑器的缩写, 它可以对文本流, 指定文件集或标准输入进行文本编辑. sed的用法是, 首先给定sed某个简单的编辑命令或是包含多个命令的脚本文件名, 然后sed便对文本流的内容执行给定的编辑命令.

```
echo "front" | sed 's/front/back/'
```

sed命令总是以单个字母开头, sed默认紧跟在命令之后的字符作为分界符, 一般使用斜线.

```
echo "front" | sed 's_front_back_'
```

sed的多数命令允许在其前添加一个地址, 该地址用来指定输入流的哪一行被编辑, 默认对输入流的每一行执行该编辑命令.

```
echo "front" | sed '1s/front/back/'
```

常用的地址表达方式如下:

| 地址 | 功能说明 |
|:--|:--|
| N | n是正整数表示的行号 |
| $ | 最后一行 |
| /regexp/ | 用POSIX基本正则表达式描述的行, 此处用斜线作为分界符, 如果用 \cregexpc 形式即表示用字符c作为分界符 |
| addr1,addr2 | 行范围, 从addr1到addr2 |
| first~step | 从first行开始, 以step为间隔的所有行, 例如 1~2 表示所有的奇数行 |
| addr1,+n | addr1行及其之后的n行 |
| addr! | 除addr之外的所有行 |

一些地址表示实例如下:

```
# 输出1到5行
sed -n '1,5p' distros.txt

# 使用正则表达式
sed -n '/SUSE/p' distros.txt

# 否定形式
sed -n '/SUSE/!p' distros.txt
```

sed的常用编辑指令如下表:

| 命令 | 功能描述 |
|:--|:--|
| = | 输出当前行号 |
| a | 在当前行后附加文本 |
| d | 删除当前行 |
| i | 在当前行前输入文本 |
| p | 打印当前行, 默认sed会输出每一行且只编辑文件内部匹配地址的行, 当指定 -n 选项时默认操作会被覆盖 |
| q | 退出sed并不再处理其他行, 如果没有指定 -n 选项则会输出当前行 |
| Q | 直接退出sed不再处理行 |
| s/regexp/replacement/ | 将regexp的内容替换为 replacement |
| y/set1/set2 | 将字符集set1替换为字符集set2, 与tr类似, 但是sed命令要求字符集等长 |

例如将 distros.txt 文件中的 MM/DD/YYYY格式的时间形式修改为 YYYY-MM-DD形式:

```
sed 's/\([0-9]\{2\}\)\/\([0-9]\{2\}\)\/\([0-9]\{4\}\)$/\3-\1-\2/' distros.txt
```

其中使用了BRE中的回参考特性, 即如果replacement中出现了 \n 转义字符, 并且n为 1到9的数字, 那么此转义字符就是指前面正则表达式中与之对应的子表达式, 只要简单的将表达式置于小括号中即可称为子表达式.
另外还使用反斜杠来避免正则表达式中的斜杠让sed混淆, 而且因为sed默认情况下只接受基本正则表达式, 所以用反斜杠对某些元字符进行转义.

sed的s命令还可以在替换字符串后紧跟可选择标志符, 例如使用 g 标志符让sed对每行的所有匹配项进行替换操作, 默认只替换第一个匹配项.

sed命令还可以使用 -f 选项建立更复杂的命令脚本文件, 例如编写以下内容的脚本 distros.sed :

```
# sed script to produce Linux distributions report

1 i\
\
Linux Distributions Report\

s/\([0-9]\{2\}\)\/\([0-9]\{2\}\)\/\([0-9]\{4\}\)$/\3-\1-\2/
y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/
```

按照如下命令运行:

```
sed -f distros.sed distros.txt
```

该命令脚本的第3～6行是要插入文本的内容, 其中i命令后紧跟转义回车符(由反斜杠和回车符组成, 中间不能有空格), 可确保文本流中的嵌入回车符不会告诉编译器已经到了行末尾.

第8行实现小写字母到大写字母的转换, sed中的y命令不支持字符范围和POSIX字符类.

### 20.5.3 aspell-交互式拼写检查工具 ###

aspell继承自早期的ispell命令, 通常为那些需要拼写检查的程序所用. 它可以智能的检查不同类型文本文件的错误, 包括HTML文件, email消息等. 命令格式如下:

```
aspell check textfile
```

aspell在检查模式下是与用户交互的, 在有拼写错误时会有一个显示界面提示用户操作.

除非指定 --dont-backup选项, 否则aspell命令会创建一个包含原文本内容的备份文件, 文件名为原文件名加上 .bak 后缀.

aspell命令可以使用 -H 选项启动 HTML 模式.

## 20.6 本章结尾语 ##

## 20.7 附加项 ##

更多的命令:

| 命令 | 功能描述 |
|:--|:--|
| split | 将文件分成多个部分 |
| csplit | 基于上下文将文件分块 |
| sdiff | 左右并排显示文件差异并比较 |

GNU 项目网站包含了本章中所讨论工具的许多在线指南.

- 来自 Coreutils 软件包:

[http://www.gnu.org/software/coreutils/manual/coreutils.html#Output-of-entire-files](http://www.gnu.org/software/coreutils/manual/coreutils.html#Output-of-entire-files)

[http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-sorted-files](http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-sorted-files)

[http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-fields-within-a-line](http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-fields-within-a-line)

[http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-characters](http://www.gnu.org/software/coreutils/manual/coreutils.html#Operating-on-characters)

- 来自 Diffutils 软件包:

[http://www.gnu.org/software/diffutils/manual/html_mono/diff.html](http://www.gnu.org/software/diffutils/manual/html_mono/diff.html)

- sed 工具:

[http://www.gnu.org/software/sed/manual/sed.html](http://www.gnu.org/software/sed/manual/sed.html)

- aspell 工具:

[http://aspell.net/man-html/index.html](http://aspell.net/man-html/index.html)

- 尤其对于 sed 工具, 还有很多其它的在线资源:

[http://www.grymoire.com/Unix/Sed.html](http://www.grymoire.com/Unix/Sed.html)

[http://sed.sourceforge.net/sed1line.txt](http://sed.sourceforge.net/sed1line.txt)

试试用 google 搜索 “sed one liners”, “sed cheat sheets” 关键字.

