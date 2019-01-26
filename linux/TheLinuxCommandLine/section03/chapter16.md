# 第十六章: 网络 #

本章介绍的命令如下:

- ping: 向网络主机发送ICMP ECHO_REQUEST数据包
- traceroute: 显示数据包到网络主机的路由路径
- netstat: 显示网络连接, 路由表, 网络接口数据, 伪连接以及多点传送成员等信息
- ftp: 文件传输命令
- lftp: 改善后的文件传输命令
- wget: 非交互式网络下载器
- ssh: OpenSSH版的SSH客户端(远程系统登录命令)
- scp: secure copy的缩写, 远程复制文件命令
- sftp: secure file transfer program的缩写, 安全文件传输命令

## 16.1 检查, 检测网络 ##

### 16.1.1 ping-向网络主机发送特殊数据包 ###

最基本的网络连接命令就是ping命令, 该命令会向指定的网络主机发送特殊网络数据包ICMP ECHO_REQUEST, 多数网络设备收到该数据包后会做出回应, 通过这种方法即可验证网络连接是否正常.

```
ping linuxcommand.org
```

### 16.1.2 traceroute-跟踪网络数据包的传输路径 ###

traceroute(有些系统使用功能相似的tracepath程序代替)会显示文件通过网络从本地系统传输到指定主机过程中所有的停靠点的列表.

```
traceroute slashdot.org
```

### 16.1.3 netstat-检查网络设置及相关统计数据 ###

netstat命令可以用于查看不同的网络设置及数据, 通过使用其丰富的参数选项, 我们可以查看网络启动过程的许多特性.

```
# 使用 -ie 选项检查系统中的网络接口信息
netstat -ie
```
以上命令会输出网络接口的具体信息, 例如:

```
lo        Link encap:本地环回  
          inet 地址:127.0.0.1  掩码:255.0.0.0
          inet6 地址: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  跃点数:1
          接收数据包:23791 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:23791 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1 
          接收字节:96862180 (96.8 MB)  发送字节:96862180 (96.8 MB)

wlp2s0    Link encap:以太网  硬件地址 **
          inet 地址:192.168.0.104  广播:192.168.0.255  掩码:255.255.255.0
          inet6 地址: fe80::aa49:33e1:f8fc:5956/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  跃点数:1
          接收数据包:121418 错误:0 丢弃:0 过载:0 帧数:0
          发送数据包:88939 错误:0 丢弃:0 过载:0 载波:0
          碰撞:0 发送队列长度:1000 
          接收字节:165495789 (165.4 MB)  发送字节:11338201 (11.3 MB)
```
该系统有两个网络端口, 第一个称为lo, 是系统用来自己访问自己的回环虚拟接口; 第二个是wlp2s0, 是以太网端口.

对网络进行日常诊断, 是看能否在每个接口信息的第四行开头找到 **UP** 以及能否在第二行的 inet\_addr字段找到有效的IP地址.
第四行的 **UP** 代表着该网络接口已启用, 而对于使用动态主机配置协议的系统(DHCP), inet\_addr 字段里面有效的IP地址则说明了DHCP正在工作.

```
# 使用 -r 选项显示是内核的网络路由表
netstat -r

# 输出如下
内核 IP 路由表
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
default         sc.10086.com    0.0.0.0         UG        0 0          0 wlp2s0
192.168.0.0     *               255.255.255.0   U         0 0          0 wlp2s0
```
该表的第二行表示接收方的IP地址为192.168.0.0, IP以0结尾表示接收方是网络而非个人主机, Gateway参数字段表示的是建立当前主机与目标网络之间联系的网关的名称或IP地址, 此参数值是星号表示无需网关.

## 16.2 通过网络传输文件 ##

### 16.2.1 ftp-采用FTP传输文件 ###

ftp是不安全的, 因为它以明文的方式传送账户名以及密码, 所以几乎所有的FTP协议进行网络文件传输都是由匿名的FTP服务器处理, 匿名服务器允许任何人使用anonymous登录名以及无意义的密码登录.

```
ftp fileserver
# 以下进入ftp命令行
anonymous
[空密码]
cd pub/cd_images/Ubuntu-8.04
ls
lcd Desktop
get ubuntu-8.04-desktop-i386.iso
bye
```

每条命令的含义与解释如下:

