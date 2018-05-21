**State**
```
/* Persistent state on all servers:
(Updated on stable storage before responding to RPCs)
*/
currentTerm                   //当前任期   
votedFor                      //当前任期的候选者编号，无则为null
log[]                         //日志条目

//Volatile state on all servers,所有服务器上维护
commitIndex             //已知的最高的可被提交的日志条目的索引，初始为0
lastApplied             //当前已提交给state machine执行的条目的索引，初始为0

//Volatile state on leaders:(Reinitialized after election)，只在leader节点上维护
nextIndex[]          //对于每一台服务器，下一条将要发给该服务器的条目的索引，初始为leader最后一条条目索引+1
matchIndex[]         //每一个服务器已知的最高的已复制的条目的索引，初始为0
```
**RequestVote RPC**
```
//Invoked by candidates to gather votes (§5.2).
Arguments:
term                      //候选者的term值
candidateId               //候选者的id
lastLogIndex              //候选者最新的日志索引
lastLogTerm               //候选者最新的日志所属的term

Results:
term 
voteGranted                //true表示投票给该candidate

Receiver implementation:
1. Reply false if term < currentTerm 
//这里投票给候选者的条件是要求候选者的日志至少比自身的要新，也就是要么lastLogIndex比自身最新的日志条目index要大。
//要么lastLogIndex和lastLogTerm都和自身最新的日志条目一致
//这里对选举的这种限制是为了保证安全性。确保commit的日志一定不会被重写。
2. If votedFor is null or candidateId, and candidate’s log is at least as up-to-date as receiver’s log, grant vote
```
**AppendEntries RPC**

```
//Invoked by leader to replicate log entries; also used as heartbeat
Arguments:
term                               //leader当前的term值
leaderId                           //follower在收到client request时，可以用该值转发给leader
prevLogIndex                       //上一条日志条目的索引
prevLogTerm                        //上一条日志条目的term
entries[]                          //日志条目，对于心跳包则该值为空，日志条目可以为多条
leaderCommit                       //leader服务器的commitIndex

Results:
term                          //当前任期
success                       //具体的判断如下

Receiver implementation:
//任期值比当前任期小，则该RPC已失效，或当前leader已变更
1. Reply false if term < currentTerm 
//不包含匹配prevLogTerm的prevLogIndex所对应的条目，通常该情况为节点挂掉一段时间，落后leader节点
//leader会重新发包含较早的prevLogTerm及prevLogIndex的RPC给该节点
2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm 
// 以下均返回true
// 若日志条目已有内容与entries里的内容冲突，则删除已有及其后的条目
3. If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it 
// 将新的日志条目追加到日志中
4. Append any new entries not already in the log
//如果leaderCommit比自身commitIndex大，则更新自身的commitIndex为min(leaderCommit,当前最新日志条目索引)
5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
```
**InstallSnapshot RPC**
```
//Invoked by leader to send chunks of a snapshot to a follower. Leaders always send chunks in order.
Arguments:
term                               //leader的当前term
leaderId                           //leader的id
lastIncludedIndex                  //该snapshot中包含的最大的日志的索引值
lastIncludedTerm                   //该snapshot中包含的最大的日志的所属的term
offset                             //用来定位shapshot文件的偏移量，snapshot文件可能很大，要分几次传，每次称之为一个chunk
data[]                             //snapshot数据，通常为state machine的当前状态     
done                               //是否为最后一个chunk

Results:
term                               //currentTerm

Receiver implementation:
1. Reply immediately if term < currentTerm
//如果是第一个chunk，则新建snapshot文件
2. Create new snapshot file if first chunk (offset is 0)
//将data的数据写入到snapshot的相应位置
3. Write data into snapshot file at given offset
//如果done为false，则重复1-3过程，回复并等待最后一个chunk
4. Reply and wait for more data chunks if done is false
//保存snapshot文件，丢弃更老的snapshot
5. Save snapshot file, discard any existing or partial snapshot with a smaller index
// 已有的日志处理
6. If existing log entry has same index and term as snapshot’s last included entry, retain log entries following it and reply
// 丢弃老的日志
7. Discard the entire log
// 按照snapshot内容重设state machine
8. Reset state machine using snapshot contents (and load snapshot’s cluster configuration)
```

**Rules for Servers**
```
All Servers:
// commitIndex > lastApplied,证明lastApplied到commitIndex之间的日志条目都可以提交给state machine执行
• If commitIndex > lastApplied: increment lastApplied, apply log[lastApplied] to state machine 
// 若有新term，则更新自己的term值
• If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower 

Followers:
//响应leaders和candidates发来的RPC，响应规则参照AppendEntries和RequestVote部分
• Respond to RPCs from candidates and leaders
// 一段时间内，没有收到AppendEntries或者RequestVote的消息，则转变为candidate
• If election timeout elapses without receiving AppendEntries RPC from current leader or granting vote to candidate: convert to candidate

Candidates :
//开启选举，增加自身term值，投票给自己，重设定时器，发送RequestVote给其他服务器
 • On conversion to candidate, start election:
  • Increment currentTerm
  • Vote for self
  • Reset election timer
  • Send RequestVote RPCs to all other servers
//从多数成员收到true的回应，则转变为leader
• If votes received from majority of servers: become leader
//收到AppendEntries，证明新的leader已产生，则自身转变为follower
• If AppendEntries RPC received from new leader: convert to follower
//如果超时，则开启下一轮选举
• If election timeout elapses: start new election

Leaders:
//发送心跳包给所有服务器，防止其他服务器超时开启新的选举
• Upon election: send initial empty AppendEntries RPCs (heartbeat) to each server; repeat during idle periods to prevent election timeouts 
//收到客户端请求，则将条目写入到日志，当条目提交之后再回复给客户端
• If command received from client: append entry to local log, respond after entry applied to state machine 
//如果当前的日志条目索引比follower大（leader自身的last log index 与其nextIndex[]比较）,则发送AppendEntries给相应follower
• If last log index ≥ nextIndex for a follower: send AppendEntries RPC with log entries starting at nextIndex
     //如果成功，则更新nextIndex数组及matchIndex数组中的follower对应的项
    • If successful: update nextIndex and matchIndex for follower 
    //如果因为日志不同步失败，则减小该follower对应的nextIndex值然后重试,
   //若相应的nextIndex值减小到leader节点已经进行了snapshot，则leader会发送InstallSnapshot RPC
    • If AppendEntries fails because of log inconsistency: decrement nextIndex and retry 
    //如果更新完matchIndex后，判断下commitIndex是否可以更新，更新条件为新的值是多数同意的，且该条目的term为当前term
    • If there exists an N such that N > commitIndex, a majority of matchIndex[i] ≥ N, and log[N].term == currentTerm: set commitIndex = N.
```
