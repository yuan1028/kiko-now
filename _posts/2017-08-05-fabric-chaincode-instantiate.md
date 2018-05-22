---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 chaincode instantiate or upgrade
tags:
  - Fabric
---
1.sdk或client发起chaincode instantiate/upgrade 请求，然后开始构建一个Proposal包体。
主要过程是通过**createProposalFromCDS**,**设置为chainID,将ChaincodeID的Name指定为lscc**，然后通过CreateProposalFromCIS以及GetSignedProposal 继续打包，主要是得到一个transcation id（txid）加上各种包头等。

```go
// createProposalFromCDS returns a deploy or upgrade proposal given a serialized identity and a ChaincodeDeploymentSpec
func createProposalFromCDS(chainID string, msg proto.Message, creator []byte, policy []byte, escc []byte, vscc []byte, propType string) (*peer.Proposal, string, error) {
    //in the new mode, cds will be nil, "deploy" and "upgrade" are instantiates.
    var ccinp *peer.ChaincodeInput
    var b []byte
    var err error
    if msg != nil {
        b, err = proto.Marshal(msg)
        if err != nil {
            return nil, "", err
        }
    }
    switch propType {
    case "deploy":
        fallthrough
    case "upgrade":
        cds, ok := msg.(*peer.ChaincodeDeploymentSpec)
        if !ok || cds == nil {
            return nil, "", fmt.Errorf("invalid message for creating lifecycle chaincode proposal from")
        }
        //这里在deploy和upgrade的时候policy,escc,vscc都来自调用时的参数
        ccinp = &peer.ChaincodeInput{Args: [][]byte{[]byte(propType), []byte(chainID), b, policy, escc, vscc}}
    case "install":
        ccinp = &peer.ChaincodeInput{Args: [][]byte{[]byte(propType), b}}
    }

    //wrap the deployment in an invocation spec to lscc...
    lsccSpec := &peer.ChaincodeInvocationSpec{
        ChaincodeSpec: &peer.ChaincodeSpec{
            Type:        peer.ChaincodeSpec_GOLANG,
            ChaincodeId: &peer.ChaincodeID{Name: "lscc"},
            Input:       ccinp}}

    //...and get the proposal for it
    return CreateProposalFromCIS(common.HeaderType_ENDORSER_TRANSACTION, chainID, lsccSpec, creator)
}
```
2.通过EndorserClient发起请求，调用endorser（通常是某个或某些peer）的**ProcessProposal**函数，处理第一步得到的Proposal包体。从[Hyperledger fabric 1.0 代码解析 之 ProcessProposal]({{site.baseurl}}/fabric-process-proposal/) 最开始的流程图，可以看出，chaincode instantiate首先需要调用**simulateProposal**和**endorseProposal**来完成整个流程。

3.simulateProposal过程中会最终通过ExecuteChaincode,执行到lscc端中的Invoke函数，处理deploy的情况。首先检查签名，然后在deploy的case中获取到policy,escc,vscc相关信息，然后传递给**executeDeploy or executeUpgrade**来进行chaincode的实际deploy或upgrade操作.

```go
func (lscc *LifeCycleSysCC) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
    args := stub.GetArgs()
    if len(args) < 1 {
        return shim.Error(InvalidArgsLenErr(len(args)).Error())
    }

    function := string(args[0])

    // Handle ACL:
    // 1. get the signed proposal
    sp, err := stub.GetSignedProposal()
    if err != nil {
        return shim.Error(fmt.Sprintf("Failed retrieving signed proposal on executing %s with error %s", function, err))
    }

    switch function {
    case INSTALL:
        /*另一篇中已有分析，这里忽略*/
    case DEPLOY:
            if len(args) < 3 || len(args) > 6 {
            return shim.Error(InvalidArgsLenErr(len(args)).Error())
        }
        chainname := string(args[1])

        if !lscc.isValidChainName(chainname) {
            return shim.Error(InvalidChainNameErr(chainname).Error())
        }

        depSpec := args[2]

        // optional arguments here (they can each be nil and may or may not be present)
        // args[3] is a marshalled SignaturePolicyEnvelope representing the endorsement policy
        // args[4] is the name of escc
        // args[5] is the name of vscc
        var policy []byte
        if len(args) > 3 && len(args[3]) > 0 {
            policy = args[3]
        } else {
            //默认的policy是channel中任何一个成员背书即可
            p := cauthdsl.SignedByAnyMember(peer.GetMSPIDs(chainname))
            policy, err = utils.Marshal(p)
            if err != nil {
                return shim.Error(err.Error())
            }
        }

        var escc []byte
        if len(args) > 4 && args[4] != nil {
            escc = args[4]
        } else {
            escc = []byte("escc")
        }

        var vscc []byte
        if len(args) > 5 && args[5] != nil {
            vscc = args[5]
        } else {
            vscc = []byte("vscc")
        }

        cd, err := lscc.executeDeploy(stub, chainname, depSpec, policy, escc, vscc)
        if err != nil {
            return shim.Error(err.Error())
        }
        cdbytes, err := proto.Marshal(cd)
        if err != nil {
            return shim.Error(err.Error())
        }
        return shim.Success(cdbytes)
    case UPGRADE:
            /*暂时忽略相应情况内容，待之后相应部分分析*/
    case GETCCINFO, GETDEPSPEC, GETCCDATA:
            /*暂时忽略相应情况内容，待之后相应部分分析*/
    case GETCHAINCODES:
            /*暂时忽略相应情况内容，待之后相应部分分析*/
    case GETINSTALLEDCHAINCODES:
           /*暂时忽略相应情况内容，待之后相应部分分析*/
    return shim.Error(InvalidFunctionErr(function).Error())
}

```
4.**executeDeploy**首先进行各项检查，然后从文件系统中获取chaincode,并对是否具有Instantiation权限进行检查。最后调用createChaincode。createChaincode主要执行的是将chaincode put state到lscc.

