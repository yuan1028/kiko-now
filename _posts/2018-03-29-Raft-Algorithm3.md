集群中的成员变更不同于机器宕机重启，成员变更会影响到leader选举等决策。变更过程必须要保证安全，即在同一时刻同一term，只能有一个leader。
但是直接从一种配置转化到新的配置，可能将集群分裂为两个独立的集群。在下图中集群从3个节点增加到5个节点，由于每台服务器在不痛的时刻更新配置。就可能存在一个时间点，在一同任期中有两个不同的leader，其中一个是旧的配置（C-old）选举出的leader，另一个是新的配置（C-new）选举出的leader。
![disjoint majorities.jpg](https://upload-images.jianshu.io/upload_images/9290815-0f19b10f9d80218a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 两阶段方法
- 第一阶段，集群由之前的状态，转化到一个合集一致性的状态（joint consensus），即新旧节点都先拿到新的配置。
- 第二阶段，一旦joint consensus被提交，则集群转化到新的配置。

![1522317081208.jpg](https://upload-images.jianshu.io/upload_images/9290815-3cb2aeef9eb758e5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
leader接收到一个改变配置从 C-old 到 C-new 的请求，会将并集的配置（C-old,new）以日志条目的形式存储并发送到新老所有节点。一旦一个服务器将（C-old,new）配置日志条目提交后，该节点就会用这个配置（C-old,new）来做出未来所有的决定。raft的机制保证只有拥有 C-old,new 日志条目的服务器才有可能被选举为领导人。当C-old,new日志条目被提交以后，leader在使用相同的策略提交C-new，如上图所示，C-old 和 C-new 没有任何机会同时做出单方面的决定，这就保证了安全性。
另外在集群成员变更的时候有3个问题需要强调。
1. 新节点要加入的时候没有存储任何日志条目，如果该节点直接加入，可能要花一段时间来追赶日志。而这段时间可能无法响应client的请求。Raft解决该问题的方法是增加一个新的阶段，先将新的节点作为不计票的成员加入到集群。等到该新节点日志一致后再开始配置的更新。
2. leader节点有可能不在新的配置中。这种情况下，leader一旦提交了C-new，就会转变成follower状态。
3. 删除节点有可能导致集群崩溃。这些被删除掉的节点将会收不到心跳，从而他们会反复地开启新的选举。raft中采取当follower确定目前有leader的时候拒绝掉RequestVote RPCs来解决该问题。确定目前有leader的方式是设置了一个最小的election timeout时间，在该时间内收到RequestVote RPCs时拒绝。由于正常选举，节点的时间都会被该时间要大，则该优化不会影响到正常选举。



