# 第二十章: 提示, 技巧和技术 #

## 对脏的工作目录进行交互式变基 ##

在工作目录是脏的情况下, 如果使用 git rebase -i 命令 Git 会拒绝进行变基. 这时可以使用 git stash 清理脏的工作目录:

```
$ git stash
$ git rebase -i master~4
$ git stash pop
```

## 删除剩余的编辑器文件 ##

假设你在版本库中无意添加了编辑器的临时文件, 可以使用如下命令解决:

```
$ git filter-branch --tree-filter 'rm -f *~' -- --all
```

## 垃圾回收 ##

Git 会使用垃圾回收(garbage collection)算法来清理未引用的对象, 可以使用 git gc 命令来触发垃圾回收, 该命令有两个作用:

1. 保持版本库对象库的整洁
2. 通过定位未压缩的对象并生成打包文件, 从而优化版本库的大小

Git 会在以下几种情况下自动进行垃圾回收:

1. 版本库里有过多的松散对象
2. 当推送到一个远程版本库时
3. 在一些命令引入许多松散对象之后
4. 但一些命令(如 git reflog expire) 明确要求时

可以在以下几种情况下手动运行 git gc:

1. 刚刚完成一个 git filter-branch
2. 在一些命令引入许多松散对象之后, 例如大量变基操作

在以下情况应该警惕垃圾回收:

1. 有想要恢复的孤立文件
2. 在 git rerere 的情况下, 而你又不必永久保存该解决方案
3. 在只有标签和分支就足以让 Git 永久保留提交的情况下
4. 在获取 FETCH_HEAD 的情况下, 因为它们即会进行垃圾回收

对于垃圾回收, 有以下几个比较重要的 git config 参数:

1. gc.auto: 在被垃圾回收打包之前, 版本库中允许存在的松散对象的数量, 默认值为 6700
2. gc.autopacklimit: 版本库内运行存在的打包文件的数量, 默认值为 50
3. gc.pruneexpire: 不可达对象在对象库里的持续时间, 默认值为两周
4. gc.reflogexpire: git reflog expire 命令会删除比这个时间旧的 reflog 条目, 默认值为 90 天
5. gc.reflogexpireunreachable: 仅当 reflog 条目在当前分支不可达时, 大于一定时间限制的条目才会被 git reflog expire 命令删除, 这个时间限制的默认值是 30 天

大部分垃圾回收配置参数都具有表示现在就做或永远不做的值.

## 拆分版本库 ##

可以使用 filter-branch 命令来拆分版本库或者提取子目录, 同时还保存目录为止的历史记录. 假设版本库中包含 part1, part2, part3 和 part4 这几个顶层目录, 可以使用如下命令将 part4 目录拆分成一个独立的版本库:

```
# 先删除所有 origin 的远程引用, 防止远程版本库被无意间破坏
$ git filter-branch --subdirectory-filter part4 HEAD

# 保留标签和所有分支
$ git filter-branch --tag-name-filter cat \
--subdirectory-filter part4 -- --all
```

## 恢复提交的小贴士 ##

使用以下命令调整 reflog 过期时间:

```
$ git config --global gc.reflogExpire "6 months"
$ git config --global gc.reflogExpireUnreachable "60 days"
$ git config --global gc.pruneexpire="1 month"
```

可以使用以下别名:

```
$ git config --global \
    alias.orphank=!gitk --all 'git reflog | cut -c1-7'&
$ git config --global \
    alias.orphanl=!git log --pretty=oneline --abbrev-commit \
    --graph --decorate 'git reflog | cut -c1-7'
```

## 转换 Subversion 的技巧 ##

### 普适建议 ###

同时维护一个 SVN 版本库和 Git 版本库工作量非常大, 有以下一些建议:

1. 确保需要这样做
2. 一次完成你的所有导入, 清理和转换
3. 确定是否需要删除 SVN 提交标识符
4. .git 目录中关于 SVN 转换的元数据会丢失
5. 取保一个很好的作者和 -email 映射文件
6. Github 的 Subversion Bridge 是一个简洁的选择

### 删除 SVN 导入后的 trunk ###

保留 SVN 的 trunk 目录是没有必要的, 可以使用如下命令删除:

