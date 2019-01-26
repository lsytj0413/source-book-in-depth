# 第二十一章: 格式化输出 #

本章介绍的命令如下:

- nl: 对行进行标号
- fold: 设定文本行长度
- fmt: 简单的文本格式化工具
- pr: 格式化打印文本
- printf: 格式化并打印数据
- grof: 文档格式化系统

## 21.1 简单的格式化工具 ##

### 21.1.1 nl-对行进行标号 ###

nl命令用于对行进行编号, 类似于 cat -n.

nl进行编号时支持一个叫做逻辑页的概念, 它可以重置数值序列. 逻辑页可以进一步分解为逻辑页标题, 正文和页脚. 如果nl的输入参数是多个文件, 那么nl会把它们当作一个文本流整体. 文本流中的每一个部分都由一些标记来区别, 如下表:

| 标记 | 含义 |
|:--|:--|
| \:\:\: | 逻辑页页眉开头 |
| \:\: | 逻辑页正文开头 |
| \: | 逻辑页页脚开头 |

每一个标记元素在一行中只允许出现一次, 每次处理完一个标记元素后, nl便将其从文本流中删除.

nl常用的选项如下表:

| 选项 | 含义 |
|:--|:--|
| -b style | 按照style格式对正文进行编号 |
| -f style | 按照style格式对页脚进行编号, 默认是不进行编号 |
| -h style | 按照style格式对标题进行编号, 默认是不进行编号 |
| -i number | 设置页编号的步进值为number, 默认值是1 |
| -n format | 设置编号格式 |
| -p | 在每个逻辑页的开始不再进行页编码重置 |
| -s string | 在每行行号后面增加string作为分隔符, 默认情况为一个tab制表符 |
| -v number | 设置每个逻辑页的第一个行号, 默认值是1 |
| -w width | 设置行号字段的宽度为width, 默认为6 |

编号style可用值如下表:

| style | 含义 |
|:--|:--|
| a | 对每行编号 |
| t | 仅仅对非空白行编号, 对正文是默认选项 |
| n | 不对任何行进行编号 |
| pregexp | 只对与基本正则表达式匹配的行进行编号 |

编号格式可用值如下表:

| 编号格式 | 含义 |
|:--|:--|
| ln | 左对齐, 无缩进 |
| rn | 右对齐, 无缩进, 默认选项 |
| rz | 右对齐, 有缩进 |

用户可以利用nl结合其他工具进行更复杂的任务, 首先利用文本编辑器编辑sed脚本, 并保存为 distros-nl.sed, 内容如下:

```
# sed script to produce Linux distributions report

1 i\
\\:\\:\\:
\
Linux Distributions Report\
\
Name         Ver.           Released\
----         ----           --------\
\\:\\:
s/\([0-9]\{2\}\)\/\([0-9]\{2\}\)\/\([0-9]\{4\}\)$/\3-\1-\2-/
$ a\
\\:\
\
End Of Report
```

该脚本文件完成了向原报告中插入nl逻辑页标记以及在报告末尾添加页脚内容的任务, 输入标记时需要使用双斜杠以避免sed将它们解释为转义字符.

然后结合sort, sed和nl创建一个报告文本:

```
sort -k 1,1 -k 2n distros.txt | sed -f distros-nl.sed | nl
```

### 21.1.2 fold-将文本中的行长度设定为指定长度 ###

fold是一个将文本行以指定长度分解的操作, 支持将一个或多个文本文件或是标准输入作为输入参数.

```
echo "The quick brown fox jumped over the lazy dog." | fold -w 12
```

在使用fold时如果没有指定宽度参数, 则默认为80, 而且fold在断行时并不会考虑单词边界, 可以添加 -s 选项使fold在到达width字符数前的最后一个有效空格处将原文本行断开.

### 21.1.3 fmt-简单的文本格式化工具 ###

