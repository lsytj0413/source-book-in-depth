# 第十五章: 存储介质 #

本章介绍的命令如下:

- mount: 挂载文件系统
- unmount: 卸载文件系统
- fdisk: 硬盘分区命令
- fsck: 检查修复文件系统
- fdformat: 格式化软盘
- mkf: 创建文件系统
- dd: 向设备直接写入, 面向块数据
- genisoimage(mkisofs): 创建一个ISO9600映像文件
- wodim(cdrecord): 向光存储介质写入数据
- md5sum: 计算MD5校验码

## 15.1 挂载, 卸载存储设备 ##

将设备添加到文件系统树中, 从而允许操作系统可以操作该设备的过程称为挂载.

/etc/fstab 文件内容列出了系统启动时挂载的设备, 例如:

```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/nvme0n1p3 during installation
UUID=dca22cc7-f4ac-4132-a504-d1742096a228 /               ext4    errors=remount-ro 0       1
# /boot was on /dev/nvme0n1p2 during installation
UUID=40fac7d4-d032-41e3-a831-7b3c22fa4d23 /boot           ext4    defaults        0       2
# /boot/efi was on /dev/nvme0n1p1 during installation
UUID=62E0-6A7E  /boot/efi       vfat    umask=0077      0       1
# /home was on /dev/nvme0n1p5 during installation
UUID=02c7ab2f-3bfb-4571-8153-0cf798c18268 /home           ext4    defaults        0       2
# swap was on /dev/nvme0n1p4 during installation
UUID=902b5bdd-e2da-4ad5-b324-b483b989d68f none            swap    sw              0       0
```

该文件中每一行的内容包含6个字段, 每个字段的含义如下表:

| 字段 | 内容 | 描述 |
|:--|:--|:--|
| 1 | 设备 | 通常表示与物理设备相关的设备文件的真实名称, 比如/dev/hda1代表第一个IDE通道上的第一块分区, 也可以使用文本标签来关联设备. |
| 2 | 挂载节点 | 设备附加到文件系统树上的目录 |
| 3 | 文件系统类型 | 支持如 ext4, FAT16, NTFS等 |
| 4 | 选项 | 挂载时的选项参数, 例如只读等 |
| 5 | 频率 | 用于dump命令决定是否对该文件系统进行备份以及多久备份一次 |
| 6 | 优先级 | 用于fsck命令决定在启动时需要被扫描的文件系统的顺序 |

### 15.1.1 查看已挂载的文件系统列表 ###

mount命令用于文件系统挂载, 不带任何参数输入该命令会调出目前已经挂载的文件系统列表. 列表的格式是: device on mount\_point type filesystem\_type(options).

```
# 挂载光盘
mkdir /mnt/cdrom
mount -t iso9660 /dev/hdc /mnt/cdrom
cd /mnt/cdrom

# 卸载光盘
# 首先将工作目录修改为非 /mnt/cdrom目录, 否则会导致卸载失败
cd /mnt
umount /dev/hdc
```

### 15.1.2 确定设备名称 ###

在 /dev 目录下存在所有设备的相关信息, 命名的固定模式如下:

| 模式 | 设备 |
|:--|:--|
| /dev/fd* | 软盘驱动器 |
| /dev/hd* | 较旧系统上的IDE硬盘, 一般主板上有两个IDE通道, 并且每个通道有两个驱动器附着点, 第一个叫做主设备, 第二个叫做从设备. /dev/hda代表第一个通道上的主设备, /dev/hdb 代表第一个通道上的从设备, /dev/hda1代表该硬盘上的第一块分区, 以此类推 |
| /dev/lp* | 打印机设备 |
| /dev/sd* | SCSI硬盘, 在最近的Linux系统中, 内核把所有的类硬盘设备都作为SCSI硬盘, 命名规则与/dev/hd* 设备类似 |
| /dev/sr* | 光驱(CD/DV播放机和刻录机)

另外经常看到的像 /dev/cdrom, /dev/dvd, /dev/floppy 这样的符号链接, 它们都是指向实际的设备文件的.

可以用以下的方法来命名插入系统的可移动设备:

```
tailf /var/log/messages
# 插入可移动设备, 可能会看到如下的日志输出:
# sdb: sdb1
# [sdb] Attached SCSI removable disk
```
可以看到设备名是 /dev/sdb, /dev/sdb1是指向此设备的第一个分区.

## 15.2 创建新的文件系统 ##

将一个使用FAT32文件系统的闪存驱动器重新格式化为Linux本地文件系统, 需要两个步骤: 一是在对现有分区不满意的情况下创建一个新的分区布局, 二是在驱动器上创建一个新的空文件系统.

### 15.2.1 用fdisk命令进行磁盘分区 ###

fdisk命令实现用户与磁盘设备进行较低层次的直接交互, 可以用来编辑, 删除以及创建设备分区.

