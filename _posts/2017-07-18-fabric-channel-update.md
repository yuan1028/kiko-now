---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 channel update
tags:
  - Fabric
---
![channel update 流程图]({{ site.baseurl }}/images/channelupdate.png)
## client端逻辑代码     
"github.com/hyperledger/fabric/peer/channel/update.go"

1. client发起*channel update –c channel_name -f tx_path*请求，其中参数channel_name为channel的名称,tx_path为需要更新的tx文件的路径。目前channel的
update操作只支持从tx文件更新。
2. 接收到update请求后，从**ConfigTx**文件中（**ReadFile**）生成包体Envelope，并对生成的包体Envelope进行校验与签名，通过**broadcast**接口向orderer发送update channel消息即签过名的包体Envelope。

## orderer端逻辑代码   
"github.com/hyperledger/fabric/orderer"
**该部分有很多和channel create的内容是重复的，主要差别在第4步及之后的内容。**

1. orderer端对于broadcast、deliver和join分别有3个对应的handler来处理（server.go）.Handle为broadcast主处理函数(broadcast/broadcast.go)，负责peer消息的处理，包括channel的创建以及其他类型的消息，其中channel通过类型HeaderType_CONFIG_UPDATE进行区分。
2. 若channelHeaderType是HeaderType_CONFIG_UPDATE，则为create channel或者是update channel的操作。此时调用process函数来获得新建或者更新的包体Envelope.

```go
if chdr.Type == int32(cb.HeaderType_CONFIG_UPDATE) {
            logger.Debugf("Preprocessing CONFIG_UPDATE")
            msg, err = bh.sm.Process(msg)
……
}
```
3. Process（Configupdate/configupdate.go）以channel相关信息生成Envelope消息，并将该Envelope消息重新打包到以TYPE: HeaderType_CONFIG_UPDATE为类型的消息体中。函数中通过判断包体传递过来的chainID是否已存在来判断是create还是update操作。chainID存在，则调用existingChannelConfig函数。
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

4. 通过**GetChain**获得**该channel**的**ChainSupport**。(在channel create的时候这个地方是system chain)

```go
support, ok := bh.sm.GetChain(chdr.ChannelId)
if !ok {
      logger.Warningf("Rejecting broadcast because channel %s was not found", chdr.ChannelId)
      return srv.Send(&ab.BroadcastResponse{Status: cb.Status_NOT_FOUND})
 }
```

5. support.Filters().Apply(msg)，该处的filter为该channel的filter。

```go
_, filterErr := support.Filters().Apply(msg)
if filterErr != nil {
     logger.Warningf("[channel: %s] Rejecting broadcast message because of filter error: %s", chdr.ChannelId, filterErr)
     return srv.Send(&ab.BroadcastResponse{Status: cb.Status_BAD_REQUEST})
}
```
以下通过代码看下普通channel的Filters与SystemChainFilters的区别。普通channel是没有SystemChainFilter的。

```go
// createStandardFilters creates the set of filters for a normal (non-system) chain
func createStandardFilters(ledgerResources *ledgerResources) *filter.RuleSet {
    return filter.NewRuleSet([]filter.Rule{
        filter.EmptyRejectRule,
        sizefilter.MaxBytesRule(ledgerResources.SharedConfig().BatchSize().AbsoluteMaxBytes),
        sigfilter.New(policies.ChannelWriters, ledgerResources.PolicyManager()),
        configtxfilter.NewFilter(ledgerResources),
        filter.AcceptRule,
    })

}
// createSystemChainFilters creates the set of filters for the ordering system chain
func createSystemChainFilters(ml *multiLedger, ledgerResources *ledgerResources) *filter.RuleSet {
    return filter.NewRuleSet([]filter.Rule{
        filter.EmptyRejectRule,
        sizefilter.MaxBytesRule(ledgerResources.SharedConfig().BatchSize().AbsoluteMaxBytes),
        sigfilter.New(policies.ChannelWriters, ledgerResources.PolicyManager()),
        newSystemChainFilter(ledgerResources, ml),
        configtxfilter.NewFilter(ledgerResources),
        filter.AcceptRule,
    })
}
```
6. Enqueue，这个地方在通道里等待orderer的处理，orderer处理方式可以在相应的consensus.go文件中找到。这边orderer何时生成block的具体代码分析，参见[Hyperledger fabric 代码解析 之 Orderer Service]({{ site.baseurl }}/fabric-orderer/)中的2.6部分

```go
if !support.Enqueue(msg) {
        return srv.Send(&ab.BroadcastResponse{Status: cb.Status_SERVICE_UNAVAILABLE})
 }
```

consensus.go

```go
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
7. WriteBlock,等待orderer这边满足写块的条件。这个地方会调用第五步filter.commiter返回的Committer的commit。
 muitichain/chainsupport.go

```go
func (cs *chainSupport) WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block {
        logger.Debugf("[hzyangwenlong] this is WriteBlock.....in and the commiters is %v", committers)
        //之前filter返回的committer到这一步才真正的commit
        for _, committer := range committers {
                committer.Commit()
        }
        // Set the orderer-related metadata field
        if encodedMetadataValue != nil {
                block.Metadata.Metadata[cb.BlockMetadataIndex_ORDERER] = utils.MarshalOrPanic(&cb.Metadata{Value: encodedMetadataValue})
        }
        cs.addBlockSignature(block)
        //configGroup的那些
        cs.addLastConfigSignature(block)

        err := cs.ledger.Append(block)
        if err != nil {
                logger.Panicf("[channel: %s] Could not append block: %s", cs.ChainID(), err)
        }
        logger.Debugf("[channel: %s] Wrote block %d", cs.ChainID(), block.GetHeader().Number)

        return block
}
```

```go
func (cc *configCommitter) Commit() {
    err := cc.manager.Apply(cc.configEnvelope)
    if err != nil {
        panic(fmt.Errorf("Could not apply config transaction which should have already been validated: %s", err))
    }
}
```

```go
// Apply attempts to apply a ConfigEnvelope to become the new config
func (cm *configManager) Apply(configEnv *cb.ConfigEnvelope) error {
    //prepareApply进行一堆的验证
    configMap, result, err := cm.prepareApply(configEnv)
    if err != nil {
        return err
    }

    result.commit()

    cm.current = &configSet{
        configMap: configMap,
        channelID: cm.current.channelID,
        sequence:  configEnv.Config.Sequence,
        configEnv: configEnv,
    }

    cm.commitCallbacks()

    return nil
}
```
8. Send
