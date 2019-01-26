# 第九章: 权限 #

本章介绍的命令如下:

- id: 显示用户身份标识
- chmod: 更改文件的模式
- umask: 设置文件的默认权限
- su: 以另一个用户的身份运行shell
- sudo: 以另一个用户的身份来执行命令
- chown: 更改文件所有者
- chgrp: 更改文件所属群组
- passwd: 更改用户密码

## 9.1 所有者, 组成员和其他所有用户 ##

在UNIX安全模型中, 一个用户可以拥有文件和目录. 当一个用户拥有一个文件或目录时, 它将对该文件或目录的访问权限拥有控制权. 用户又属于一个群组, 该群组由一个或多个用户组成, 组中用户对文件和目录的访问权限由其所有者赋予. 除了可以授予群组访问权限之外, 文件所有者也可以授予所有用户一些访问权限.

使用id命令可以获得用户身份标识的相关信息:

```
id
uid=1000(soren) gid=1000(soren) 组=1000(soren),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
```

在创建用户的时候, 用户将被分配一个用户ID, 用户ID与用户名一一映射. 同时用户将被分配一个有效组ID, 而且该用户也可以归属于其他的群组.

在Fedora系统中, 普通用户帐号是从500开始编号的, 在Ubuntu系统中是从1000开始的.

用户帐号定义在 /etc/passwd文件中, 用户组定义在 /etc/group文件中, 文件 /etc/shadow 中保存了用户的密码信息. 对于每一个用户账户, 文件 /etc/passwd 中都定义了对应用户的用户名, uid, gid, 账户的真实姓名, 主目录以及登录shell信息.

## 9.2 读取, 写入和执行 ##

对文件和目录的访问权限是按照读访问, 写访问以及执行访问来定义的.

常见的文件类型如下表:

| 属性 | 文件类型 |
|:--|:--|
| - | 普通文件 |
| d | 目录文件 |
| l | 符号链接, 文件属性始终为rwxrwxrwx, 这是个伪属性值, 指向的文件属性才是真的文件属性 |
| c | 字符设备文件, 表示以字节流形式处理数据的设备, 如终端 |
| b | 块设备文件, 表示以数据块方式处理数据的设备, 如光盘驱动 |

文件属性中剩下的9个字符称为文件模式, 分别表示文件所有者, 文件所属群组和其他用户对该文件的读取, 写入和执行权限.

权限属性如下表:

| 属性 | 文件 | 目录 |
|:--|:--|:--|
| r | 允许打开和读取 | 如果设置了执行权限, 则运行列出目录下的内容 |
| w | 允许写入或截断文件, 如果也设置了执行权限, 那么目录中的文件允许创建, 被删除以及被重命名 | 不允许重命名或删除文件, 该功能由目录权限决定 |
| x | 允许把文件当作程序一样来执行, 以脚本编写的文件必须被设置为可读 | 允许进入目录下 |

### 9.2.1 chmod-更改文件模式 ###

文件所有者和超级用户可以使用chmod命令修改文件的模式. chmod支持两种不同的改变文件模式的方法, 一是八进制数字表示法, 二是符号表示法.

1. 八进制数字表示法

八进制表示法指的是使用八进制数字来设置所期望的权限模式. 表示法如下表:

| 八进制 | 二进制 | 文件模式 |
|:--|:--|:--|
| 0 | 000 | --- |
| 1 | 001 | --x |
| 2 | 010 | -w- |
| 3 | 011 | -wx |
| 4 | 100 | r-- |
| 5 | 101 | r-x |
| 6 | 110 | rw- |
| 7 | 111 | rwx |

例如:

```
chmod 600 foo.txt
```

2. 符号表示法

chmod命令支持一种符号表示法来指定文件模式, 该符号表示法分为三个部分: 更改会影响谁, 要执行哪个操作以及要设置哪种权限.

对象表:

| 符号 | 含义 |
|:--|:--|
| u | user的简写, 表示文件或者目录的所有者 |
| g | 文件所属群组 |
| o | others的简写, 表示其他所有用户 |
| a | all的简写, 默认值 |

操作符 [+] 表示添加权限, [-] 表示删除权限, [=] 表示只有指定权限可用, 其他的权限都被删除. 权限由字符 [r], [w], [x] 指定.

