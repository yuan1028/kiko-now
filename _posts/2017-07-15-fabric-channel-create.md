---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 channel create
tags:
  - Fabric
---
![channel create 流程图]({{ site.baseurl }}/images/channelcreate.png)

## Client端逻辑代码     
"github.com/hyperledger/fabric/peer/channel/create.go"

1.client发起channel create –c channel_name请求，其中参数channel_name为channel的名称。

2.client端从ConfigTx文件中（createChannelFromConfigTx）或者按照默认配置（createChannelFromDefaults）生成包体Envelope，并对生成的包体Envelope进行校验与签名，通过broadcast接口向orderer发送创建channel消息即签过名的包体Envelope。 （sendCreateChainTransaction）

3.client端收到channel创建成功消息后， 通过deliver接口获取orderer中第0块block（**getGenesisBlock**），并将该block存入本地生成channelname.block文件，作为该channel的创始块（**WriteFile**）。

## Orderer端逻辑代码   
"github.com/hyperledger/fabric/orderer"
     
1.列表orderer端对于broadcast、deliver和join分别有3个对应的handler来处理（server.go）.Handle为broadcast主处理函数(broadcast/broadcast.go)，负责peer消息的处理，包括channel的创建以及其他类型的消息，其中channel通过类型**HeaderType_CONFIG_UPDATE**进行区分。

2.若channelHeaderType是**HeaderType_CONFIG_UPDATE**，则为create channel或者是update channel的操作。此时调用**process**函数来获得新建或者更新的包体Envelope.
```go
if chdr.Type == int32(cb.HeaderType_CONFIG_UPDATE) {
            logger.Debugf("Preprocessing CONFIG_UPDATE")
            msg, err = bh.sm.Process(msg)
            //...
}
```
3.**Process**（Configupdate/configupdate.go）以channel相关信息生成Envelope消息，并将该Envelope消息重新打包到以TYPE: HeaderType_CONFIG_UPDATE为类型的消息体中。函数中通过判断包体传递过来的chainID是否已存在来判断是create还是update操作。chainID不存在，则调用**newChannelConfig**函数生成新的channelconfig。（这里其实有很多内容待会再分析）
Configupdate/configupdate.go

```go
func (p *Processor) Process(envConfigUpdate *cb.Envelope) (*cb.Envelope, error) {
    channelID, err := channelID(envConfigUpdate)
    if err != nil {
        return nil, err
    }

    support, ok := p.manager.GetChain(channelID)
    if ok {
        logger.Debugf("Processing channel reconfiguration request for channel %s", channelID)
        return p.existingChannelConfig(envConfigUpdate, channelID, support)
    }

    logger.Debugf("Processing channel creation request for channel %s", channelID)
    return p.newChannelConfig(channelID, envConfigUpdate)
}
```
4.通过**GetChain**获得**System channel**的**ChainSupport**。(现在这个chain还不存在)

```go

support, ok := bh.sm.GetChain(chdr.ChannelId)
if !ok {
      logger.Warningf("Rejecting broadcast because channel %s was not found", chdr.ChannelId)
      return srv.Send(&ab.BroadcastResponse{Status: cb.Status_NOT_FOUND})
 }
```
5.**support.Filters().Apply(msg)**，其中filter由一个名为**systemChainFilter**(mutichain/systemchain.go) 负责，其功能为生成一个新的chain（这个时候还暂时不会生成真正的chain,只是会得到一个相应的**commiter**）。

```go
_, filterErr := support.Filters().Apply(msg)
if filterErr != nil {
     logger.Warningf("[channel: %s] Rejecting broadcast message because of filter error: %s", chdr.ChannelId, filterErr)
     return srv.Send(&ab.BroadcastResponse{Status: cb.Status_BAD_REQUEST})
}
```

 mutichain/systemchain.go
 
