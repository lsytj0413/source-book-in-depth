# 第十八章: 结合 SVN 版本库使用 Git #

## 例子: 对单一分支的浅克隆 ##

先对单个 SVN 分支做一个浅克隆:

```
# 从 SVN 的 1.5.x 分支上取出 33005~33142 的特定修订集
$ git svn clone -r33005:33142 \
http://svn.collab.net/repos/
svn/branches/1.5.x/ svn.git
```

- 现在可以对导入的提交使用 Git 操作, 只有 git svn 命令和 SVN服务器进行通信
- 在工作目录中没有 .svn 目录, git svn 会使用一个 .git/svn 目录
- 即使检出了 1.5.x 分支, 但是本地分支还是叫 master, 它有一个远程版本库的引用, 叫做 git-svn
- 作者的名字和 email 在 git log 中显示的比较特别, 作者名是 SVN登录名， email 包含 SVN 版本库的唯一ID
- 如果在 SVN的提交中没有写提交日志, 那么 git log 会使用一些 diff信息来显示
- 在每条提交中, 还有以 git-svn-id 开头的一行, 用来追踪提交来源
- git svn 会为每个提交生成一个新的提交ID

### 在 Git 中进行修改 ###

使用以下命令代替 git push 将提交推送到远程版本库:

```
$ git svn dcommit
```

### 在提交前进行抓取操作 ###

使用以下命令获取最新版本:

```
$ git svn fetch
```

### 通过 git svn rebase 提交 ###

将你的修改变基到 git-svn 分支的顶部:

```
$ git checkout master
$ git rebase git-svn
```

## 在 git svn 中使用推送, 拉取, 分支和合并 ##

非线性的历史记录不会在 SVN 中, SVN 仅仅会记录压缩的合并提交.

### 直接使用提交 ID ###

要在 git svn 的用户间协作有一个简单的技巧: 保证只有一个 Git 版本库, 只对这个版本库使用 git svn fetch 或 git svn dcommit.

### 克隆所有分支 ###

使用以下代码克隆所有分支:

```
$ git svn clone --stdlayout --prefix=svn/
-r33005:33142 http://svn.collab.net/repos/svn svn-all.git
```

- 通过 --stdlayout 选项告诉 git svn 版本库分支以标准 SVN 的方式创建, 创建 /trunk, /branches 和 /tags 子目录, 以便维护开发, 分支和标签
- 通过 --prefix=svn/ 选项使所有远程引用以 svn/ 前缀开头

现在将 master 分支指向主干:

```
$ git reset --hard svn/trunk
```

### 分享版本库 ###

发布中心版本库:

```
$ mkidr ../svn-bare.git
$ cd ../svn-bare.git
$ git init --bare
$ cd ../svn-all.git
$ git push --all ../svn-bare.git

# 显示指定复制的分支
$ git push ../svn-bare.git 'refs/remotes/svn/*:refs/heads/svn/*'
```

### 合并回 SVN ###

假设将 new-feature 分支的变更提交到 svn/trunk 中:

```
$ git checkout svn/trunk
$ git merge --no-ff new-feature
$ git svn dcommit
```

## 在和 SVN 一起使用时的一些注意事项 ##

### svn:ignore 与 .gitignore ###

git svn 提供了两种方式将 svn:ignore 转变为 .gitignore:

1. git svn create-ignore 自动创建 .gitignore
2. git svn show-ignore 找到整个项目的 svn:ignore 属性, 然后输出并放到 .git/info/exclude 文件中

### 重建 git-svn 的缓存 ###

使用以下操作重建缓存信息:

```
$ cd svn-all.git
$ mv .git/svn /tmp/git-svn-backup
$ git svn fetch -r33005:33142
```
