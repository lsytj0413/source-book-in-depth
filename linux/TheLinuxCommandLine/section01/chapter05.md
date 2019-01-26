# 第五章: 命令的使用 #

本章介绍的命令如下:

- type: 说明如何解释命令名
- which: 显示会执行哪些可执行程序
- man: 显示命令的手册页
- apropos: 显示一系列合适的命令
- info: 显示命令的info条目
- whatis: 显示一条命令的简述
- alias: 创建一条命令的别名

## 5.1 究竟什么是命令 ##

一条命令不外乎以下4中情况:

1. 可执行程序: 就像在/usr/bin目录里所看到的所有文件一样, 在该程序类别中程序可以编译为二进制文件, 可以是C++, Python等
2. shell内置命令: bash支持许多在内部称之为shell builtin的内置命令, 例如cd
3. shell函数: 合并到环境变量中的小型shell脚本
4. alias命令: 在其他命令的基础上自定义的自己的命令

## 5.2 识别命令 ##

Linux提供了两种方法来识别命令类型.

### 5.2.1 type-显示命令的类型 ###

type命令是一个shell内置命令, 可根据指定的命令名显示shell将要执行的命令类型.

### 5.2.2 which-显示可执行程序的位置 ###

有时还系统中可能安装了一个可执行程序的多个版本, which命令可以确定一个给定可执行文件的准确位置. which命令只适合可执行程序.

## 5.3 获得命令文档 ##

### 5.3.1 help-获得shell内置命令的帮助文档 ###

bash会每一个shell内置命令提供了一个内置的帮助工具, 输入help然后输入shell内置命令的名称即可使用该帮助工具.

### 5.3.2 help-显示命令的使用信息 ###

很多可执行程序都支持 --help选项, 该选项描述了命令支持的语法和选项.

### 5.3.3 man-显示程序的手册页 ###

大多数供命令行使用的可执行文件都提供了一个称之为manual或者是man page的正式文档, 该文档可以可以用一种称为man的特殊分页程序来查看.

```
man program
```

在大多数Linux系统中, man命令调用less命令来显示手册文档. man命令显示的手册文档被分为多个部分, 不仅包括用户命令, 也包括系统管理命令, 程序接口等, 如下表:

| 部分 | 内容 |
|:--:|:--|
| 1 | 用户命令 |
| 2 | 内核系统调用的程序接口 |
| 3 | C库函数程序接口 |
| 4 | 特殊文件, 如设备节点和驱动程序 |
| 5 | 文件格式 |
| 6 | 游戏和娱乐, 例如屏幕保护程序 |
| 7 | 其他杂项 |
| 8 | 系统管理命令 |

我们可以在man命令中指定具体的部分:

```
man section search_term
```

### 5.3.4 apropos-显示合适的命令 ###

我们可能会基于某个搜索条目进行搜索来确定使用的条目.

```
apropos floppy
```

带有 -k 选项的man命令与apropos 命令在功能上基本是一致的.

### 5.3.5 whatis-显示命令的简要描述 ###

whatis 程序显示匹配具体关键字的手册页的名字和一行描述.

### 5.3.6 info-显示程序的info条目 ###

GNU提供了info页面来代替手册文档, info页面可以通过info阅读器来显示, info页面使用超链接, 与网页结构相似.

info程序读取的info文件是树形结构, 分为各个单独的节点, 每个节点包含一个主题.

info页面中常用的命令如下:

| 命令 | 功能 |
|:--|:--|
| ? | 显示命令帮助 |
| PAGE UP 或 BACKSPACE | 返回上一页 |
| PAGE DOWN 或Spacebar | 翻到下一页 |
| n | 显示下一个节点 |
| p | 显示上一个节点 |
| u | 显示目前显示节点的父节点, 通常是一个菜单 |
| ENTER | 进入光标所指的超链接 |
| q | 退出 |

### 5.3.7 README和其他程序文档文件 ###

系统中安装的很多软件包都有自己的文档文件, 存放在 /usr/share/doc 目录中. 其中大部分文档文件是以纯文本格式存储的, 因此可以用less命令来查看; 有些是HTML格式, 可以用Web浏览器来查看; 有些是以 .gz扩展名结尾的文件, 这是以gzip压缩过的, 可以使用gzip包中的一个特殊的less版本, zless查看.

## 5.4 使用别名创建自己的命令 ##

可以使用 alias 命令创建新的命令别名.

```
alias foo='cd /usr; ls; cd -'
```

可以使用unalias来删除别名.

## 5.5 扩展阅读 ##

在网上有许多关于 Linux 和命令行的文档. 以下是一些最好的文档:

- Bash 参考手册是一本 bash shell 的参考指南. 它仍然是一本参考书, 但是包含了很多实例, 而且它比 bash 手册页容易阅读:

[http://www.gnu.org/software/bash/manual/bashref.html](http://www.gnu.org/software/bash/manual/bashref.html)

- Bash FAQ 包含关于 bash经常提到的问题的答案. 这个列表面向 bash 的中高级用户, 但它包含了许多有帮助的信息:

[http://mywiki.wooledge.org/BashFAQ](http://mywiki.wooledge.org/BashFAQ)

- GUN 项目为它的程序提供了大量的文档, 这些文档组成了 Linux 命令行实验的核心. 这里你可以看到一个完整的列表:

[http://www.gnu.org/manual/manual.html](http://www.gnu.org/manual/manual.html)

- Wikipedia 有一篇关于手册页的有趣文章:

[http://en.wikipedia.org/wiki/Man_page](http://en.wikipedia.org/wiki/Man_page)