| 命令 | 含义 |
|:--|:--|
| ftp fileserver | 启动ftp并与fileserver连接 |
| anonymous | 登录名 |
| [空密码] | 在登录名后输入, 可以输入空白, 否则尝试 user@example.com这样的格式 |
| cd pub/cd_images/Ubuntu-8.04 | 进入远程系统中的目录 |
| ls | 列出目录列表 |
| lcd Desktop | 切换到本地系统的 Desktop 目录 |
| get ubuntu-8.04-desktop-i386.iso | 下载文件到本地 |
| bye | 注销登录并退出ftp, 同quit或exit命令 |
| help | 显示命令列表 |

### 16.2.2 lftp-更好的ftp ###

lftp也是一个ftp客户端, 包括更多的功能, 例如多协议支持, 下载失败时自动重新尝试, 后台进程支持, Tab键完成文件名输入等.

### 16.2.3 wget-非交互式网络下载工具 ###

wget 是另一个用于文件下载的命令行程序, 即可以用于从网站上下载内容, 也可以从FTP站点下载, 单个文件, 多个文件甚至整个网站都可以被下载.

```
wget http://linuxcommand.org/index.php
```

## 16.3 与远程主机的安全通信 ##

在早期, 登录远程主机有两个命令--rlogin和telnet, 但是它们的通信都是以明文方式传递的, 所以现在几乎不再使用.

### 16.3.1 ssh-安全登录远程计算机 ###

SSH协议有两个优势:

- 该协议能验证远程主机的身份是否真实, 从而避免中间人攻击
- 该协议将本机和远程主机之间的通信内容全部加密

SSH协议包括两个部分: 一个是运行在远程主机上的SSH服务端, 用来监听端口22上可能过来的连接请求; 另一个是本地系统的SSH客户端, 用来与远程服务器进行通信.

多数Linux发行版都采用BSD项目的openSSH方法实现SSH, 有些发行版如Red Hat会默认安装客户端和服务端包, 有些如Ubuntu则只提供客户端包, 如需要服务端则可以安装OpenSSH-server包并配置运行.

ssh命令的使用方式如下:

```
# 连接到远程主机, 账户名使用本地账户名
ssh remote-sys

# 使用bob用户登录远程主机
ssh boob@remote-sys
```
如果ssh命令没有成功验证远程主机, 则会跳出相应的警告信息, 一般有两个原因会导致这个错误:

- 有攻击者正在尝试中间人攻击, 这种情况很少见
- 远程系统在某种程度上被改变, 例如重新安装系统或ssh服务端, 多见

如果是第二种情况, 只需要在客户端中移除相应主机过时的密钥即可:

```
# 警告信息中包含的话
Offending key in /home/me/.ssh/known_hosts: 1
```
只需删除该文件的第一行即可.

ssh命令除了能够开启远程系统上的shell会话外, 还能在远程系统上执行单个简单的命令.

```
# 远程主机上执行free命令并输出到本地
ssh remote-sys free
# 远程主机上执行ls并输出到本地文件, 使用引号包含命令以抑制命令行路径扩展
ssh remote-sys 'ls *' > dirlist.txt
# 远程主机上执行ls并输出到远程文件
ssh remote-sys 'ls * > dirlist.txt'
```

可以运行远程主机上的X Window并把图像化效果显示到本地:

```
# 对有些系统可能是 -Y选项
ssh -X remote-sys
xload
```

### 16.3.2 scp和sftp-安全传输文件 ###

OpenSSH软件包包含了里两个使用SSH加密隧道进行网络间文件复制的命令, 其中scp与cp命令相似, sftp是ftp的安全版本.

```
scp remote-sys:document.txt .
```

sftp命令的使用方式同ftp, 不过它并不需要远程主机运行ftp服务器, 只需要ssh服务器即可.

## 16.4 扩展阅读 ##

- Linux 文档项目提供了 Linux 网络管理指南, 可以广泛地(虽然过时了)了解网络管理方面的知识:

[http://tldp.org/LDP/nag2/index.html](http://tldp.org/LDP/nag2/index.html)

- Wikipedia 上包含了许多网络方面的优秀文章. 这里有一些基础的:

[http://en.wikipedia.org/wiki/Internet_protocol_address](http://en.wikipedia.org/wiki/Internet_protocol_address)

[http://en.wikipedia.org/wiki/Host_name](http://en.wikipedia.org/wiki/Host_name)

[http://en.wikipedia.org/wiki/Uniform_Resource_Identifier](http://en.wikipedia.org/wiki/Uniform_Resource_Identifier)