```go
// executeDeploy implements the "instantiate" Invoke transaction
func (lscc *LifeCycleSysCC) executeDeploy(stub shim.ChaincodeStubInterface, chainname string, depSpec []byte, policy []byte, escc []byte, vscc []byte) (*ccprovider.ChaincodeData, error) {
    cds, err := utils.GetChaincodeDeploymentSpec(depSpec)

    /*各种检查，忽略*/

    //从文件系统中获取chaincode
    ccpack, err := ccprovider.GetChaincodeFromFS(cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version)
    if err != nil {
        return nil, fmt.Errorf("cannot get package for the chaincode to be instantiated (%s:%s)-%s", cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version, err)
    }

    //this is guarantees to be not nil
    cd := ccpack.GetChaincodeData()

    //retain chaincode specific data and fill channel specific ones
    cd.Escc = string(escc)
    cd.Vscc = string(vscc)
    cd.Policy = policy

    // retrieve and evaluate instantiation policy
    cd.InstantiationPolicy, err = lscc.getInstantiationPolicy(chainname, ccpack)
    if err != nil {
        return nil, err
    }
    err = lscc.checkInstantiationPolicy(stub, chainname, cd.InstantiationPolicy)
    if err != nil {
        return nil, err
    }

    err = lscc.createChaincode(stub, cd)

    return cd, err
}
```
5.第4步lscc的executeDeploy执行完毕后，simulateProposal的callChaincode中的第一个executeChaincode完成。对于deploy这种比较特殊的情况,在callChaincode中还需要调用chaincode.Execute来完成chaincode的真正部署(启动了对应的docker 容器，完成了chaincode的Init操作)。其中启动相应docker 容器部分，参考[Hyperledger fabric 1.0 代码解析 之 chaincode Launch]({{site.baseurl}}/fabric-chaincode-launch)

6.**endorseProposal**对整个结果进行背书。整个过程依然是一个callChaincode的过程，最终调用到escc的Invoke函数。Invoke函数中主要是对各个参数进行检查，并使用peer的身份进行签名。

core/scc/escc/endorser_onevalidsignature.go。

```go
func (e *EndorserOneValidSignature) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
    args := stub.GetArgs()
    if len(args) < 6 {
        return shim.Error(fmt.Sprintf("Incorrect number of arguments (expected a minimum of 5, provided %d)", len(args)))
    } else if len(args) > 8 {
        return shim.Error(fmt.Sprintf("Incorrect number of arguments (expected a maximum of 7, provided %d)", len(args)))
    }

    logger.Debugf("ESCC starts: %d args", len(args))

    // handle the header
    var hdr []byte
    if args[1] == nil {
        return shim.Error("serialized Header object is null")
    }

    hdr = args[1]

    // handle the proposal payload
    var payl []byte
    if args[2] == nil {
        return shim.Error("serialized ChaincodeProposalPayload object is null")
    }

    payl = args[2]

    // handle ChaincodeID
    if args[3] == nil {
        return shim.Error("ChaincodeID is null")
    }

    ccid, err := putils.UnmarshalChaincodeID(args[3])
    if err != nil {
        return shim.Error(err.Error())
    }

    // handle executing chaincode result
    // Status code < shim.ERRORTHRESHOLD can be endorsed
    if args[4] == nil {
        return shim.Error("Response of chaincode executing is null")
    }

    response, err := putils.GetResponse(args[4])
    if err != nil {
        return shim.Error(fmt.Sprintf("Failed to get Response of executing chaincode: %s", err.Error()))
    }

    if response.Status >= shim.ERRORTHRESHOLD {
        return shim.Error(fmt.Sprintf("Status code less than %d will be endorsed, received status code: %d", shim.ERRORTHRESHOLD, response.Status))
    }

    // handle simulation results
    var results []byte
    if args[5] == nil {
        return shim.Error("simulation results are null")
    }

    results = args[5]

    // Handle serialized events if they have been provided
    // they might be nil in case there's no events but there
    // is a visibility field specified as the next arg
    events := []byte("")
    if len(args) > 6 && args[6] != nil {
        events = args[6]
    }

    var visibility []byte
    if len(args) > 7 {
        visibility = args[7]
    }

    // obtain the default signing identity for this peer; it will be used to sign this proposal response
    localMsp := mspmgmt.GetLocalMSP()
    if localMsp == nil {
        return shim.Error("Nil local MSP manager")
    }

    signingEndorser, err := localMsp.GetDefaultSigningIdentity()
    if err != nil {
        return shim.Error(fmt.Sprintf("Could not obtain the default signing identity, err %s", err))
    }

    // obtain a proposal response
    presp, err := utils.CreateProposalResponse(hdr, payl, response, results, events, ccid, visibility, signingEndorser)
    if err != nil {
        return shim.Error(err.Error())
    }

    // marshall the proposal response so that we return its bytes
    prBytes, err := utils.GetBytesProposalResponse(presp)
    if err != nil {
        return shim.Error(fmt.Sprintf("Could not marshal ProposalResponse: err %s", err))
    }

    pResp, err := utils.GetProposalResponse(prBytes)
    if err != nil {
        return shim.Error(err.Error())
    }
    if pResp.Response == nil {
        fmt.Println("GetProposalResponse get empty Response")
    }

    logger.Debugf("ESCC exits successfully")
    return shim.Success(prBytes)
}

```
7.sdk或者client拿到结果后，还会与orderer交互，最后写block等，这部分回头再补充。