fmt命令同样会折叠文本, 它即可以处理文件也可以处理标准输入, 并对文本流进行段落格式化, 它可以在保留空白行和缩进的同时对文本进行填充和连接.

```
fmt -w 50 fmt-info.txt | head
```

在fmt命令中, 默认情况下, 空白行, 单词之间的空格和缩进都保留在输出结果中; 不同缩进量的连续输入行并不进行拼接; 制表符会在输入中扩展并直接输出.

fmt命令常用的选项如下表:

| 选项 | 功能描述 |
|:--|:--|
| -c | 保留段落前两行的缩进, 随后的行都与第二行的缩进对齐 |
| -p string | 只格式化以前缀字符串string开头的行, 格式化后string的内容会作为每一个格式化行的前缀 |
| -s | 仅截断行模式, 在此模式下将会只根据指定的列宽截断行, 而短行不会与其他行结合 |
| -u | 字符间隔统一, 字符之间间隔一个空格字符, 句子之间间隔两个空格字符 |
| -w width | 默认值为75 |

通过 -p 选项, 我们可以选择性的格式化文件内容, 前提是要格式化的文本行都以相同的字符序列开头.

```
cat > fmt-code.txt
# This file contains code with comments.

# This line is a comment.
# Followed by another comment line.
# ANd another.

This, on the other hand, is a line of code.
And another line of code.
ANd another.

fmt -w 50 -p '# ' fmt-code.txt
```

### 21.1.4 pr-格式化打印文本 ###

pr命令用于给文本标页码, 在打印文本时, 通常希望将输出内容分成几页, 并且每页的顶部和底部都留出几行空白行, 这些空白行可以用于插入页眉和页脚.

```
pr -l 15 -w 65 distros.txt
```

在上面的例子中, 利用 -l 选项指定页长(每页行数), -w 选项指定页宽(每行字符数).

### 21.1.5 printf-格式化并打印数据 ###

printf命令并不适用于管道传输, 并且在命令行应用中也不常见. 最初是为C语言开发的, 后来的许多编程语言也都实现了这一功能, 包括shell环境, 事实上bash中内置了printf这一功能. 常见的用法如下:

```
printf "format" arguments
```

该命令参数包含一个格式说明的字符串, 然后将该格式应用于arguments所代表的输入内容.

```
printf "I formatted the string: %s\n" foo
```

常用的数据类型如下表:

| 指定符 | 说明 |
|:--|:--|
| D | 将一个数字格式化为有符号的十进制表示形式 |
| F | 格式化数字并以浮点数的格式输出 |
| O | 将一个整数格式化为八进制格式的整数 |
| s | 格式化字符串 |
| x | 将一个整数格式化为十六进制的数, 并且在使用字母是用小写字母 a~f 表示 |
| X | 类似x选项, 用大写字母表示十六进制 |
| % | 打印符号 % |

```
printf "%d, %f, %o, %s, %x, %X\n" 380 380 380 380 380 380
```

转换说明符也可以通过增加一些可选组件对输出效果进行调整, 一个完整的转换规格可能会包含以下内容:

```
$[flags][width][.precision]conversion_specification
```

常见的组成部分如下表:

| 组件 | 功能描述 |
|:--|:--|
| flags | 标识, 如下表 |
| width | 一个数字, 指定了最小字段宽度 |
| .precision | 对于浮点数, 便是指定小数点后的小数精确度, 对于字符串则是指定了输出字符的个数 |

flags的可选项如下表:

| 选项 | 功能描述 |
|:--|:--|
| # | 使用替代格式输出, 对于o类转换则输出结果以0开头, 对于x或X类转换则以0x和0X开头 |
| 0 | 用0填充输出 |
| - | 输出左对齐, 默认情况下为右对齐 |
| [空格] | 为正数产生一个前导空格 |
| + | 正数符号, 默认情况下只会输出负数的符号 |

一些格式化实例如下:

