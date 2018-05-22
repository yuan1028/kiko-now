---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 channel join
tags:
  - Fabric
---
![channeljoin.png]({{ site.baseurl }}/images/channeljoin.png) 
1.sdk或client发起*join channel*请求，然后开始构建一个**Proposal**包体。
主要过程是通过**getJoinCCSpec**得到**ChaincodeInvocationSpec**（主要是指定chaincode为cscc，将block文件打包到input中），然后通过**CreateProposalFromCIS**以及**GetSignedProposal** 继续打包，主要是得到一个transcation id（txid）加上各种包头等。注意这里打包最后得到的Proposal请求理的channel id是““。

2.通过EndorserClient发起请求，调用endorser（通常是某个或某些peer）的**ProcessProposal**函数，处理第一步得到的Proposal包体。ProcessProposal中会调用simulateProposal对chaincode进行模拟执行。模拟执行过程中是通过一套状态机，通过消息变化在对transcation进行处理的时候，调用beforeTranscation，然后最终达到调用cscc的joinChain函数的。ProcessProposal的详细过程见[Hyperledger fabric 1.0 代码解析 之 ProcessProposal]({{ site.baseurl }}/fabric-process-proposal/)

3.cscc端首先获得签名的Proposal，并进行policy的验证。验证成功后调用joinchain函数。
core/scc/cscc/configure.go
```
func (e *PeerConfiger) Invoke(stub shim.ChaincodeStubInterface) pb.Response {

	/*忽略Args参数个数检查部分*/

	cnflogger.Debugf("Invoke function: %s", fname)
	// Handle ACL:
	// 1. get the signed proposal
	sp, err := stub.GetSignedProposal()
	if err != nil {
		return shim.Error(fmt.Sprintf("Failed getting signed proposal from stub: [%s]", err))
	}

	switch fname {
	case JoinChain:
		/*以下代码中的错误处理暂时都忽略了*/
		block, err := utils.GetBlockFromBlockBytes(args[1])
		cid, err := utils.GetChainIDFromBlock(block)
		if err := validateConfigBlock(block); err != nil {
			/**/
		}

		// 2. check local MSP Admins policy
		if err = e.policyChecker.CheckPolicyNoChannel(mgmt.Admins, sp); err != nil {
			/**/
		}

		return joinChain(cid, block)
		/*忽略掉其他case情况*/
	}
}

```
4.根据发起请求自带的block参数，创建相应的chain信息，包括chain对应的ledger，并将该block数据提交到该ledger中。
core/scc/cscc/configure.go
```
func joinChain(chainID string, block *common.Block) pb.Response {
	if err := peer.CreateChainFromBlock(block); err != nil {
		return shim.Error(err.Error())
	}

	peer.InitChain(chainID)

	if err := producer.SendProducerBlockEvent(block); err != nil {
		cnflogger.Errorf("Error sending block event %s", err)
	}

	return shim.Success(nil)
}
```

```
// CreateChainFromBlock creates a new chain from config block
func CreateChainFromBlock(cb *common.Block) error {
	cid, err := utils.GetChainIDFromBlock(cb)
	if err != nil {
		return err
	}

	var l ledger.PeerLedger
	if l, err = ledgermgmt.CreateLedger(cb); err != nil {
		return fmt.Errorf("Cannot create ledger from genesis block, due to %s", err)
	}

	return createChain(cid, l, cb)
}
```
5.peer将创建好的chain信息同步到相同组织其它的peer中。并与orderer通信，并建立与orderer之间在该channerl上的数据传输通道。
```
// createChain creates a new chain object and insert it into the chains
func createChain(cid string, ledger ledger.PeerLedger, cb *common.Block) error {

	envelopeConfig, err := utils.ExtractEnvelope(cb, 0)
	if err != nil {
		return err
	}

	configtxInitializer := configtx.NewInitializer()

	gossipEventer := service.GetGossipService().NewConfigEventer()

	gossipCallbackWrapper := func(cm configtxapi.Manager) {
		ac, ok := configtxInitializer.ApplicationConfig()
		if !ok {
			// TODO, handle a missing ApplicationConfig more gracefully
			ac = nil
		}
		gossipEventer.ProcessConfigUpdate(&chainSupport{
			Manager:     cm,
			Application: ac,
		})
		service.GetGossipService().SuspectPeers(func(identity api.PeerIdentityType) bool {
			// TODO: this is a place-holder that would somehow make the MSP layer suspect
			// that a given certificate is revoked, or its intermediate CA is revoked.
			// In the meantime, before we have such an ability, we return true in order
			// to suspect ALL identities in order to validate all of them.
			return true
		})
	}

	trustedRootsCallbackWrapper := func(cm configtxapi.Manager) {
		updateTrustedRoots(cm)
	}

	configtxManager, err := configtx.NewManagerImpl(
		envelopeConfig,
		configtxInitializer,
		[]func(cm configtxapi.Manager){gossipCallbackWrapper, trustedRootsCallbackWrapper},
	)
	if err != nil {
		return err
	}

	// TODO remove once all references to mspmgmt are gone from peer code
	mspmgmt.XXXSetMSPManager(cid, configtxManager.MSPManager())

	ac, ok := configtxInitializer.ApplicationConfig()
	if !ok {
		ac = nil
	}
	cs := &chainSupport{
		Manager:     configtxManager,
		Application: ac, // TODO, refactor as this is accessible through Manager
		ledger:      ledger,
	}

	c := committer.NewLedgerCommitterReactive(ledger, txvalidator.NewTxValidator(cs), func(block *common.Block) error {
		chainID, err := utils.GetChainIDFromBlock(block)
		if err != nil {
			return err
		}
		return SetCurrConfigBlock(block, chainID)
	})

	ordererAddresses := configtxManager.ChannelConfig().OrdererAddresses()
	if len(ordererAddresses) == 0 {
		return errors.New("No ordering service endpoint provided in configuration block")
	}
	service.GetGossipService().InitializeChannel(cs.ChainID(), c, ordererAddresses)

	chains.Lock()
	defer chains.Unlock()
	chains.list[cid] = &chain{
		cs:        cs,
		cb:        cb,
		committer: c,
	}

	
	var vi *pp.VariableInspector
	vi = pp.GetInstance()
	vi.Register("chains",chains)
	return nil
}
```

6.返回创建chain创建成果通知消息。该部分从代码层面会回到core/chaincode/shim/chaincode.go的handleTranscation中触发Transcation的nextState即COMPELETED。状态发生变化后会执行chatWithPeer以及processStream的相应函数。最终会调用notify函数。通知加入channel完成。
core/chaincode/handler.go
```
func (handler *Handler) HandleMessage(msg *pb.ChaincodeMessage) error {
	chaincodeLogger.Debugf("[%s]Fabric side Handling ChaincodeMessage of type: %s in state %s", shorttxid(msg.Txid), msg.Type, handler.FSM.Current())

	if (msg.Type == pb.ChaincodeMessage_COMPLETED || msg.Type == pb.ChaincodeMessage_ERROR) && handler.FSM.Current() == "ready" {
		chaincodeLogger.Debugf("[%s]HandleMessage- COMPLETED. Notify", msg.Txid)
		handler.notify(msg)
		return nil
	}
}
```
