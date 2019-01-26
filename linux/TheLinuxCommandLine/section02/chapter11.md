# 第十一章: 环境 #

在shell会话调用环境期间, shell会存储大量的信息, 程序使用存储在环境中的数据来确定我们的配置. 本章介绍的命令如下:

- printenv: 打印部分或全部的环境信息
- set: 这只shell选项
- export: 将环境导出到随后要运行的程序中
- alias: 为命令创建一个别名

## 11.1 环境中存储的是什么 ##

shell中有两种数据类型: 环境变量和shell变量. shell变量是由bash存放的少量数据, 环境变量是除此之外的所有其他变量. 除变量之外shell还存储了一些编程数据, 也就是别名和shell函数.

### 11.1.1 检查环境 ###

bash中集成了set命令和printenv命令, set命令会同时显示shell变量和环境变量, 而printenv只会显示环境变量.

```
printenv | less
printenv USER
```
使用set命令时如果不带选项或参数, 只会显示shell变量, 环境变量以及任何以定义的shell函数, set命令的输出结果是按照字母顺序排列的.

```
set | less
echo $HOME
```
set命令和printenv命令都不能显示别名, 要查看别名需要使用不带任何参数的alias命令。

### 11.1.2 一些有趣的变量 ###

常用的环境变量如下:

| 变量 | 说明 |
|:--|:--|
| DISPLAY | 运行图形界面环境时界面的名称, 通常为:O, 表示由X服务器生成的第一个界面 |
| EDITOR | 用于文本编辑器的程序名称 |
| SHELL | 本机shell名称 |
| HOME | 本机主目录的路径名 |
| LANG | 定义了本机语言的字符集和排序规则 |
| OLD_PWD | 先前的工作目录 |
| PAGER | 用于分页输出的程序名称, 通常为 /usr/bin/less |
| PATH | 以冒号分割的一个目录列表, 在该目录列表中查找可执行程序名称 |
| PS1 | 提示符字符串1, 定义了本机shell系统提示符的内容 |
| PWD | 当前工作目录 |
| TERM | 终端类型名称, 类UNIX系统支持多种终端协议, 此变量指定协议类型 |
| TZ | 指定本机所处的时区, 大多数类UNIX系统用UTC来维护计算机的内部时钟, 而显示的本地时间是根据本变量确定的时差计算出来的 |
| USER | 用户名 |

## 11.2 环境是如何建立的 ##

用户登录系统后, bash程序会启动并读取一系列称为启动文件的配置脚本, 这些脚本定义了所有用户共享的默认环境. 接下来bash会读取更多存储在主目录下的用于定义个人环境的启动文件. 这些步骤的执行顺序是由启动的shell会话类型决定的.

### 11.2.1 login和non-login shell ###

shell会话存在两种类型: login shell和non-login shell会话.

login shell会话会提示用户输入用户名和密码, 而在GUI中启动的终端会话就是一个典型的non-login shell会话.

login shell读取的启动文件如下表:

| 文件 | 说明 |
|:--|:--|
| /etc/profile | 适用于所有用户的全局配置脚本 |
| ~/.bash_profile | 用户个人的启动文件, 可扩展或重写全局配置脚本中的参数 |
| \~/.bash_login | 若\~/.bash_profile缺失则bash尝试读取此脚本 |
| \~/.profile | 若\~/.bash_profile和\~/.bash_login都缺失, 则bash尝试读取此文件, 在基于Debian的Linux版本中这是默认值 |

non-login shell读取的启动文件如下表:

| 文件 | 说明 |
|:--|:--|
| /etc/bash.bashrc | 适用于所有用户的全局配置脚本 |
| ~/.bashrc | 用户的个人启动文件 |

在读取以上启动文件之外, non-login shell还会继承父类进程的环境, 父类进程通常是一个login shell.

对普通用户来说, ~/.bashrc 可能是最重要的启动文件, 因为non-login shell会默认读取它, 而且login shell的启动文件也能以读取它的方式来编写.

### 11.2.2 启动文件中有什么 ###

一个常见的 .bashrc 文件如下(来自UBUNTU 16.04 TLS):