| 参数 | 格式 | 转换结果 | 说明 |
|:--|:--|:--|:--|
| 380 | "%d" | 380 | 输出整数 |
| 380 | "%#x" | 0x17c | 输出十六进制数 |
| 380 | "%05d" | 00380 | 输出至少为5个字符宽度的字段 |
| 380 | "%05.5f" | 380.00000 | 将数字格式化为精确到小数点后5位的浮点数 |
| 380 | "%010.5f" | 0380.00000 | 把最小字段宽度增加为10, 并且0填充可见 |
| 380 | "%+d" | +380 | 输出整数, 正数添加 + 号 |
| 380 | "%-d" | 380 | 输出整数, 左对齐 |
| abcdefghijk | "%5s" | abcdefghijk | 用最小的字段宽度来格式化字符串 |
| abcdefghijk | "%.5s" | abcde | 根据字符串的精确位数截断字符串 |

## 21.2 文档格式化系统 ##

### 21.2.1 roff和TEX家族 ###

当今主宰文档格式化领域的主要有两大家族: 即是从原始roff程序延伸而来的nroff和troff, 以及基于Donald Knuth的TEX排版系统.

### 21.2.2 groff-文档格式化系统 ###

groff其实是GNU实现方式的troff系列程序集, 它还包含一个用于模拟nroff及其他roff家族系列的程序功能的脚本.

man命令是由groff使用mandoc的宏包进行浏览的, 所以以下两条命令等价:

```
man ls | head
zcat /usr/share/man/man1/ls.1.gz | groff -mandoc -T ascii | head
```

groff可以输出多种不同格式的结果, 如果未指定输出格式, 那么PostScript便是其默认输出格式:

```
zcat /usr/share/man/man1/ls.1.gz | groff -mandoc | head

%!PS-Adobe-3.0
%%Creator: groff version 1.22.3
%%CreationDate: Thu Apr 27 12:38:03 2017
%%DocumentNeededResources: font Times-Roman
%%+ font Times-Bold
%%+ font Times-Italic
%%DocumentSuppliedResources: procset grops 1.22 3
%%Pages: 4
%%PageOrder: Ascend
%%DocumentMedia: Default 595 842 0 () ()
```

PostScript是一种页面描述语言, 一般用于描述送至类排字机设备打印的页面内容, 可以提取groff命令的输出结果并保存到文件中, 然后启动页面浏览器查看.

```
zcat /usr/share/man/man1/ls.1.gz | groff -mandoc > ~/桌面/foo.ps
# 转换为PDF
ps2pdf ~/桌面/foo.ps ~/桌面/ls.pdf
```

ps2pdf程序是ghostscript软件包的一个小成员, 该程序在多数支持打印的Linux系统上都有安装.

Linux系统通常包含许多进行文件格式转换的命令行程序, 命名方式通常为format2format, 可以使用以下命令输出程序名列表:

```
ls /usr/bin/*[[:alpha:]]2[[:alpha:]]*
```

首先使用表格格式化工具tbl对distros.txt文件中的Linux发行版本列表进行排版, 先修改sed脚本 distros-tbl.sed如下:

```
# sed script to produce Linux distributions report

1 i\
.TS\
center box;\
cb s s\
l n c.\
Linux Distributions Report\
=\
Name Version Released\
_
s/\([0-9]\{2\}\)\/\([0-9]\{2\}\)\/\([0-9]\{4\}\)$/\3-\1-\2-/
$ a\
.TE
```

然后查看排版输出:

```
sort -k 1,1 -k 2n distros.txt | sed -f distros-tbl.sed | groff -t -T ascii 2> /dev/null
```

在groff命令中使用 -t选项表示先用tbl预处理文本流.

### 21.3 本章结尾语 ###

简单的格式化工具, 例如fmt和pr在一些生成短文档的脚本中用途很大, 而像groff这类复杂工具则可以用来排版书籍.
