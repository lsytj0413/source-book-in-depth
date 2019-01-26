# 第十七章: 文件搜索 #

本章介绍的命令如下:

- locate: 通过文件名查找文件
- find: 在文件系统目录框架中查找文件
- xargs: 从标准输入中建立, 执行命令行
- touch: 更改文件的日期时间
- stat: 显示文件或文件系统的状态

## 17.1 locate-较简单的方式查找文件 ##

locate命令通过快速搜索数据库, 以寻找路径名与给定字符串相匹配的文件, 同时输出所有匹配结果.

```
# 查找bin目录下的zip开头的文件
locate bin/zip
# 更复杂的搜索
locate zip | grep bin
```
locate命令有很多衍生版本, 其中slocate与mlocate是比较常用的, 它们通常由名为locate的符号链接访问.

locate搜索的数据库来自与另一个叫做updatedb的命令创建的, 通常该命令由cron任务定期执行, 多数系统每天执行一次, 所以locate可能会出现不能搜索到较新的文件的情况.
如果想要对较新的文件生效, 可以手动运行updatedb命令.

## 17.2 find-较复杂的方式查找文件 ##

find命令可以根据文件的各种属性在既定的目录及其子目录中查找.

```
# 列出家目录下的文件列表
find ~
# 统计文件数量
find ~ | wc -l
```
find命令可以用来搜索符合特定要求的文件, 通过综合应用test选项, action选项以及options选项来实现高级文件搜索.

### 17.2.1 test选项 ###

可以使用以下test选项来查找目录或普通文件:

```
# 目录
find ~ -type d | wc -l
# 普通文件
find ~ -type f | wc -l
```

常用的文件类型如下表:

| 文件类型 | 描述 |
|:--|:--|
| b | 块设备文件 |
| c | 字符设备文件 |
| d | 目录 |
| f | 普通文件 |
| l | 符号链接 |

还可以添加其他选项实现依据文件大小和文件名的搜索:

```
find ~ -type f -name "*.JPG" -size +1M | wc -l
```
使用-name选项表示查找文件的通配符格式, 此处使用双引号来避免shell的路径名扩展. 使用-size选项来表示文件大小的匹配, 加号表示比给定数值大, 减号表示比给定数值小, 没有符号则表示完全相等.

find支持的计量单位如下表:

| 字母 | 单位 |
|:--|:--|
| b | 512字节的块(默认值) |
| c | 字节 |
| w | 两个字节的字 |
| k | KB |
| M | MB |
| G | GB |

find命令支持的test选项参数如下表(加号和减号的规则适用于所有用到数值参数的情况):

| test参数 | 描述 |
|:--|:--|
| -cmin n | 匹配n分钟前改变状态(内容或属性)的文件或目录, -n表示不到n分钟, +n表示超过n分钟 |
| -cnewer file | 内容或属性修改时间比文件file更晚的文件或目录 |
| -ctime n | 匹配系统中n*24小时前文件状态(内容, 属性或访问权限)被改变的文件或目录 |
| -empty | 空文件或空目录 |
| -group name | 匹配属于name组的文件或目录, name可以是名称或组ID号 |
| -iname pattern | 与-name选项类似, 不区分大小写 |
| -inum n | 匹配索引节点是n的文件, 用于查找某个特定索引节点的所有硬件链接 |
| -mmin n | 匹配n分钟前改变内容的文件或目录 |
| -mtime n | 匹配24*n小时前只有内容被改变的文件或目录 |
| -name pattern | 匹配特定通配符模式的文件或目录 |
| -newer file | 内容修改时间比file更近的文件或目录, 文件备份时非常有用, 每次备份时更新某个文件(如日志), 然后用此参数确定上一次更新后哪个文件被改变 |
| -nouser | 匹配不属于有效用户的文件或目录, 可用来查找属于已删除账户的文件, 也可以用来检测攻击者的活动 |
| -nogroup | 匹配不属于有效组的文件或目录 |
| -perm mode | 寻找访问权限与既定模式匹配的文件或目录, 既定模式可以以八进制或符号的形式表示 |
| -samefile name | 与-inum 类似, 匹配与file文件用相同的inode号的文件 |
| -size n | 匹配n大小的文件 |
| -type c | 匹配c类型的文件 |
| -user name | 属于name用户的文件或目录, name可以是用户名或用户ID号 |

find命令的test选项可以结合逻辑操作从而建立具有复杂逻辑关系的匹配条件, 支持的逻辑操作符如下表:

| 操作符 | 功能描述 |
|:--|:--|
| -and | 与操作, 操作符两侧的条件同时满足. 同 -a选项, 默认逻辑关系 |
| -or | 或操作, 操作符两侧的条件满足一个, 同 -o选项 |
| -not | 非操作, 操作符后的条件为假, 同 -! 选项 |
| () | 括号操作, 进行优先级修改. 默认情况下find从左至右运算逻辑值. 括号在shell中有特殊意义, 在命令行中应该用引号包含, 或使用反斜杠转义 |

```
# 查找访问权限不是0600的文件和访问权限不是0700的子目录
find ~ \( -type f -not -perm 0600 \) -or \( -type d -not -perm 0700 \)
```

### 17.2.2 action选项 ###

find命令允许直接对搜索结果执行动作, 即action选项.

find命令预定义了一些动作, 如下表:

