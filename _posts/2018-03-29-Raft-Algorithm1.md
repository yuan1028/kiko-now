---
layout: post
title: Raft算法流程详解
tags:
  - Raft
---

[Raft官方网站](https://raft.github.io/)，其中有个5个节点可以自主控制的例子：
[一个很好的学习raft的动画](http://thesecretlivesofdata.com/raft/)
下文中的图片均来自raft论文。
##一致性问题
在分布式系统中,一致性问题(consensus problem)是指对于一组服务器，给定一组操作，我们需要一个协议使得最后它们的结果达成一致。
一致性的一般实现的原理：当其中某个服务器收到客户端的一组指令时,它必须与其它服务器交流以保证所有的服务器都是以同样的顺序收到同样的指令,所有的服务器产生一致的结果,看起来就像是一台机器一样。
##Raft算法中的基本概念
###复制状态机
![statemachine.jpg]({{ site.baseurl }}/images/statemachine.jpg)

上图为复制状态机的架构。简单看日志（Log）通常就是一系列指令等，状态机（State Machine）用来处理从日志中读出的指令。这里先假设状态机为确定性状态机，即按照相同顺序运行相同的指令后，达到的状态是一致的。因而只要保证日志中消息内容和顺序的一致性，即可保证状态的一致性。一致性算法负责确保日志的一致性，状态机的执行由各个服务器独立完成。
通常一致性算法中通常都会有一下几个特点。
- 在非拜占庭情况下，包括网络延时、分区、丢包、重复、和重新排序等错误情况，保证安全。
- 只要半数以上的节点存活且可以相互间通信以及与client通信，集群就是可用的。当有节点挂掉之后，可以从存储中恢复起来并重新加入到网络中。
- 不依靠时间来保证一致性。
- 少数慢的服务器（网络延迟较长、或者服务器挂掉等）不影响整个系统的性能。
###服务器状态
在Raft算法中，服务器有三种状态：
- Leader
- Follower
- Candidate

正常情况下，只会存在一个Leader，然后其他服务器均为Follower。Follower是被动响应的，并不会自己处理请求而是简单的回应Leaders或者Candidates。Leader节点处理所有客户端的请求。Candidate是用来选举出新的Leader的一个中间状态。
![role.png]({{ site.baseurl }}/images/role.png)
###Term
Raft中将时间划分为不定长度的任期Terms，Terms为连续的数字。每个Term以选举开始，如果选举成功，则集群会在当前Term下由当前的leader管理。如果选举失败，并没有选举出新的单一leader，则会开启新的Term，重新开始选举。
![term.jpg]({{ site.baseurl }}/images/term.jpg)
###RPC
在Raft算法中，有三种RPC
- RequestVote RPC，由candidate发送，要求其他节点选举其为leader的RPC
- AppendEntries RPC，由leader发送，向follower发送心跳或者同步日志。
- InstallSnapshot RPC，服务器之间传递镜像快照使用。
##Raft中的选举流程
Raft中使用心跳机制来出发leader选举。当服务器启动的时候，服务器成为follower。只要follower从leader或者candidate收到有效的RPCs就会保持follower状态。如果follower在一段时间内（该段时间被称为*election timeout*）没有收到消息，则它会假设当前没有可用的leader，然后开启选举新leader的流程。流程如下：
1. follower增加当前的term，转变为candidate。
2. candidate投票给自己，并发送RequestVote RPC给集群中的其他服务器。
3. 收到RequestVote的服务器，在同一term中只会按照先到先得投票给至多一个candidate。且只会投票给log至少和自身一样新的candidate。
4. candidate节点保持2的状态，直到下面三种情况中的一种发生。
4.1 该节点赢得选举。即收到大多数的节点的投票。则其转变为leader状态。
4.2 另一个服务器成为了leader。即收到了leader的合法心跳包（term值等于或大于当前自身term值）。则其转变为follower状态。
4.3 一段时间后依然没有胜者。该种情况下会开启新一轮的选举。

Raft中使用随机选举超时时间来保证当票数相同无法确定leader的问题可以被快速的解决。
## 日志复制与执行
当选举结束，leader就开始响应客户端的请求。流程如下：
1. leader将客户端的请求命令作为一条新的条目写入日志。
2. leader发送AppendEntries RPCs给其他的服务器去备份该日志条目。
3. follower收到leader的AppendEntries RPCs，将该条目记录到日志并返回回应给leader。
4. 当该条日志被安全的备份即leader收到了半数以上的机器回应该条目已记录，则可认为该条目是有效的，leader节点将该条目交由state machine处理，并将执行结果返回给客户端。leader节点在下次心跳等AppendEntries RPCs中，会记录可被提交的日志条目编号commitIndex。
5. follower在收到leader的AppendEntries RPCs，其中会包含commitIndex，follower判断该条目是否已执行，若未执行则执行commitIndex以及之前的相应条目。

![log.jpg]({{ site.baseurl }}/images/log.jpg)

##日志压缩
在正常运行过程中，raft集群的日志增长非常的快。通常使用镜像快照来压缩日志。即通过将当前的饿state写入到存储的snapshot中，然后到该点的日志即可被丢弃。
![snapshot.jpg]({{ site.baseurl }}/images/snapshot.jpg)
raft中的snapshot机制如下
- 在Raft中集群中的每个节点都会自主的产生snapshot
- snapshot中包含当前state machine状态、last included index（snapshot中包含的最后一条日志条目的索引）、last included term（snapshot中包含的最后一条日志条目所属的term）、集群成员信息。
- 当leader发现其要发送给follower的日志条目已经在snapshot中，则会发送installSnapshot RPCs给follower，该种情况通常发生在集群有新节点加入的时候。
- follower收到InstallSnapshot RPC的时候将snapshot写入到本地，并根据snapshot内容重设state machine，如果本地有更老的snapshot或者日志，则可以丢弃。

##集群中成员变更
集群中的成员变更不同于机器宕机重启，成员变更会影响到leader选举等决策。变更过程必须要保证安全，即在同一时刻同一term，只能有一个leader。
但是直接从一种配置转化到新的配置，可能将集群分裂为两个独立的集群。在下图中集群从3个节点增加到5个节点，由于每台服务器在不痛的时刻更新配置。就可能存在一个时间点，在一同任期中有两个不同的leader，其中一个是旧的配置（C-old）选举出的leader，另一个是新的配置（C-new）选举出的leader。
![disjoint majorities.jpg]({{ site.baseurl }}/images/disjointmajorities.jpg
## 两阶段方法
- 第一阶段，集群由之前的状态，转化到一个合集一致性的状态（joint consensus），即新旧节点都先拿到新的配置。
- 第二阶段，一旦joint consensus被提交，则集群转化到新的配置。

![joint consensus.jpg]({{ site.baseurl }}/images/jointconsensus.jpg)
leader接收到一个改变配置从 C-old 到 C-new 的请求，会将并集的配置（C-old,new）以日志条目的形式存储并发送到新老所有节点。一旦一个服务器将（C-old,new）配置日志条目提交后，该节点就会用这个配置（C-old,new）来做出未来所有的决定。raft的机制保证只有拥有 C-old,new 日志条目的服务器才有可能被选举为领导人。当C-old,new日志条目被提交以后，leader在使用相同的策略提交C-new，如上图所示，C-old 和 C-new 没有任何机会同时做出单方面的决定，这就保证了安全性。
另外在集群成员变更的时候有3个问题需要强调。
1. 新节点要加入的时候没有存储任何日志条目，如果该节点直接加入，可能要花一段时间来追赶日志。而这段时间可能无法响应client的请求。Raft解决该问题的方法是增加一个新的阶段，先将新的节点作为不计票的成员加入到集群。等到该新节点日志一致后再开始配置的更新。
2. leader节点有可能不在新的配置中。这种情况下，leader一旦提交了C-new，就会转变成follower状态。
3. 删除节点有可能导致集群崩溃。这些被删除掉的节点将会收不到心跳，从而他们会反复地开启新的选举。raft中采取当follower确定目前有leader的时候拒绝掉RequestVote RPCs来解决该问题。确定目前有leader的方式是设置了一个最小的election timeout时间，在该时间内收到RequestVote RPCs时拒绝。由于正常选举，节点的时间都会被该时间要大，则该优化不会影响到正常选举。