### 9.2.2 采用GUI设置文件模式 ###

通过GUI的对话框也可以设置文件模式.

### 9.2.3 umask-设置默认权限 ###

umask命令设置创建文件时指定给文件的默认权限, 它使用八进制表示法表示从文件模式属性中删除一个位掩码.

```
# 查看当前位掩码
umask
```

除了常见的读写执行权限之外, 还有几个少用的权限设置.

1. setuid

八进制为 (4000), 当把它应用到一个可执行文件时, 有效用户id将从实际用户id设置成程序所有者的id.

2. setgid

八进制为 (2000), 把有效组id从用户的实际组id更改为文件所有者的组id. 如果对一个目录设置setgid位, 那么该目录下新创建的文件将由该目录所在组所有, 而不是文件创建者所在组所有. 当一个公共组下的成员需要访问共享目录下的所有文件时特别有效.

3. sticky

八进制为 (1000), 表示一个可执行文件是不可变换的. 在linux下会忽略文件的sticky位, 但是如果对一个目录设置sticky位, 那么将阻止用户删除或重命名文件, 除非用户是这个目录的所有者, 文件所有者或者是超级用户.

```
# setuid
chmod u+s file
# setgid
chmod g+s file
# sticky
chmod +t dir
```

## 9.3 更改身份 ##

有三种方式用来转换身份:

- 注销系统并以其他用户的身份重新登录系统
- 使用su命令
- 使用sudo命令

在shell会话状态下, 使用su命令将允许你假定为另一个用户的身份, 既可以用这个用户的ID来启动一个新的shell会话, 也可以以这个用户的身份来发布一个命令. 使用sudo命令将允许管理者创建一个称为 /etc/sudoer 的配置文件, 并且定义一些特定的命令, 这些命令只有被赋予为假定身份的特定用户才允许执行.

### 9.3.1 su-以其他用户的组ID的身份来运行shell ###

su命令用来以另一个用户的身份来启动shell, 该命令的一般形式为:

```
su [-[l]] [user]
```
如果包含 [-l] 选项, 那么得到的shell会话界面将是用户指定用户的登录界面, 这意味着该指定用户的运行环境将被加载, 而且工作目录也将更改为指定用户的主目录. 如果没有指定用户则默认为超级用户. [-l] 选项可以缩写为 [-].

也可以使用su命令执行单个命令而不开启新的交互式命令界面, 操作方式如下符:

```
su -c 'command'
```
使用这种方式, 单个命令行将被传递到一个新的shell环境下进行执行.

### 9.3.2 sudo-以另一个用户的身份执行命令 ###

管理者可以通过配置sudo命令, 使系统以一种可控的方式允许一个普通用户以一个不同的身份执行命令. 而且sudo命令并不需要输入超级用户的密码, 而是需要输入自己的密码来进行认证. sudo命令也并不启动一个新的shell环境, 也不需要加载另一个用户的运行环境.

### 9.3.3 chown-更改文件所有者和所属群组 ###

chown命令用来更改文件或目录的所有者和所属群组, 使用这个命令需要超级用户的权限. 命令格式如下:

```
chown [owner][:[group]] file ...
```

### 9.3.4 chgrp-更改文件所属群组 ###

使用方式和chown相似, 是更早版本的命令.

## 9.4 权限的使用 ##

实例: 创建一个共享目录, 由两个用户 bill 和 karen 共同存储音乐文件.

```
# 首先创建一个组 music, 将两个用户都加入该组
# bill 创建共享目录
sudo mkdir /usr/local/share/Music
# bill 更改目录所属组
sudo chown :music /usr/local/share/Music
sudo chmod 755 /usr/local/share/Music
# 设置目录下文件的有效组
sudo chmod g+s /usr/local/share/Music
umask 0002
```

## 9.5 更改用户密码 ##

命令格式如下:

```
passwd [user]
```

## 9.6 扩展阅读 ##

- Wikipedia 上面有一篇关于 malware(恶意软件)好文章:

[http://en.wikipedia.org/wiki/Malware](http://en.wikipedia.org/wiki/Malware)

还有一系列的命令行程序, 可以用来创建和维护用户和用户组. 更多信息查看以下命令的手册页:

- adduser
- useradd
- groupadd