# 第十四章: 软件包管理 #

软件包管理是一种在系统上安装, 维护软件的方法. 本章主要介绍一些用于Linux软件包管理的命令行工具.

## 14.1 软件包系统 ##

不同的Linux发行版用的是不同的软件包系统, 一般分为两种软件包技术: Debian的.deb技术和Red Hat的.rpm技术.

主流软件包系统类如下表:

| 软件包系统 | 发行版本(部分) |
|:--|:--|
| Debian类(.deb技术) | Debian, Ubuntu, Xandros, Linspire |
| Red Hat类(.rpm技术) | Fedora, CentOS, Red Hat Enterprise Linux, openSUSE, Mandriva, PCLinuxOS |

## 14.2 软件包系统工作方式 ##

### 14.2.1 软件包文件 ###

包文件是组成软件包系统的基本软件单元, 是由组成软件包的文件压缩而成的文件集. 一个包可能包含大量的额程序以及支持这些程序的数据文件, 包文件即包含了安装文件, 还包含
了有关软件包自身及其内容的文本说明之类的软件包元数据. 此外还可能包含安装软件包前后执行配置任务的安装脚本.

### 14.2.2 库 ###

Linux用户可以从其所使用的Linux版本的中心库中获得软件包, 中心库一般包含了成千上万各软件包, 而且每一个都是专门为该发行版本建立和维护的.

### 14.2.3 依赖关系 ###

现代软件包管理系统都提供依赖性解决策略, 从而确保用户安装了软件包的同时也安装了其所有的依赖关系.

### 14.2.4 高级和低级软件包工具 ###

软件包管理系统通常包含两类工具: 执行如安装, 删除软件包文件等任务的低级工具和进行元数据搜索及提供依赖性解决的高级工具.

| 发行版本 | 低级工具 | 高级工具 |
|:--|:--|:--|
| Debian类 | dpkg | apt-get, aptitude |
| Fedora, Red Hat Enterprise Linux, CentOS | rpm | yum |

## 14.3 常见关键包管理任务 ##

注意: 在之后的描述中pagkage\_name指软件包的实际名称, package\_file指包含该软件包的文件名.

### 14.3.1 在库里面查找软件包 ###

包搜索命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | apt-get udpate; apt-cache search search_string |
| Red Hat类 | yum search search_string |

### 14.3.2 安装库中的软件包 ###

软件包安装命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | apt-get update; apt-get install package_name |
| Red Hat类 | yum install package_name |

### 14.3.3 安装软件包文件中的软件包 ###

低级软件包安装命令(不安装依赖性关系):

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | dpkg --install package_file |
| Red Hat类 | rpm -i package_file |

### 14.3.4 删除软件包 ###

软件包移除命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | apt-get remove package_name |
| Red Hat类 | yum erase package_name |

### 14.3.5 更新库中的软件包 ###

软件包更新命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | apt-get update; apt-get upgrade |
| Red Hat类 | yum update |

### 14.3.6 更新软件包文件中的软件包 ###

低级软件包更新命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | dpkg --install package_file |
| Red Hat类 | rpm -U package_file |

### 14.3.7 列出已安装的软件包列表 ###

软件包列表命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | dpkg --list |
| Red Hat类 | rpm -qa |

### 14.3.8 判断软件包是否安装 ###

软件包状态命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | dpkg --status package_name |
| Red Hat类 | rpm -q package_name |

### 14.3.9 显示已安装软件包的相关信息 ###

软件包信息查看命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | apt-cache show package_name |
| Red Hat类 | yum info package_name |

### 14.3.10 查看某具体文件由哪个软件包安装得到 ###

查询文件所属命令:

| 系统类型 | 命令 |
|:--|:--|
| Debian类 | dpkg --search file_name |
| Red Hat类 | rpm -qf file_name |

## 14.4 扩展阅读 ##

花些时间来了解你所用发行版中的软件包管理系统. 每个发行版都提供了关于自带软件包管理工具的文档. 另外, 这里有一些更普遍的资源:

- Debian GNU/Linux FAQ 关于软件包管理一章对软件包管理进行了概述:

[http://www.debian.org/doc/FAQ/ch-pkgtools.en.html](http://www.debian.org/doc/FAQ/ch-pkgtools.en.html)

- RPM 工程的主页:

[http://www.rpm.org](http://www.rpm.org)

- 杜克大学 YUM 工程的主页:

[http://linux.duke.edu/projects/yum/](http://linux.duke.edu/projects/yum/)

- 了解一点儿背景知识, Wikipedia 上有一篇关于 metadata 的文章:

[http://en.wikipedia.org/wiki/Metadata](http://en.wikipedia.org/wiki/Metadata)