```
$ git filter-branch --subdirectory-filter trunk HEAD
```

trunk 目录下的所有文件都会提升到同一级, 而 trunk 目录则会被删除.

### 删除 SVN 提交 ID ###

首先使用以下命令删除 SVN 提交 ID:

```
$ git filter-branch --msg-filter 'sed -e "/^git-svn-id:/d"'
```

然后删除 reflog:

```
$ git reflog expire --verbose --expire=0 -all
```

删除旧的分支:

```
$ rm -rf .git/refs/original
$ git reflog expire --verbose --expire=0 -all
$ git gc --prune=0
$ git repack -ad
```

## 操作来自两个版本库的分支 ##

需要比较的版本库必须拥有同一个祖先, 这样 Git 能够法系拿两个库的提交图和分支历史并能够将它们联系在一起. 而且 Git 只可以比较版本库里的分支, 所以只需要将所有版本库里的所有分支会聚到一个库里, 即可进行对比.

## 从上游变基中恢复 ##

也许可以通过简单的 git pull --rebase 命令进行恢复, 或者采用以下步骤:

1. 重命名你旧的上游分支, 例如 git branch save-origin-master origin/master
2. 从上游 fetch
3. 使用 cherry-pick 或 rebase 命令将你的提交从重命名的分支变基到新的上游分支, 例如 git rebase --onto origin/master save-origin-master master
4. 清理并删除临时分支, 例如 git branch -D save-origin-master

## 定制 Git 命令 ##

首先编写以下脚本:

```
#!/bin/sh
#git-top-check --Is this the top level directory of a Git repo?

if [ -d ".git" ]; then
    echo "This is a top level Git development repository."
    exit 0
fi

echo "This is not a top level Git development repository."
exit -1
```

然后将脚本存放在文件 ~/bin/git-top-check 或者 PATH 下, 这时就可以使用下面的命令:

```
$ git top-check
```

## 快速查看变更 ##

可以使用如下命令查看变更:

```
$ git whatchanged --since="three days ago" --oneline
```

每个提交都会跟着一个文件列表, 包括文件名 和 blob 的 SHA1.

也可以使用以下命令查看其他分支或限制文件路径:

```
$ git whatchanged ORIG_HEAD..HEAD --oneline Makefile
```

## 清理 ##

git clean 命令可以从你的工作树中移除未追踪的文件. 默认情况下它只删除所有不受版本控制的文件(保留目录, 除非使用 -d 选项).

在 .gitignore 和 .git/info/exclude 文件内提到的文件不是不受版本控制的, 除非使用 -x 选项, 不然 Git 不会把这些文件清理掉, 也可以使用 -X 选项来删除被 Git 忽视的文件. 也可以先执行 --dry-run 命令来尝试获取被删除文件的列表输出.

## 使用 git-grep 来搜索版本库 ##

git grep 命令可以搜索版本库内文件的内容:

```
$ git grep -i loeliger
```

## 更新和删除 ref ##

可以使用以下命令更新 ref:

```
$ git update-ref someref SHA1
```

删除 ref:

```
$ git update-ref -d someref
```

## 跟踪移动的文件 ##

可以使用如下命令查看文件的提交日志, 包括移动之前的历史记录:

```
$ git log --follow file
```

可以使用 --name-only 选项来显示重命名的文件名.

## 保留但不追踪文件 ##

可以使用如下命令来保留文件:

```
# 标记文件未不追踪
$ git update-index --no-assume-unchanged Makefile

# 标记文件为追踪
$ git update-index --assume-unchanged Makefile
```

## 你来过这里吗 ##

可以使用 rerere(重用已录制的解决方法, reuse record resolution) 功能来自动解决相同的合并或变基冲突.

```
# 启用配置
$ git config --global rerere.enabled true
```

在启用之后会在 .git/rr.cache 目录下记录合并冲突的左右两侧, 并在相同的冲突发生时自动解决冲突. 该功能会阻止合并的自动提交, 以便能在冲突解决方案成为提交历史之前能再检查一遍.

rerere 有一个明显的缺点: rr-cache 目录不具有可移植性, 不能通过 push 或 pull 操作来传输.