```
# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
HISTCONTROL=ignoreboth

# append to the history file, don't overwrite it
shopt -s histappend

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# If set, the pattern "**" used in a pathname expansion context will
# match all files and zero or more directories and subdirectories.
#shopt -s globstar

# make less more friendly for non-text input files, see lesspipe(1)
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, unless we know we "want" color)
case "$TERM" in
    xterm-color|*-256color) color_prompt=yes;;
esac

# uncomment for a colored prompt, if the terminal has the capability; turned
# off by default to not distract the user: the focus in a terminal window
# should be on the output of commands, not on the prompt
#force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
	# We have color support; assume it's compliant with Ecma-48
	# (ISO/IEC-6429). (Lack of such support is extremely rare, and such
	# a case would tend to support setf rather than setaf.)
	color_prompt=yes
    else
	color_prompt=
    fi
fi

if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi
unset color_prompt force_color_prompt

# If this is an xterm set the title to user@host:dir
case "$TERM" in
xterm*|rxvt*)
    PS1="\[\e]0;${debian_chroot:+($debian_chroot)}\u@\h: \w\a\]$PS1"
    ;;
*)
    ;;
esac

# enable color support of ls and also add handy aliases
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'

    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi

# colored GCC warnings and errors
#export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# some more ls aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'

# Add an "alert" alias for long running commands.  Use like so:
#   sleep 10; alert
alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'

# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

# enable programmable completion features (you don't need to enable
# this, if it's already enabled in /etc/bash.bashrc and /etc/profile
# sources /etc/bash.bashrc).
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi

alias ..="cd .."
export GOPATH=/home/soren/gosrc/lib:/home/soren/gosrc
export PATH=$PATH:/home/soren/gosrc/lib/bin
# export http_proxy="http://127.0.0.1:1080"
# export https_proxy=$http_proxy
# export ftp_proxy=$http_proxy
# export rsync_proxy=$http_proxy
```
文件中以 [#] 开头的行是注释行, 使用 export 命令导出环境变量, 以及通过if读取 ~/.bash_aliases文件和设定PATH变量的值.


## 11.3 修改环境 ##

### 11.3.1 用户应当修改哪些文件 ###

一般来说, 在PATH中添加目录或定义额外的环境变量, 则需要将这些更改放入到 .bash_profile或其他等效文件中, 其他修改则应录入 .bashrc 文件中.

### 11.3.2 文本编辑器 ###

文本编辑器大概分为两类: 图形界面的和基于文本的.

对于图形界面的, 例如GNOME中配备的编辑器叫做gedit, 而KDE中是kedit, kwrite和kate(复杂度递增).

对于基于文本的, 常见的是 nano, vi 和 emacs.

### 11.3.3 使用文本编辑器 ###

nano是第一个基于文本的编辑器, 例如我们用nano来编辑 .bashrc 文件:

```
nano .bashrc
```
nano 的屏幕显示内容分为三个部分: 顶端的标题, 中间的可编辑文本以及底部的命令菜单.

在nano中, 可以按下 Ctrl-X[即 ^X]退出程序, 按Ctrl-O完成保存, 使用方向键移动光标到目标区域即可进行编辑. 例如输入以下内容:

```
# 设置umask值
umask 0002
# 使shell的历史记录功能忽略与上一条录入的命令重复的命令
export HISTCONTROL=ignoredups
# 时命令历史记录规模从默认的500行增加到1000行
export HISTSIZE=1000
# 创建新命令 [l.], 显示以 [.] 开头的目录条目
alias l.='ls -d .* --color=auto'
# 创建新命令 [ll], 以长列表形式显示目录列表
alias ll='ls -l --color=auto'
```

### 11.3.4 激活我们的修改 ###

因为只有在启动shell时才会读取 .bashrc, 所以对它的修改只有在关闭shell终端会话并重启时才会生效, 也可以使用以下命令让bash重新读取该文件:

```
source .bashrc
```

## 11.4 扩展阅读 ##

bash 手册页的 INVOCATION 部分非常详细地讨论了 bash 启动文件.
