# 第二章: 安装 Git #

## Linux ##

### Debian/Ubuntu ###

Git 包含以下的包:

- git
- git-arch
- git-cvs
- git-svn
- git-gui
- gitk
- gitweb
- git-email
- git-daemon-run

可以通过以下命令安装重要的 Git包:

```
apt-get install git git-doc gitweb git-gui gitk git-email git-svn
```

### 其他发行版 ###

```
# Gentoo
emerge dev-util/git
# Fedora
yun install git
```

## 源码安装 ##

```
# 获取源码
cd git-1.7.9
./configure
make all
make install
```

## Windows ##

在 Windows 上有两个 Git 包:

- 基于 Cygwin 的 Git
- 原生版本的 msysGit
