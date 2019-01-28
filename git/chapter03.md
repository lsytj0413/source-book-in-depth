# 第三章: 起步 #

## 命令行 ##

```
# 列出选项和常用子命令
git
# 完整的子命令列表
git help --all
# 版本号
git --version
# 子命令选项
git commit -amend
git --git-dir=project.git repack -d
# 子命令文档
git help subcommand
git --help subcommand
git subcommand --help
# 短选项和长选项
git commit -m "Fixed a typo."
git commit --message="Fixed a typo."
# 使用双破折号分离参数(控制部分和操作数部分)
git diff -w master origin -- tools/Makefile
# 使用双破折号分离参数(标识文件名)
git checkout main.c # 检出 tag main.c
git checkout -- main.c # 检出文件 main.c
```

## 快速入门 ##

### 创建初始版本库 ###

```
mkdir ~/public_html
cd ~/public_html
echo 'My website is alive!' > index.html
# 将目录转换为 Git 版本库
git init
```

git init 命令会创建一个隐藏目录, 在项目的顶层目录下, 名为 .git . Git 把所有的修订信息都放在这唯一的顶层 .git 目录中.

### 添加文件到版本库 ###

git init 命令创建一个新的版本库, 最初的版本库是空的, 需要将管理内容明确的放入到版本库中.

```
# 将 index.html 文件添加到版本库, 暂存状态
git add index.html
# 显示版本库状态
git status
# 提交到版本库
git commit -m "Initial" --author="xxx"
```

可以在命令行中提供提交日志消息, 但是更典型的做法是在交互式编辑器会话期间创建消息. 为了在 git commit 命令中打开编辑器, 需要设置 GIT_EDITOR 环境变量.

```
# tcsh
setenv GIT_EDITOR emacs
# bash
export GIT_EDITOR=vim
```

### 配置提交作者 ###

在最版本库进行提交之前, 需要建立一些基本环境和配置选项. 例如 Git 需要知道你的名字的 email 地址, 可以在 git commit 命令中指定, 也可以通过 git config 命令配置:

```
# 名字
git config user.name "name"
# email
git config user.email "email"
```

也可以使用 GIT\_AUTHOR\_NAME 和 GIT\_AUTHOR\_EMAIL 环境变量来指定名字和 email 地址, 这些变量的设置会覆盖所有的配置信息.

### 再次提交 ###

```
# 修改 index.html
cd ~/public_html
vim index.html
# 提交, 通过 commit 命令指定文件, 省略 git add
git commit index.html
```

### 查看提交 ###

```
# 查看一系列单独提交的历史
git log
# 查看特定提交的详细信息, 如果未指定 commit_id 则默认显示最新的提交
git show COMMID_ID
# 查看当前分支的单行摘要
git show-branch --more=10
```

### 查看提交差异 ###

```
# 查看两个e版本的差异
git diff COMMID_ID1 COMMID_ID2
```

### 删除和重命名文件 ###

假设在网站目录中已存在一个不再需要的 poem.html 文件 和 foo.html 文件:

```
# 删除文件
git rm poem.html
git commit -m "remove a poem"
# 重命名文件
git mv foo.html bar.html
git commit -m "Moved foo to bar"
```

### 创建版本库副本 ###

```
# 创建副本
cd ~
git clone public_html my_website
```

## 配置文件 ##

Git 配置文件都是 .ini 风格的文本文件, Git 支持不同层次的配置文件, 按照优先级递减的顺序如下:

- .git/config: 版本库特定配置设置, 可用 --file 选项修改, 这是默认选项, 拥有最高优先级
- ~/.gitconfig: 用户特定配置设置, 可用 --global 选项修改
- /etc/gitconfig: 系统范围配置, 可用 --system 选项修改, 该文件根据不同的系统可能存在不同的目录中

```
# 设置配置
git config --global user.name "global name"
git config --global user.email "global email"
git config user.email "file email"
# 查看配置
git config -l

# 查看版本库配置
cat .git/config

# 移除配置
git config --unset --global user.email
```

多个配置选项和环境变量常常是为了同一目的出现, 例如编辑提交日志的编辑器选项顺序是:

1. GIT_EDITOR 环境变量
2. core.editor 配置选项
3. VISUAL 环境变量
4. EDITOR 环境变量
5. vi 命令

可以通过以下命令来为一个命令设置别名:

```
# 设置别名
git config --global alias.show_graph 'log --graph --abbrev-commit --pretty=oneline'
# 使用别名
git show-graph
```
