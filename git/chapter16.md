# 第十六章: 合并项目 #

子模块不但构成了你自己的 Git 版本库的一部分, 而且它也独立的存在于它自己的源代码版本库中.

## 旧解决方案: 部分检出 ##

传统的版本控制系统有一种流行的功能, 部分检出. 通过部分检出可以选择只检出特定的子目录, 并只在其中工作. 但当前的 Git 架构不支持部分检出技术.

## 显而易见的解决方案: 将代码导入版本库 ##

通过直接导入的方式来使用子模块, 有两种方法: 一是手动复制文件, 二是导入历史记录.

### 手动复制导入子项目 ###

直接把需要的文件复制进来即可.

### 通过 git pull -s subtree 导入子项目 ###

也可以直接合并该子项目的所有历史记录. 首先创建一个项目仓库 myapp, 然后导入 git 项目:

```
$ cd /tmp
$ mkdir myapp
$ cd myapp
$ git init
$ echo hello > hello.txt
$ git add hello.txt
$ git commit -m "first commit"

# 导入 git
$ mkdir git && cd git
$ (cd ~/git.git && git archive v1.6.0) | tar -xf -
$ cd ..
$ git add git
$ git commit -m "imported git v1.6.0"
```

然后使用 git pull -s ours 策略来通知 Git 你已经在 myapp 项目中导入了 git 项目:

```
# 简单的指定 v1.6.0 是不正确的, 需要完整的名称
$ git pull -s ours ~/git.git refs/tagsv1.6.0
```

现在可以在 git 目录中进行一个提交:

```
$ cd git
$ echo "i am a git" > c.txt
$ git add c.txt
$ git commit -m "my first c.txt to git"
```

现在 Git 子项目版本为 v1.6.0, 而且拥有一个额外的补丁. 然后再尝试将 git 更新至 v1.6.0.1:

```
$ git pull -s subtree ~/git.git refs/tags/v1.6.0.1
```

现在检查文件是否已经正确更新, 因为在 1.6.0.1 中所有的文件都被移动到了 git 目录中, 需要在 git diff 中采用选择器语法:

```
$ git diff HEAD^2 HEAD:git
```

可以看到, 与 v1.6.0.1 的唯一区别就是我们的那个补丁提交.

### 将更改提交到上游 ###

可以通过使用 -s subtree 合并策略将你的项目历史记录合并回 git.git 中, 但这回将你项目的完整历史导入, 然后在合并时删除除 git 目录以外的所有文件.

## 自动化解决方案: 使用自定义脚本检出子项目 ##

有一个显而易见的方法: 每次当你克隆主项目时, 就手动把子项目克隆到项目的子目录中:

```
$ git clone myapp myapp-test
$ cd myapp-test
$ git clone ~/git.git git
$ echo git >.gitignore
```

使用该方法最好编写一些简单的脚本并添加到版本库中以便于标准化.

有一些有关标准化和自动化这个过程的脚本: 例如 [externals(也叫ext)](https://nopus.com/ext-tutorial), 它可以非常方便的在任何 SVN 或者 Git 项目与子项目组合之间使用.

## 原生解决方案: gitlink 和 git submodule ##

Git 中有一条命令( git submodule )是为了子模块而设计的, 但是它有以下两个缺点:

- 比简单的导入子项目更复杂
- 和基于脚本的解决方案一致, 但是有更多的限制

git submodule 命令由两个独立的功能组合而成: gitlink 和 git submodule.

### gitlink ###

gitlink 就是从一个树对象到一个提交对象的链接, 即一个树对象指向一个提交对象. 使用同样的方式新建一个 myapp 仓库, 然后导入 git 项目:

```
$ git clone ~/git.git git
$ cd git
$ git checkout v1.6.0
$ cd ..

# 不能使用 git add git/, 这种方式不会创建 gitlink
$ git add git
$ git commit -m "imported git v1.6.0"
```

Git 通常将 gitlink 当作简单的指针值或者其他版本库的引用, 绝大部分的 Git 操作不会对 gitlink 解引用, 并作用在子模块版本库上.

使用 git clone 克隆一个新的 myapp 操作, 在操作后 git 子目录依然是空的, 需要使用 git submodule 命令将它们找回.

### git submodule 命令 ###

git submodule 在编写本书是实际上是一个 700 行的 UNIX shell 脚本, 它的工作原理很简单: 按照 gitlink 的链接检出相应的版本库.

git submodule 从一个名为 .gitmodules 的文件中检索需要的信息, 例如:

```
[submodule "git"]
    path = git
    url = /home/bob/git.git
```

使用这个文件需要创建 .gitmodules 文件, 可以手动创建或者通过 git submodule add 命令创建. 因为我们已经使用过 gitlink, 所以此时需要手动创建该文件:

```
cat >.gitmodules <<EOF
[submodule "git"]
    path = git
    url = /home/bob/git.git
EOF

# 等价于命令
git submodule add /home/bob/git.git git
```

然后使用 git submodule init 命令将 .gitmodules 文件中的设置复制到 .git/config 文件中:

```
$ git submodule init
```

最后, 可以输入 git submodule update 命令来更新子模块的文件:

```
$ git submodule update
```

git submodule update 命令会到你的 .git/config 文件中指定的版本库地址, 获取在 git ls-tree HEAD - git 中找到的提交ID, 并检出 .git/config 文件中指定的版本. 还有以下一些注意事项:

- 当切换分支或 git pull 别人的分支时, 始终需要运行 git submodule update 命令来获取匹配的子模块集
- 如果切换了分支但不运行 git submodule update, Git 会认为你故意改变了子模块使其指向一个新提交, 假如你接下来运行 git commit -a, 那么可能会修改 gitlink
- 可以通过检出某个子模块的正确版本, 然后在子模块目录上执行 git add, 接着执行 git commit 来更新一个现有的 gitlink
- 如果你在你的分支上更新并提交了某个 gitlink, 然后 git pull 或 merge 了另一个对该 gitlink 做了不同修改的分支, 那么 Git 会随便选择一个来解决这个冲突. 必须自己解决 gitlink 的冲突问题
