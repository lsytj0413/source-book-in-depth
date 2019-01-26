# 第十八章: 归档和备份 #

本章介绍的命令如下:

- gzip: 压缩和解压缩文件工具
- bzip2: 块排序文件压缩工具
- tar: 磁带归档工具
- zip: 打包和压缩文件
- rsync: 远程文件和目录的同步

## 18.1 文件压缩 ##

压缩算法一般分为两大类: 无损压缩和有损压缩. 在下面的讨论中仅仅设计无损压缩.

### 18.1.1 gzip-文件压缩与解压缩 ###

gzip命令用于压缩一个或多个文件, 执行命令后原文件会被压缩文件取代, 执行gunzip命令则可以将压缩文件还原为原文件.

```
ls -l /etc > foo.txt
# 压缩, 生成foo.txt.gz文件, foo.txt被删除
gzip foo.txt
# 解压缩foo.txt.gz文件
gunzip foo.txt
```

gzip常用的选项如下表:

| 选项 | 功能描述 |
|:--|:--|
| -c | 将输出内容写道标准输出并且保持原文件, 同 --stdout或 --to-stdout选项 |
| -d | 解压缩, 加上此命令后则等同 gunzip命令, 同 --decompress或 --uncompress 选项 |
| -f | 强制压缩, 即使压缩版本已经存在, 同 --force 选项 |
| -h | 显示帮助信息, 同 --help 选项 |
| -l | 列出所有压缩文件的压缩统计, 同 --list 选项 |
| -r | 如果操作参数有一个或多个目录, 则递归压缩包含在目录中的文件, 同 --recursive 选项 |
| -t | 检验压缩文件的完整性, 同 --test 选项 |
| -v | 在压缩时显示详细信息, 同 --verbose 选项 |
| -number | 设定压缩级别, number为 1(速度最快, 压缩比最小) 到9(速度最慢, 压缩比最大), 其中1等于 --fast, 9等于 --best, 默认级别为6 |

```
gzip foo.txt
gzip -tv foo.txt.gz
gzip -d foo.txt.gz

ls -l /etc | gzip > foo.txt.gz
# gunzip默认解压缩后最为 .gz 的文件
gunzip foo.txt
# 只查看压缩文本文件的内容
gunzip -c foo.txt | less
```

可以利用zcat命令联合gzip, 效果等同于带有 -c 选项的gunzip. zcat 的功能与cat命令相同, 只是它的操作对象是压缩文件.

```
zcat foo.txt.gz | less
```

同样也有zless命令, 与less管道功能相同.

### 18.1.2 bzip2-牺牲速度以换取高质量的数据压缩 ###

bzip2与gzip命令功能相仿, 但使用不同的压缩算法, 该算法具有高质量的数据压缩能力, 但却降低了压缩速度. 多数情况下用法与gzip相似, 只是用bzip2压缩后的文件以 .bz2 为后缀.

```
ls -l /etc > foo.txt
ls -l foo.txt
bzip2 foo.txt
ls -l foo.txt.bz2
bunzip2 foo.txt.bz2
```
前面讨论的gzip的所有选项(除 -r选项), bzip2都支持. 两者的压缩级别选项有些许不同, 同时解压缩文件的专用工具是 bunzip2 额 bzcat 命令.

bzip2还有 bzip2recover 命令, 用于恢复损坏的 .bz2 文件.

## 18.2 文件归档 ##

归档是与压缩操作配合使用的一个常用文件管理任务, 归档是一个聚集众多文件并将它们组合为一个大文件的过程.

### 18.2.1 tar-磁带归档工具 ###

tar命令是类UNIX系统中用于归档文件的经典工具, 是 tap archive的缩写. 经常看到的以 .tar 和 .tgz 结尾的文件, 分别是用普通的tar命令归档的文件和用gzip归档的文件. tar 归档可以由多个独立的文件, 一个或多个目录层次或者前两者的混合组合而成, 用法如下:

```
tar mode[options] pathname ...
```

常用的操作模式如下表:

| 模式 | 描述 |
|:--|:--|
| c | 创建文件和/或目录列表的归档文件 |
| x | 从归档中提取文件 |
| t | 在归档文件末尾追加指定路径名 |
| r | 列出归档文件的内容 |

```
# 创建归档文件
tar cf playground.tar playground
# 查看备份文件
tar tvf playground.tar
# 提取归档文件
mkdir foo & cd foo
tar xf ../playground.tar
```
除非是以超级用户的身份执行该命令, 不然从归档文件中提取出来的文件和目录的所有权属于执行归档操作的用户而不是文件的原始作者.

tar命令默认的路径名是相对路径而不是绝对路径, tar命令创建归档文件时会简单的通过移除路径名前面的斜杠来实现相对路径.

```
tar cf playground2.tar ~/playground
# 提取归档文件
cd foo
tar xf ../playground2.tar
```
此时的提取归档时是在foo目录下创建了一个 home 目录.

例如备份主目录并复制到另一个系统:

```
tar cf /media/BigDisk/home.tar /home
cd /
tar xf /media/BigDisk/home.tar
```

从归档中提取文件时, 可以限制只提取某些文件, 格式如下:

