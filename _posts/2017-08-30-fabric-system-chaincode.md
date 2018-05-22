---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 system chaincode
tags:
  - Fabric
---

### cscc:PeerConfiger
```
{
        Enabled:           true,
        Name:              "cscc",
        Path:              "github.com/hyperledger/fabric/core/scc/cscc",
        InitArgs:          [][]byte{[]byte("")},
        Chaincode:         &cscc.PeerConfiger{},
        InvokableExternal: true, // cscc is invoked to join a channel
},
```
主要处理参数："JoinChain","GetConfigBlock","GetChannels".
主要用处：对peer进行config,具体如下：
1. **channel join**过程中会调用，涉及到其**JoinChain**过程，其作用主要是通过block创建chain,并进行初始化。详见：[Hyperledger fabric 1.0 代码解析 之 channel join](http://192.168.9.251:4567/topic/14/hyperledger-fabric-1-0-%E4%BB%A3%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B9%8B-channel-join)的第3步到第7步。
2. **chaincode的InitCmdFactory**过程，对于invoke,instantiate等需要orderer参与的过程，在InitCmdFactory中需要GetOrdererEndpointOfChain中会涉及到cscc的**GetConfigBlock**过程,主要用处是从css获得configblock,然后从configblock中解析出orderer的address.
3. **channel list**过程会调用，涉及到其**GetChannels**过程，其作用主要是获取peer加入了那些channel。
### lscc:LifeCycleSysCC
```
 {
        Enabled:           true,
        Name:              "lscc",
        Path:              "github.com/hyperledger/fabric/core/scc/lscc",
        InitArgs:          [][]byte{[]byte("")},
        Chaincode:         &lscc.LifeCycleSysCC{},
        InvokableExternal: true, // lscc is invoked to deploy new chaincodes
        InvokableCC2CC:    true, // lscc can be invoked by other chaincodes
},
```
主要处理参数： "install","deploy","upgrade","getid","getdepspec","getccdata","getchaincodes","getinstalledchaincodes"
主要用处：管理chaincode的整个生命周期，具体如下：
1. **chaincode install**过程中会调用，涉及到其**install**过程，其作用主要是将chaincode写入文件系统。详见:[Hyperledger fabric 1.0 代码解析 之 chaincode install](http://192.168.9.251:4567/topic/17/hyperledger-fabric-1-0-%E4%BB%A3%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B9%8B-chaincode-install)的第3步和第4步。
2. **chaincode instantiate**过程中会调用，涉及到其**deploy**过程,其主要作用是将用户的chaincode put到lscc中。详见：[Hyperledger fabric 1.0 代码解析 之 chaincode instantiate or upgrade](http://192.168.9.251:4567/topic/20/hyperledger-fabric-1-0-%E4%BB%A3%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B9%8B-chaincode-instantiate-or-upgrade)
3. **chaincode upgrade**过程会调用，涉及到其**upgrade**过程,其主要作用是将用户的chaincode put到lscc中。
4. **chaincode invoke**过程中，对于**非system chaincode会首先需要从lscc中得到chaincode data**,涉及其**getccdata**过程。
5. **chaincode Lanuch**过程中，对于部分情况，需要调用GetCDSFromLSCC，涉及到lscc的**getdepspec**过程。
6. "getid","getchaincodes","getinstalledchaincodes"暂未找到相关调用。
### escc:EndorserOneValidSignature
```
{
        Enabled:   true,
        Name:      "escc",
        Path:      "github.com/hyperledger/fabric/core/scc/escc",
        InitArgs:  [][]byte{[]byte("")},
        Chaincode: &escc.EndorserOneValidSignature{},
},
```
主要处理参数：只做签名背书用，无参数
主要用处:**ProcessProposal**中的**endorseProposal**过程会调用，用以对模拟执行的结果进行签名。
### vscc:ValidatorOneValidSignature
```
{
        Enabled:   true,
        Name:      "vscc",
        Path:      "github.com/hyperledger/fabric/core/scc/vscc",
        InitArgs:  [][]byte{[]byte("")},
        Chaincode: &vscc.ValidatorOneValidSignature{},
},
```
主要处理参数：只做验证背书用，无参数
主要用处：peer端在处理**deliverBlock**（从orderer端传过来的block）的时候,在真正**commit block**过程中做证实用。
### qscc:LedgerQuerier
```
{
        Enabled:           true,
        Name:              "qscc",
        Path:              "github.com/hyperledger/fabric/core/chaincode/qscc",
        InitArgs:          [][]byte{[]byte("")},
        Chaincode:         &qscc.LedgerQuerier{},
        InvokableExternal: true, // qscc can be invoked to retrieve blocks
        InvokableCC2CC:    true, // qscc can be invoked to retrieve blocks also by a cc
},
```
主要处理参数："GetChainInfo"，"GetBlockByNumber"，"GetBlockByHash"，"GetTransactionByID"，"GetBlockByTxID"
主要用处:客户端和SDK端调用，用来查询当前区块和交易等。