| 动作 | 功能描述 |
|:--|:--|
| -delete | 删除匹配文件 |
| -ls | 对匹配文件执行ls操作, 以标准格式输出其文件名以及所要求的其他信息 |
| -print | 输出匹配文件的全路径, 默认操作 |
| -quit | 一旦匹配成功则退出 |

```
# 删除.BAK文件
find ~ -type f -name "*.BAK" -delete
```
每个test选项和action选项之间的默认逻辑关系是与逻辑.

除了已有的预定义操作命令, 同样也可以任意调用用户想要执行的操作命令, 传统的办法是在命令行中使用 -exec 操作:

```
# 格式如下, command表示要执行的命令名, {}代表当前路径, 分号表示命令结束
# -exec command {};
# 例如实现 -delete 命令:
-exec rm '{}' ';'
```
可以使用 -ok 选项取代 -exec 选项, 这样每一次指定命令执行之前系统都会询问用户.

```
find ~ -type f -name 'foo*' -ok ls -l '{}' ';'
```

当使用 -exec选项时, 每次查找到匹配文件后都会调用执行一次指定命令, 但有时用户希望只调用一次命令就完成对所有匹配文件的操作. 实现这样的一次操作有两种方法:

- 使用外部命令 xargs
- 使用find本身自带的新特性

通过将命令行末尾的分号修改为加号, 便可将find命令的匹配结果作为指定命令的输入, 从而一次完成对所有文件的操作:

```
find ~ -type f -name 'foo*' -exec ls -l '{}' +
```

xargs命令处理标准输入信息并将其转变为某指定命令的输入参数列表:

```
find ~ -type f -name 'foo*' -print | xargs ls -l
```

注意: 存在命令行过长而使得shell编辑器无法处理的情况, 如果命令行中包含的输入参数太多而超过了系统支持的最大长度, xargs只会尽可能对最大数量的参数执行指定操作, 并不断重复这一过程直到所有标准输入全部处理完毕.
在xargs命令后面添加 --show-limits选项可输出命令行最大能承受的参数数量.

在类UNIX系统中允许文件名里面含有空格甚至换行符, 这会给像xargs这些为其他程序创建参数列表的命令带来一些问题, 因为内嵌的空格可能会被当作分隔符, 而要执行的操作可能会把空格隔开的单词当作不同的输入参数来处理.
为了解决这个问题, find和xargs命令允许使用空字符(ASCII码为0, 空格符为32)作为分隔符, find命令提供 -print0选项来产生以空字符作为各参数之间分隔符的输出结果, xargs命令提供 --null 选项支持接收空字符作为参数分隔符的输入.

```
find ~ -iname '*.jpg' -print0 | xargs --null ls -l
```

### 17.2.3 返回到playground文件夹 ###

本节使用一个find命令的实例来介绍find命令的应用.

首先, 创建一个包含很多目录和文件的文件夹:

```
# 创建100个目录
mkdir -p playground/dir-{00{1..9},0{10..99},100}
# 每个目录下创建一些文件
touch playground/dir-{00{1..9},0{10..99},100}/file-{A..Z}
```

查找名为 file-A 的文件:

```
find playground -type f -name 'file-A'
```
find命令不会产生排序的结果, 其输出顺序是由在存储设备中的布局决定的.

创建一个新文件:

```
touch playground/timestamp
# 使用stat命令输出系统掌握的文件所有信息及属性
stat playground/timestamp

# 再次touch文件查看时间更新
touch playground/timestamp
stat playground/timestamp
```

更新名为file-B的文件的时间:

```
find playground -type f -name 'file-B' -exec touch '{}' ';'
```

按时间查找文件(即file-B):

```
find playground -type f -newer playground/timestamp
```

查找具有不安全访问权限的文件, 并修改权限:

```
find playground \( -type f -not -perm 0600 \) -or \( -type d -not -perm 0700 \)
# 修改权限
find playground \( -type f -not -perm 0600 -exec chmod 0600 '{}' ';' \) -or \( -type d -not -perm 0700 -exec chmod 0700 '{}' ';' \)
```
注意: 一般来说将以上的命令拆分为两个, 分别针对文件和针对目录更为易读与理解.

### 17.2.4 option选项 ###

option选项用于控制find命令的搜索范围, 可能包含在其他测试选项或行为选项之中, 常用的选项如下表:

| 选项 | 描述 |
|:--|:--|
| -depth | 引导find程序优先处理目录内文件, 当指定 -delete 操作时, 该选项会自动调用 |
| -maxdepth levels | 当执行测试条件行为时, 设置find程序陷入目录数的最大级别数 |
| -mindepth levels | 在应用测试条件和行为之前, 设置find程序陷入目录数的最小级别数 |
| -mount | 引导find程序不去遍历挂载在其他文件系统上的目录 |
| -noleaf | 引导find不要基于正在搜索类UNIX文件系统的假设来优化搜索, 当扫描DOS/Windows文件系统和CD时会使用此选项 |

## 17.3 扩展阅读 ##

- 程序 locate, updatedb, find 和 xargs 都是 GNU 项目 findutils 软件包的一部分. 这个 GUN 项目提供了大量的在线文档, 这些文档相当出色, 如果你在高安全性的环境中使用这些程序, 你应该读读这些文档.

[http://www.gnu.org/software/findutils/](http://www.gnu.org/software/findutils/)