```
tar xf archive.tar pathname
```
在命令后面添加要提取的文件的路径名, 可以确保只恢复指定的文件, 而且可以指定多个路径名. 指定的路径名必须是存储在归档文件中的完整, 准确的相对路径名.
在指定路径名时通常不支持通配符, 但是GNU版本的tar命令(在Linux发行版中该版本最常见)通过使用 --wildcards选项而支持通配符.

```
cd foo
tar xf ../playground2.tar --wildcards 'home/me/playground/dir-*/file-A'
```
创建归档时通常辅助以find命令:

```
find playground -name 'file-A' -exec tar rf playground.tar '{}' '+'
find playground -name 'file-A' | tar cf - --files-from=- | gzip > playground.tgz
```
在tar命令中, 如果文件名前面明确指定有连字符[-], 则表示是标准输入输出的文件(这是一个惯例, 其他许多命令也采用连字符表示标准输入输出), --files-from选项(同 -T ) 则指定了tar命令从文件中而不是命令行中读取文件路径名列表. tgz是经gzip压缩的tar归档文件的后缀, 有时也用 .tar.gz 为后缀.

现代GNU版本的tar命令提供 gzip+z和 bzip2+j 选项创建压缩归档文件.

```
find playground -name 'file-A' | tar czf playground.tgz -T -
find playground -name 'file-A' | tar cjf playground.tbz -T -
```

可以利用tar命令在系统之间传输网络文件:

```
ssh remote-sys 'tar cf - Documents' | tar xf -
```

### 18.2.2 zip-打包压缩工具 ###

zip程序既是文件压缩工具也是文件归档工具. Linux用户主要使用zip程序与Windows系统交换文件, 而不是将其用于压缩或是归档文件. 命令的格式如下:

```
zip options zipfile file...
```

例如:

```
zip -r playground.zip playground
```
使用 -r 选项让zip命令递归压缩, 否则只会保留playground目录而不包括目录中的内容. zip命令会自动添加 .zip 后缀.

zip使用两种存储方式向归档文件中添加文件, 一是不对文件进行压缩直接存储, 而是压缩后存储.

可以使用unzip命令提取内容:

```
cd foo
unzip ../playground.zip
unzip ../playground.zip playground/dir-087/file-Z
```

如果指定的归档文件已经存在, 那么zip命令会更新而不是取代该文件. 这表示原来存在的归档文件会保留, 只是增加一些新文件, 原有匹配文件则被替换.

使用 -l 选项, unzip只会列出归档文件的内容而不会提取文件, 可以使用 -v 选项得到更详细的列表.

zip 命令也可以利用标准输入输出, 可以用 -@ 选项将多个文件送至zip进行压缩.

```
find playground -name 'file-A' | zip -@ file-A.zip
ls -l /etc | zip ls-etc.zip -
```

unzip命令不支持标准输入输出, 当指定 -p 选项后, unzip命令便会将输出结果以标准形式输出:

```
unzip -p ls-etc.zip | less
```

## 18.3 同步文件和目录 ##

将一个或多个目录与本地系统或是远程系统上的其他目录进行同步, 是维护系统备份文件的常用方法.

### 18.3.1 rsync-远程文件, 目录的同步 ###

针对类UNIX系统, 完成这一同步任务最合适的工具是 rsync命令. 该命令通过运用 rsync 远程更新协议同步本地系统与远程系统上的目录, 该协议允许 rsync 命令快速检测到本地和远程系统上的两个目录之间的不同,
从而以最少数量的复制动作完成两个目录之间的同步. 该命令的格式如下:

```
rsync options source destination
```

这里的source和destination是下列选项之一:

- 一个本地文件或目录
- 一个远程文件或目录, 形式为 [user@]host:path
- 一个远程rsync服务器, 由 rsync://[user@]host[:port]/path 指定

source和destinaion 之间必须有一个本地文件, 因为 rsync 不支持远程系统与远程系统之间的复制.

在本地系统进行同步的实例如下:

```
rm -rf foo/*
# -a选项: 递归归档并保留文件属性, -v选项: 详细输出
rsync -av playground foo
```

备份系统到外部硬盘的实例如下:

```
mkdir /media/BigDisk/backup
sudo rsync -av --delete /etc /home /usr/local /media/BigDisk/backup
```

### 18.3.2 在网络上使用rsync命令 ###

远程复制有两种方式.

方法之一是针对已安装了rsync命令以及诸如ssh等远程shell程序的系统. 假定本地网络有另一个具有足够可利用硬盘空间的系统, 同时希望利用远程系统而非外部设备进行备份操作,
假设远程系统已经有一个用于存放备份的 /backup 目录, 则命令如下:

```
sudo rsync -av --delete --rsh=ssh /etc /home /usr/local remote-sys:/backup
```


方法之二是使用rsync服务器同步网络文件, 通过配置rsync运行一个守护进程监听进来的同步请求.

```
rsync -av --delete rsync://rsync.gtlib.gatech.edu/fedora-linux-core/development/i386/os fedora-devel
```

## 18.4 扩展阅读 ##

- 在这里讨论的所有命令的手册文档都相当清楚明白, 并且包含了有用的例子. 另外, GNU 版本的 tar 命令有一个不错的在线文档. 可以在下面链接处找到:

[http://www.gnu.org/software/tar/manual/index.html](http://www.gnu.org/software/tar/manual/index.html)