```
# 首先卸载设备
umount /dev/sdb1
# fdisk操作整个设备
fdisk /dev/sdb
```

fdisk命令会进入自有的程序菜单, 在该菜单中可以使用以下命令:

| 命令 | 作用 |
|:--|:--|
| m | 帮助信息 |
| p | 打印设备的分区表 |
| l | 输出包含所有可能分区类型的对照表 |
| t | 修改分区ID |
| w | 写入对设备的修改并退出 |
| q | 不保存改动并退出 |

### 15.2.2 用mkfs命令创建新的文件系统 ###

分区编辑已经完成, 我们便可以在闪存上创建新的文件系统. mkfs命令可以用来创建各种类型的文件系统.

```
# 创建ext3类型的文件系统
mkfs -t ext3 /dev/sdb1
# 格式化为FAT32
mkfs -t vfat /dev/sdb1
```

## 15.3 测试, 修复文件系统 ##

每次启动系统时, 挂载文件系统之前都会由fsck程序对文件系统的完整性进行检查. fstab文件的每个条目的末尾数字正是对应文件系统的检查优先级, 优先级为0表示设备不执行检查.

fsck还能修复损坏的文件系统, 修复程度取决于损坏程度. 对于类UNIX文件系统, 修复的文件会存放在文件系统根目录下的 lost+fount 目录中.

```
fsck /dev/sdb1
```

## 15.4 格式化软盘 ##

格式化软盘示例:

```
fdformat /dev/fd0
# 创建一个msdos文件系统
mkfs -t msdos /dev/fd0
```

## 15.5 直接从/向设备转移数据 ##

dd命令可以将数据块从一个地方复制到另一个地方, 格式如下:

```
dd if=input_file of=output_file [bs=block_size [count=blocks]]
```

例如, 将U盘/dev/sdb的数据复制到 /dev/sdc,

```
dd if=/dev/sdb of=/dev/sdc
# 或者复制到文件
dd if=/dev/sdb of=flash_drive.img
```

## 15.6 创建CD-ROM映像 ##

向CD-ROM(CD-R或是CD-RW)写入数据包括两个步骤, 首先创建一个ISO映像文件, 即CD-ROM文件系统映像; 然后将此映像文件写入到CD-ROM介质中.

### 15.6.1 创建一个CD-ROM文件映像副本 ###

使用dd命令将CD光盘设备制作为ISO文件:

```
dd if=/dev/cdrom of=ubuntu.iso
```
该方法同样适用于数据类DVD, 但不适用于音频DVD, 因为音频DVD不使用文件系统实现存储, 对于音频CD可以使用cdrdao命令.

### 15.6.2 从文件集合中创建映像文件 ###

genisoimage命令通常用于创建包含一个目录内容的ISO映像文件, 首先创建一个目录包含有所有希望加入映像文件的文件, 然后使用该命令创建映像文件即可.

```
genisoimage -o cd-rom.iso -R -J ~/cd-rom-files
```
-R选项添加了允许ROck Ridge延伸的元数据, 此延伸允许在Linux中使用较长文件名以及POSIX风格的文件; -J选项允许Joliet延伸, 此延伸允许在Windows中使用较长文件名.

## 15.7 向CD-ROM写入映像文件 ##

### 15.7.1 直接挂载ISO映像文件 ###

```
mkdir /mnt/iso_image
mount -t iso9660 -o loop image.iso /mnt/iso_image
```

### 15.7.2 擦除可读写CD-ROM ###

```
wodim dev=/dev/cdrw blank=fast
```

### 15.7.3 写入映像文件 ###

```
wodim dev=/dev/cdrw image.iso
```
wodim还支持很多选项, 常见的-v选项强调输出显示详细信息, -dao 选项则决定光盘刻录采用一次刻录模式, 该模式一般用户光盘商业化生产. wodim默认模式是一次轨道模式, 用户录制音乐曲目.

## 15.8 附加认证 ##

可以使用md5sum命令生成校验和, 以确保文件的内容没有发生变化.

```
md5sum image.iso
```

md5sum还可用于验证新刻录的光学介质. 通常只计算光学介质中包含该映像文件部分的校验和, 常用的计算方法如下: 计算映像文件中包含的2K字节块的数量(光学介质总是以2K字节的块为单位进行擦写),
然后从光学介质中读取同样多数量的块, 计算这些块的校验和.

某些光学介质不需要计算块数量, 例如一次刻录的CD-R可以直接使用以下命令进行计算:

```
md5sum /dev/cdrom
```

但也有许多类型的介质如DVD, 需要精确计算块的数量:

```
md5sum dvd-image.iso;
dd if=/dev/dvd bs=2048 count=$(( $(stat -c "%s" dvd-image.iso) / 2048 )) | md5sum
```
