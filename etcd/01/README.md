# 1.1 etcd 简介 #

etcd 是 CoreOS 团队于 2013 年 6 月发起的开源项目, 它的目标是构建一个高可用的分布式kv数据库, 基于 Go 语言实现.

## etcd-raft 的使用分析 ##

在 etcd 中实现了一个相对独立的 raft 协议库, 代码在 github.com/coreos/etcd/raft 目录下. 并且 etcd 实现了一个使用该库的 example 项目, 地址在 github.com/coreos/etcd/contrib/raftexample 目录下.

httpapi 中包含一个 store, 该 store 的实现类型为 kvstore, 并包含一个 proposeC channel(同时传递给了 raftNode).

一个客户端读请求被 raft 处理的流程如下:

1: httpapi.go 中实现客户端请求接口的处理(ServeHTTP函数)
2: 如果为 GET 则直接从 store 中读取(此处不考虑非 master 或 linearizable read)

一个客户端写请求被 raft 处理的流程如下:

1: httpapi.go 中实现客户端请求接口的处理(ServeHTTP函数)
2: 如果为 PUT 则调用 store 的 Propose 函数, 并转发到 proposeC channel
3: raft.go 文件中的 serveChannels 方法会从 proposeC 中读取数据
4: 将读取到的数据交由 raft.Node 接口处理
5: raft.Propose(将数据封装为 MsgProp) 中调用 node.step 函数, 并将消息发送到 n.propc 中
6: api 处理完成(实际工作中还需要等待结果), propc 中的消息会在 node.run 中取出处理
7: node.run 从 propc 中取出一条消息, 将 From 赋值为 raft.id, 然后调用 raft.Step
8: 在 raft.Step 中先处理消息的任期, 可能导致本节点转换为 follower. 然后调用 r.step
9: r.step 是一个函数, 可能为 stepLeader, stepFollower, stepCandicate
10: 在 stepLeader 中, 首先判断自己还是不是集群成员(可能被 ConfChange 移除)
11: 判断 ConfChange 消息, 保证同一时刻只有一个 ConfChange
12: 将 entries 保存到 r.raftLog (unstable)
13: 更新 commited (计算每个节点的 Match, 取最小值)
14: 调用 raft.bcastAppend 将 propc 发送给 follower, 对每个节点调用 raft.sendAppend
15: 在 raft.sendAppend 函数中, 首先判断是否需要发送 snapshot, 然后将消息类型转换为 MsgApp, 并复制 entries 转发给 follower, 调用 raft.send 函数(将内容保存到 r.msgs)
16: 在 node.run 函数中会调用 raft.newReady 函数, 获取所有的 msgs
17: 将 Ready 数据交由外部的程序处理, 此处转到 example 中的 raftNode.serveChannels 函数
18: 将 Ready.Entries(来自于 unstable) 保存到 raftStorage
19: 将 Ready.Messages 交由 rafthttp.Transport 转发给 follower -> 23
20: 将 Ready.CommittedEntries 发送到 raftNode.commitC channel
21: 调用 raft.Node.Advance 准备接收下一批次的 Ready
22: 发送到 commitC 中的数据, 在 kvstore.readCommits 函数中取出, 然后应用到状态机
23: 在 rafthttp.Transport.Send 函数中, 调用 rafthttp.Peer.send 将数据发送给 follower(传递到 writec)
24: writec 来自于定义的 writer, 取值为 streamWriter, writec 为 streamWriter.msgc
25: 在 streamWriter.run 中读取 msgc 中的消息, 然后使用 messageEncoder.encode 函数编码数据(写入到 conn.Writer), 并在 msgc 中没有更多消息或者缓冲数量达到一定要求时进行发送(http.flusher.Flush)

follower 的处理:

26: 在 streamReader.decodeLoop 中读取到数据, 然后发送到 recvc(即 peer.recvc)
27: 在 startPeer 函数中启动一个 goroutine, 循环从 recvc 中获取数据, 调用 Raft.Process 处理, 其中 Raft 取值来源于 transport.Raft. transport.Raft 在 raftexample.raftNode.startRaft 函数中被赋值为 raftNode
28: 在 raft.node.step 函数中将消息发送到 n.recvc, 进一步调用到 stepFollower, 然后将日志添加到 raft.msgs, 等待外部调用 Ready 进行处理
