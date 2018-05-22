---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 chaincode install
tags:
  - Fabric
---
1.sdk或client发起install chaincode请求，然后开始构建一个Proposal包体。
主要过程是通过**createProposalFromCDS**将**chainId设置为"",将ChaincodeID的Name指定为lscc**，然后通过CreateProposalFromCIS以及GetSignedProposal 继续打包，主要是得到一个transcation id（txid）加上各种包头等。

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
        /*暂时忽略相应情况内容，待之后相应部分分析*/
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
2.通过EndorserClient发起请求，调用endorser（通常是某个或某些peer）的**ProcessProposal**函数，处理第一步得到的Proposal包体。从[Hyperledger fabric 1.0 代码解析 之 ProcessProposal]({{site.baseurl}}/fabric-process-proposal/) 最开始的流程图，可以看出，install chaincode的处理过程只需要调用simulateProposal,且因为lscc是systemCC,**该次ProcessProposal最终只会调用一次ExecuteChaincode,调用的是lscc中install相关内容**。

3.lscc端中的Invoke函数，处理install的情况，首先检查签名，及是否有install chaincode的相关权限。然后执行**executeInstall**来进行install chaincode的实际操作.

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
        if len(args) < 2 {
            return shim.Error(InvalidArgsLenErr(len(args)).Error())
        }

        // 2. 权限检查，查看是否有install chaincode的权限，检查签名证书是否有Admins权限
        if err = lscc.policyChecker.CheckPolicyNoChannel(mgmt.Admins, sp); err != nil {
            return shim.Error(fmt.Sprintf("Authorization for INSTALL on %s has been denied with error %s", args[1], err))
        }

        depSpec := args[1]
        //executeInstall
        err := lscc.executeInstall(stub, depSpec)
        if err != nil {
            return shim.Error(err.Error())
        }
        return shim.Success([]byte("OK"))
    case DEPLOY:
            /*暂时忽略相应情况内容，待之后相应部分分析*/
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
4.**executeInstall**首先获取**ccpack**(一会用来打包chaincode)，并对chaincodeName,chaincodeVersion进行检查，随后调用**PutChaincodeToFS**将chaincode写入文件系统。

```go
// executeInstall implements the "install" Invoke transaction
func (lscc *LifeCycleSysCC) executeInstall(stub shim.ChaincodeStubInterface, ccbytes []byte) error {
    ccpack, err := ccprovider.GetCCPackage(ccbytes)
    if err != nil {
        return err
    }

    cds := ccpack.GetDepSpec()

    if cds == nil {
        return fmt.Errorf("nil deployment spec from from the CC package")
    }

    if err = lscc.isValidChaincodeName(cds.ChaincodeSpec.ChaincodeId.Name); err != nil {
        return err
    }

    if err = lscc.isValidChaincodeVersion(cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version); err != nil {
        return err
    }

    //everything checks out..lets write the package to the FS
    if err = ccpack.PutChaincodeToFS(); err != nil {
        return fmt.Errorf("Error installing chaincode code %s:%s(%s)", cds.ChaincodeSpec.ChaincodeId.Name, cds.ChaincodeSpec.ChaincodeId.Version, err)
    }

    return err
}

```

```go
//PutChaincodeToFS - serializes chaincode to a package on the file system
func (ccpack *CDSPackage) PutChaincodeToFS() error {
   /*各种为空的时候的错误处理，暂时忽略*/
    ccname := ccpack.depSpec.ChaincodeSpec.ChaincodeId.Name
    ccversion := ccpack.depSpec.ChaincodeSpec.ChaincodeId.Version

    //return error if chaincode exists
    path := fmt.Sprintf("%s/%s.%s", chaincodeInstallPath, ccname, ccversion)
    if _, err := os.Stat(path); err == nil {
        return fmt.Errorf("chaincode %s exists", path)
    }
   //写入文件
    if err := ioutil.WriteFile(path, ccpack.buf, 0644); err != nil {
        return err
    }

    return nil
}
```