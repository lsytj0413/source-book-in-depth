# 第七章: 分支 #

## 使用分支的原因 ##

- 一个分支通常代表一个单独的客户发布版
- 一个分支可以封装一个开发阶段
- 一个分支可以隔离一个特性的开发或者研究特别复杂的 bug
- 每一个分支可以代表单个贡献者的工作

分支和标签看着相似, 其实它们适用于不同的目的:

- 标签是一个静态的名字, 不随着时间而改变
- 分支是动态的, 随着每次提交而移动

## 分支名 ##

版本库中默认的分支命名为 master. 给分支命名基本上是任意的, 也可以创建一个带层次的分支名, 类似于 UNIX 的路径名.

分支命名需要遵守以下规则:

1. 可以使用斜杠创建一个分层的命名方案, 但是分支名不能以斜杠结尾
2. 分支名不能以减号开始
3. 以斜杠分组的组件不能以点号开始, 例如 feature/.new 是无效的
4. 分支名的任意地方都不能包含两个连续的点号
5. 此外, 分支名不能包含以下内容:
    - 任何空格或其他空白字符
    - 在 Git 中有特殊含义的字符, 包括波浪线, 插入符(^), 冒号, 问号, 星号和左方括号
    - ASCII 控制字符, 即小于八进制 \040 的字符, 或是 DEL字符(八进制\177)

这些分支命名规则是由 git check-ref-format 底层命令强制检测的.

## 使用分支 ##

在任何给定的时间里, 版本库中可能有多个不同的分支, 但最多只有一个当前或活动分支, 而这个分支往往是 Git 命令的隐含操作数. 默认情况下, master 是默认分支.

Git 不会保留分支的起源信息, 可以从分叉的新分支和源分支名使用以下命令找到:

```
git merge-base original-branch new-branch
```

## 创建分支 ##

新的分支基于版本库中现有的提交. 使用以下命令即可从当前分支的 HEAD 创建一个新的分支:

```
git branch branch-name [starting-commit]
```

其中 starting-commit 参数为新分支起始的提交或者一个分支名, 默认为 HEAD.

需要注意的是, git branch 命令只是把分支名引入版本库, 并没有将当前分支切换到新的分支.

## 列出分支名 ##

可以使用 git branch 命令列出版本库中的分支名.

## 查看分支 ##

git show-branch 命令提供比 git branch 更为详细的输出, 按时间以递序的形式列出对一个或多个分支有贡献的提交.

```
$ git show-branch
warning: refname 'v1.0.0' is ambiguous.
* [develop] fix grayburd -> garyburd
 ! [heads/v1.0.0] rename
  ! [master] Merge pull request #8 from lsytj0413/develop
---
*   [develop] fix grayburd -> garyburd
*   [develop^] install-packages.sh
*   [develop~2] --net=bridge
*   [develop~3] ContainerPort --net=host
- - [master] Merge pull request #8 from lsytj0413/develop
* + [master^2] autoremove
* + [master^2^] install build-essential in Dockerfile
* + [master^2~2] dockerapi
* + [master^2~3] registrator_test.go
* + [master^2~4] go-dockerclient
* + [master^2~5] implements AddxxxFactory and GetxxxFactory
- - [master^] Merge pull request #7 from lsytj0413/develop
* + [master^2~6] fix go test bug
* + [master^2~7] factory and adapter
* + [master^2~8] ModuleFacotry
* + [master^2~9] EcodeRequestParameterMissing
* + [master^2~10] errorCause
* + [master^2~11] remove mux
* + [master^2~12] Query Param
* + [master~2] apiRandomNext apiSegmentNext
- - [master~3] Merge pull request #6 from lsytj0413/develop
* + [master~3^2] ErrorXXX -> EcodeXXX
* + [master~3^2^] GIN_MODE
* + [master~3^2~2] LogHandler
- - [master~4] Merge pull request #5 from lsytj0413/develop
* + [master~4^2] gin router
* + [master~4^2^] middleware APIErrorHandler
* + [master~4^2~2] api.go
* + [master~5] modules.go
* + [master~6] serviceInfo
- - [master~7] Merge branch 'master' of https://github.com/lsytj0413/dhetis
- - [master~7^2] Merge pull request #4 from lsytj0413/develop
- - [master~7^2^] Merge pull request #3 from lsytj0413/develop
- - [master~7^2~2] Merge pull request #2 from lsytj0413/develop
* + [master~8] Dockerfile.dev
* + [master~9] remove unuse
* + [master~10] os.Getenv
* + [master~11] getFunc
* + [master~12] mv err -> error
* + [master~13] modify getLogLevel
* + [master~14] delete cli
* + [master~15] dhetis.go
* + [master~16] dhetismain
- - [master~17] Merge branch 'master' into develop
- - [master~7^2~3] Merge pull request #1 from lsytj0413/v1.0.0
*++ [heads/v1.0.0] rename
```