```go
func (scf *systemChainFilter) Apply(env *cb.Envelope) (filter.Action, filter.Committer) {
    msgData := &cb.Payload{}

   //msgData 的各种检查
    err := proto.Unmarshal(env.Payload, msgData)
    if err != nil {
        return filter.Forward, nil
    }
   //...

//当前是否可以创建channel的检查，（检查maxChannels）
    maxChannels := scf.support.SharedConfig().MaxChannelsCount()
     //...

     //验证是否有权限操作，主要涉及签名的验证等
     err = scf.authorizeAndInspect(configTx)
     if err != nil {
          logger.Debugf("Rejecting channel creation because: %s", err)
          returnfilter.Reject, nil
     }
     return     filter.Accept,&systemChainCommitter{
                    filter:   scf,
                    configTx: configTx,
     }
}
```
 6.**Enqueue**，这个地方在通道里等待orderer的处理，orderer处理方式可以在相应的consensus.go文件中找到。这边orderer何时生成block的具体代码分析，参见[Hyperledger fabric 代码解析 之 Orderer Service）]({{ site.baseurl }}/fabric-orderer/)  中的2.6部分
 
```go
if !support.Enqueue(msg) {
        return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE})
 }


     consensus.go

func (ch *chain) main() {
    var timer <-chan time.Time

    for {
        select {
        case msg := <-ch.sendChan:
            batches, committers, ok := ch.support.BlockCutter().Ordered(msg)
            if ok && len(batches) == 0 && timer == nil {
                timer = time.After(ch.batchTimeout)
                continue

            }
            //将batches里的block挨个处理
           for i, batch := range batches {
                block := ch.support.CreateNextBlock(batch)
                ch.support.WriteBlock(block, committers[i], nil)
            }
            if len(batches) > 0 {
                timer = nil
            }

     //超时处理
        case <-timer:
            //clear the timer
            timer = nil

            batch, committers := ch.support.BlockCutter().Cut()
            if len(batch) == 0 {
                logger.Warningf("Batch timer expired with no pending requests, this might indicate a bug")
                continue
            }
            logger.Debugf("Batch timer expired, creating block")
            block := ch.support.CreateNextBlock(batch)
            ch.support.WriteBlock(block, committers, nil)
        case <-ch.exitChan:
            logger.Debugf("Exiting")
            return
        }
    }
}
```
7.**WriteBlock**,等待orderer这边满足写块的条件。这个地方第五步**filter.commiter**返回的systemChainCommitter的**commit**，这个地方才会去调用**newChain**，才是真正的创建channel。
muitichain/chainsupport.go

```go
func (cs *chainSupport) WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block {
        for _, committer := range committers {
                committer.Commit()
        }
        // Set the orderer-related metadata field
        if encodedMetadataValue != nil {
                block.Metadata.Metadata[cb.BlockMetadataIndex_ORDERER] = utils.MarshalOrPanic(&cb.Metadata{Value: encodedMetadataValue})
        }
        cs.addBlockSignature(block)
        cs.addLastConfigSignature(block)

        err := cs.ledger.Append(block)
        if err != nil {
                logger.Panicf("[channel: %s] Could not append block: %s", cs.ChainID(), err)
        }
        logger.Debugf("[channel: %s] Wrote block %d", cs.ChainID(), block.GetHeader().Number)

        return block
}
```

multichain/systemchain.go
```go
func (scc *systemChainCommitter) Commit() {
        logger.Warningf("Commit channel creation in systemCommitter")
        scc.filter.cc.newChain(scc.configTx)

}
```

multichain/manager.go
```go
func (ml *multiLedger) newChain(configtx *cb.Envelope) {
        ledgerResources := ml.newLedgerResources(configtx)
        ledgerResources.ledger.Append(ledger.CreateNextBlock(ledgerResources.ledger, []*cb.Envelope{configtx}))

        // Copy the map to allow concurrent reads from broadcast/deliver while the new chainSupport is
        newChains := make(map[string]*chainSupport)
        for key, value := range ml.chains {
                newChains[key] = value
        }

        cs := newChainSupport(createStandardFilters(ledgerResources), ledgerResources, ml.consenters, ml.signer)
        chainID := ledgerResources.ChainID()

        logger.Infof("Created and starting new chain %s", chainID)

        newChains[string(chainID)] = cs
        cs.start()

        ml.chains = newChains
}
```
8.**Send**
