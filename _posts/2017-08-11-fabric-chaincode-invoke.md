---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 chaincode invoke or query
tags:
  - Fabric
---
1.sdk或client发起chaincode invoke／query 请求，然后开始构建一个Proposal包体。
主要过程是通过**getChaincodeSpec**,**设置为chainID和chaincodeName**，然后通过CreateProposalFromCIS以及GetSignedProposal 继续打包，主要是得到一个transcation id（txid）加上各种包头等。

```go
// getChaincodeSpec get chaincode spec from the cli cmd pramameters
func getChaincodeSpec(cmd *cobra.Command) (*pb.ChaincodeSpec, error) {
    spec := &pb.ChaincodeSpec{}
    if err := checkChaincodeCmdParams(cmd); err != nil {
        return spec, err
    }

    // Build the spec
    input := &pb.ChaincodeInput{}
    if err := json.Unmarshal([]byte(chaincodeCtorJSON), &input); err != nil {
        return spec, fmt.Errorf("Chaincode argument error: %s", err)
    }

    chaincodeLang = strings.ToUpper(chaincodeLang)
    if pb.ChaincodeSpec_Type_value[chaincodeLang] == int32(pb.ChaincodeSpec_JAVA) {
        return nil, fmt.Errorf("Java chaincode is work-in-progress and disabled")
    }
    spec = &pb.ChaincodeSpec{
        Type:        pb.ChaincodeSpec_Type(pb.ChaincodeSpec_Type_value[chaincodeLang]),
        ChaincodeId: &pb.ChaincodeID{Path: chaincodePath, Name: chaincodeName, Version: chaincodeVersion},
        Input:       input,
    }
    return spec, nil
}
```
2.通过EndorserClient发起请求，调用endorser（通常是某个或某些peer）的**ProcessProposal**函数，处理第一步得到的Proposal包体。从[Hyperledger fabric 1.0 代码解析 之 ProcessProposal]({{site.baseurl}}/fabric-process-proposal) 最开始的流程图，可以看出，chaincode invoke和query首先需要调用**simulateProposal**和**endorseProposal**来完成整个流程。

3.simulateProposal过程中大多数情况chaincode不是system chaincode,这个时候需要首先调用GetChaincodeFromLSCC,从lscc获取到chaincode data。（所以这里其实会先有一个执行lscc chaincode的过程）

4.调用callChaincode,最终到chaincode的Invoke函数。该处Invoke函数一般为用户的Invoke函数，在此不作分析。

5.调用endoserProposal对结果进行背书。

6.如果是invoke操作，这里会涉及到之后与orderer通信，orderer创建block，再分发给peer的过程。

7.peer从orderer或者通过gossip协议从其他节点拿到区块。然后验证通过后写区块，更新数据库。该部分内容见[hyperledger fabric 1.0 代码解析 之 Commit Block]({{ site.baseurl }}/fabric-commit-block/)

```go
// DeliverBlocks used to pull out blocks from the ordering service to
// distributed them across peers
func (b *blocksProviderImpl) DeliverBlocks() {
    errorStatusCounter := 0
    statusCounter := 0
    defer b.client.Close()
    for !b.isDone() {
        msg, err := b.client.Recv()
        if err != nil {
            logger.Warningf("[%s] Receive error: %s", b.chainID, err.Error())
            return
        }
        switch t := msg.Type.(type) {
        case *orderer.DeliverResponse_Status:
            //...
        case *orderer.DeliverResponse_Block:
        //从orderer节点获得了block区块
            errorStatusCounter = 0
            statusCounter = 0
            seqNum := t.Block.Header.Number

            marshaledBlock, err := proto.Marshal(t.Block)
            if err != nil {
                logger.Errorf("[%s] Error serializing block with sequence number %d, due to %s", b.chainID, seqNum, err)
                continue
            }
            //
            if err := b.mcs.VerifyBlock(gossipcommon.ChainID(b.chainID), seqNum, marshaledBlock); err != nil {
                logger.Errorf("[%s] Error verifying block with sequnce number %d, due to %s", b.chainID, seqNum, err)
                continue
            }

            numberOfPeers := len(b.gossip.PeersOfChannel(gossipcommon.ChainID(b.chainID)))
            // Create payload with a block received
            payload := createPayload(seqNum, marshaledBlock)
            // Use payload to create gossip message
            gossipMsg := createGossipMsg(b.chainID, payload)

            logger.Debugf("[%s] Adding payload locally, buffer seqNum = [%d], peers number [%d]", b.chainID, seqNum, numberOfPeers)
            // Add payload to local state payloads buffer
            b.gossip.AddPayload(b.chainID, payload)

            // Gossip messages with other nodes
            logger.Debugf("[%s] Gossiping block [%d], peers number [%d]", b.chainID, seqNum, numberOfPeers)
            b.gossip.Gossip(gossipMsg)
        default:
            logger.Warningf("[%s] Received unknown: ", b.chainID, t)
            return
        }
    }
}
```

通过gossip从其他peer节点处获得区块
```
// validateMsg checks the signature of the message if exists,
// and also checks that the tag matches the message type
func (g *gossipServiceImpl) validateMsg(msg proto.ReceivedMessage) bool {
    if err := msg.GetGossipMessage().IsTagLegal(); err != nil {
        g.logger.Warning("Tag of", msg.GetGossipMessage(), "isn't legal:", err)
        return false
    }

    //...
    if msg.GetGossipMessage().IsDataMsg() {
        blockMsg := msg.GetGossipMessage().GetDataMsg()
        if blockMsg.Payload == nil {
            g.logger.Warning("Empty block! Discarding it")
            return false
        }

        // If we're configured to skip block validation, don't verify it
        if g.conf.SkipBlockVerification {
            return true
        }

        //拿到的是区块的消息
        seqNum := blockMsg.Payload.SeqNum
        rawBlock := blockMsg.Payload.Data
        if err := g.mcs.VerifyBlock(msg.GetGossipMessage().Channel, seqNum, rawBlock); err != nil {
            g.logger.Warning("Could not verify block", blockMsg.Payload.SeqNum, ":", err)
            return false
        }
    }

    // ...
    return true
}
```