git show-branch 命令的输出被一排破折号拆分为两个部分: 破折号上方为分支名, 并显示每个分支的HEAD日志消息的第一行.

输出的下半部分是一个表示每个分支中提交的矩阵, 每个提交后面显示该提交日志的第一行. 如果有一个加号, 星号或减号在分支的列中, 即对应的提交就会在该分支中显示. 加号表示提交在一个分支中, 星号突出显示存在于活动分支的提交, 减号表示一个合并提交.

当调用时, git show-branch 会遍历所有显示在分支上的提交, 并在它们最近的共同提交处停止, 这是默认的启发策略. 如果需要更多的提交历史记录, 可以使用参数 --more=num 选项, 指定在共同提交后看到多少个额外的提交.

git show-branc 命令接受一组分支名作为参数, 允许限定在这些分支的历史记录中显示. 在这种方式中, Git 运行通配符匹配分支名, 例如只显示以 bug/ 开头的分支的历史:

```
$ git show-branch bug/pr-1 bug/pr-2
# 等价于
$ git show-branch bug/*
```

## 检出分支 ##

可以使用 git checkout 命令来检出一个分支, 并把该分支变成新的活动分支.

### 检出分支的一个简单例子 ###

例如检出 bug/pr-1 分支:

```
$ git checkout bug/pr-1
```

检出一个新分支可能会对工作树文件和目录结构产生巨大的影响, 例如:

- 在要被检出的分支中但不在当前分支中的文件和目录, 会从对象库中检出并放在工作树中
- 在当前分支但不在要被检出分支中的文件和目录, 会从工作树中删除
- 两个分支都有的内容会被替换为要检出分支中的内容

### 有未提交的更改时进行检出 ###

如果一个文件的本地修改不同于新分支上的变更, 那么 Git 会拒绝检出目标分支. 例如以下情况:

```
# 本地工作目录下的 NewStuff 文件内容如下
$ cat NewStuff
Something
Something else

# 本地分支 HEAD 上的文件内容
$ git show master:NewStuff
Something

# dev 分支的内容
$ git show dev:NewStuff
Something
A Change

# 尝试检出 dev 分支, 会报错
$ git checkout dev
# 可以使用 -f 命令强制检出
```

错误信息会提示将修改提交到任意分支, 然后才能继续检出操作. 此时你可以将变更提交到 master 分支, 但是如果你想要提交到 dev 分支, 那么此时是一个困难的情况: 你不能把变更提交到 dev 分支, 因为你不能检出该分支.

### 合并变更到不同分支 ###

在上一节中, 当前工作目录的状态与你想切换到的分支相冲突, 而我们需要把工作目录中的修改和被检出的文件合并. 可以使用 -m 参数来要求 Git 通过在本地的修改和对目标分支之间进行一次合并操作, 尝试将本地的修改加入到新工作目录中:

```
git checkout -m dev
```

成功的检出了 dev 分支, 但是此时可能还需要解决合并文件导致的合并冲突.

### 创建并检出新分支 ###

Git 提供了 -b 选项来实现创建一个新的分支并同时切换到该分支.

```
$ git checkout -b new-branch [start-point]
# 等价于
$ git branch new-branch [start-point]
$ git checkout new-branch
```

### 分离 HEAD 分支 ###

通常情况下, git checkout 会改变期望分支的头部; 然后可以检出任何提交, 在这时 Git 会创建一种匿名分支, 称为一个分离的 HEAD. 在以下情况下 Git 会创建一个分离的 HEAD:

1. 检出的提交不是分支的 HEAD
2. 检出一个追踪分支
3. 检出标签引用的提交
4. 启动一个 bisect 操作
5. 使用 git submodule update 命令

如果你在一个分离的 HEAD 上, 并决定用新的提交留住它们, 那么使用以下命令创建一个新分支即可:

```
# 基于分离的 HEAD 创建一个新分支
$ git checkout -b new-branch
```

或者你只需放弃这种状态, 那么使用以下命令检出到一个存在的分支即可:

```
$ git checkout branch
```

## 删除分支 ##

可以使用 git branch -d branch 从版本库中删除分支, Git 会阻止你删除当前分支, 而且 Git 也会阻止你删除一个包含不存在与当前分支中的提交的分支, 在这种情况下可以使用 -D 选项来实现删除.

Git 会删除那些不再被引用的提交和不能从某些命名的引用(如分支或标签名)可达的提交. 如果想保留在分支中被删除的提交, 可以将它们合并到另一个分支, 或是为它们创建一个分支, 也可以用标签指向它们.
否则如果没有对它们的引用和提交, blob 是不可达的, 它们最终会被 Git gc 工具回收.

在意外删除分支或其他引用后, 可以使用 git reflog 命令恢复; 其他命令如 git fsck 或配置选项 gc.reflogExpire 和 gc.pruneExpire 同样可以帮助恢复丢失的提交, 文件和分支的头部.
