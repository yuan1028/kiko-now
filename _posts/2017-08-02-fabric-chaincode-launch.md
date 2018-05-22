---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 chaincode launch
tags:
  - Fabric
---
### Launch chaincode整体流程
智能合约(chaincode)的执行一般是在docker中进行的，所以在执行之前，需要先完成将chaincode打包到docker容器中的过程。整个过程在core/chaincode/chaincode_support.go 的Launch函数中完成。
1. 已经Launch的情况。对于正在运行（running）的则直接返回。
2. 得到cds(ChaincodeDeploymentSpec),对于deploy情况在函数开始的地方便可以获得，对于非deploy情况下(cds == nil)需要先从lscc中得到chaincode data，然后从返回中得到cds。
3. 对于系统container以及非dev模式需要lanuch容器。其它情况直接执行步骤6.（peer节点配置文件中的chaincode.mode，有两种模式分别为"dev"和"net","dev"chaincode是在本地执行，"net"模式chaincode是在docker中执行。"dev"的时候userRunsCC为true,即本地执行。执行环境exec分为有system和docker两种)
4. 若codePackage为空,对于!userRunsCC且执行环境不是ChaincodeDeploymentSpec_SYSTEM，由于是需要部署在docker上的，必须要有chaincode信息,则尝试从文件系统中读取，若失败则直接返回。
5. 创建builder(这里builder主要是读取cds的reader,之后在必要时用来创建镜像),然后交由launchAndWaitForRegister处理。该过程中会检查是否已经有镜像，没有则创建它，并启动相应的container。
6. chaincodeSupport调用sendReady,保证container进入状态机的ready态。这部分是状态机的状态变化和事件。详见core/chaincode/handler.go

```go
// Launch will launch the chaincode if not running (if running return nil) and will wait for handler of the chaincode to get into FSM ready state.
func (chaincodeSupport *ChaincodeSupport) Launch(context context.Context, cccid *ccprovider.CCContext, spec interface{}) (*pb.ChaincodeID, *pb.ChaincodeInput, error) {
    //build the chaincode
     /*得到chaincodeID和chaincodeInput以及cds...*/
    var cID *pb.ChaincodeID
    var cMsg *pb.ChaincodeInput

    var cds *pb.ChaincodeDeploymentSpec
    var ci *pb.ChaincodeInvocationSpec
    if cds, _ = spec.(*pb.ChaincodeDeploymentSpec); cds == nil {
        if ci, _ = spec.(*pb.ChaincodeInvocationSpec); ci == nil {
            panic("Launch should be called with deployment or invocation spec")
        }
    }
    if cds != nil {
        cID = cds.ChaincodeSpec.ChaincodeId
        cMsg = cds.ChaincodeSpec.Input
    } else {
        cID = ci.ChaincodeSpec.ChaincodeId
        cMsg = ci.ChaincodeSpec.Input
    }
    
    canName := cccid.GetCanonicalName()

    /*canName已经 launch的情况*/
    chaincodeSupport.runningChaincodes.Lock()
    var chrte *chaincodeRTEnv
    var ok bool
    var err error
    //if its in the map, there must be a connected stream...nothing to do
    if chrte, ok = chaincodeSupport.chaincodeHasBeenLaunched(canName); ok {
        if !chrte.handler.registered {
            chaincodeSupport.runningChaincodes.Unlock()
            chaincodeLogger.Debugf("premature execution - chaincode (%s) is being launched", canName)
            err = fmt.Errorf("premature execution - chaincode (%s) is being launched", canName)
            return cID, cMsg, err
        }
        if chrte.handler.isRunning() {
            if chaincodeLogger.IsEnabledFor(logging.DEBUG) {
                chaincodeLogger.Debugf("chaincode is running(no need to launch) : %s", canName)
            }
            chaincodeSupport.runningChaincodes.Unlock()
            return cID, cMsg, nil
        }
        chaincodeLogger.Debugf("Container not in READY state(%s)...send init/ready", chrte.handler.FSM.Current())
    }
    chaincodeSupport.runningChaincodes.Unlock()


    if cds == nil {//非deploy的情况
        if cccid.Syscc {
            return cID, cMsg, fmt.Errorf("a syscc should be running (it cannot be launched) %s", canName)
        }

        if chaincodeSupport.userRunsCC {
            chaincodeLogger.Error("You are attempting to perform an action other than Deploy on Chaincode that is not ready and you are in developer mode. Did you forget to Deploy your chaincode?")
        }

        var depPayload []byte

        //从lscc中得到chaincode data
        depPayload, err = GetCDSFromLSCC(context, cccid.TxID, cccid.SignedProposal, cccid.Proposal, cccid.ChainID, cID.Name)
        if err != nil {
            return cID, cMsg, fmt.Errorf("Could not get deployment transaction from LSCC for %s - %s", canName, err)
        }
        if depPayload == nil {
            return cID, cMsg, fmt.Errorf("failed to get deployment payload %s - %s", canName, err)
        }

        //从返回中得到ChaincodeDeploymentSpec
        cds = &pb.ChaincodeDeploymentSpec{}
        //Get lang from original deployment
        err = proto.Unmarshal(depPayload, cds)
        if err != nil {
            return cID, cMsg, fmt.Errorf("failed to unmarshal deployment transactions for %s - %s", canName, err)
        }
    }


    //launch container if it is a System container or not in dev mode
    if (!chaincodeSupport.userRunsCC || cds.ExecEnv == pb.ChaincodeDeploymentSpec_SYSTEM) && (chrte == nil || chrte.handler == nil) {
  
        if cds.CodePackage == nil {
            //no code bytes for these situations
            if !(chaincodeSupport.userRunsCC || cds.ExecEnv == pb.ChaincodeDeploymentSpec_SYSTEM) {
                //从文件系统中得到chaincode
                ccpack, err := ccprovider.GetChaincodeFromFS(cID.Name, cID.Version)
                if err != nil {
                    return cID, cMsg, err
                }

                cds = ccpack.GetDepSpec()
                chaincodeLogger.Debugf("launchAndWaitForRegister fetched %d bytes from file system", len(cds.CodePackage))
            }
        }

        builder := func() (io.Reader, error) { return platforms.GenerateDockerBuild(cds) }

        cLang := cds.ChaincodeSpec.Type
        err = chaincodeSupport.launchAndWaitForRegister(context, cccid, cds, cLang, builder)
        if err != nil {
            chaincodeLogger.Errorf("launchAndWaitForRegister failed %s", err)
            return cID, cMsg, err
        }
    }

    if err == nil {
        //launch will set the chaincode in Ready state
        err = chaincodeSupport.sendReady(context, cccid, chaincodeSupport.ccStartupTimeout)
        if err != nil {
            chaincodeLogger.Errorf("sending init failed(%s)", err)
            err = fmt.Errorf("Failed to init chaincode(%s)", err)
            errIgnore := chaincodeSupport.Stop(context, cccid, cds)
            if errIgnore != nil {
                chaincodeLogger.Errorf("stop failed %s(%s)", errIgnore, err)
            }
        }
        chaincodeLogger.Debug("sending init completed")
    }

    chaincodeLogger.Debug("LaunchChaincode complete")

    return cID, cMsg, err
}

```
### launchAndWaitForRegister流程
1. 调用getArgsAndEnv获取参数和环境，构建startImageReq(启动container的请求)。
2. 调用VMCProcess来处理startImageReq。这里是执行实际的start container的一些操作。
3. 状态机进入了register过程，等待register的完成。

```go
// launchAndWaitForRegister will launch container if not already running. Use the targz to create the image if not found
func (chaincodeSupport *ChaincodeSupport) launchAndWaitForRegister(ctxt context.Context, cccid *ccprovider.CCContext, cds *pb.ChaincodeDeploymentSpec, cLang pb.ChaincodeSpec_Type, builder api.BuildSpecFactory) error {
    canName := cccid.GetCanonicalName()
    if canName == "" {
        return fmt.Errorf("chaincode name not set")
    }

    chaincodeSupport.runningChaincodes.Lock()
    //if its in the map, its either up or being launched. Either case break the
    //multiple launch by failing
    if _, hasBeenLaunched := chaincodeSupport.chaincodeHasBeenLaunched(canName); hasBeenLaunched {
        chaincodeSupport.runningChaincodes.Unlock()
        return fmt.Errorf("Error chaincode is being launched: %s", canName)
    }

    chaincodeSupport.runningChaincodes.Unlock()

    //launch the chaincode

    //获取参数和环境
    args, env, err := chaincodeSupport.getArgsAndEnv(cccid, cLang)
    if err != nil {
        return err
    }

    chaincodeLogger.Debugf("start container: %s(networkid:%s,peerid:%s)", canName, chaincodeSupport.peerNetworkID, chaincodeSupport.peerID)
    chaincodeLogger.Debugf("start container with args: %s", strings.Join(args, " "))
    chaincodeLogger.Debugf("start container with env:\n\t%s", strings.Join(env, "\n\t"))

    //sysCC是ChaincodeDeploymentSpec_SYSTEM，其它是ChaincodeDeploymentSpec_DOCKER
    vmtype, _ := chaincodeSupport.getVMType(cds)

    /*set up the shadow handler JIT before container launch to reduce window of when an external chaincode can sneak in and use the launching context and make it its own*／
    var notfy chan bool
    preLaunchFunc := func() error {
        notfy = chaincodeSupport.preLaunchSetup(canName)
        return nil
    }

    //构建startImageReq
    sir := container.StartImageReq{CCID: ccintf.CCID{ChaincodeSpec: cds.ChaincodeSpec, NetworkID: chaincodeSupport.peerNetworkID, PeerID: chaincodeSupport.peerID, Version: cccid.Version}, Builder: builder, Args: args, Env: env, PrelaunchFunc: preLaunchFunc}

    ipcCtxt := context.WithValue(ctxt, ccintf.GetCCHandlerKey(), chaincodeSupport)

    resp, err := container.VMCProcess(ipcCtxt, vmtype, sir)
    if err != nil || (resp != nil && resp.(container.VMCResp).Err != nil) {
        if err == nil {
            err = resp.(container.VMCResp).Err
        }
        err = fmt.Errorf("Error starting container: %s", err)
        chaincodeSupport.runningChaincodes.Lock()
        delete(chaincodeSupport.runningChaincodes.chaincodeMap, canName)
        chaincodeSupport.runningChaincodes.Unlock()
        return err
    }

    //wait for REGISTER state
    select {
    case ok := <-notfy:
        if !ok {
            err = fmt.Errorf("registration failed for %s(networkid:%s,peerid:%s,tx:%s)", canName, chaincodeSupport.peerNetworkID, chaincodeSupport.peerID, cccid.TxID)
        }
    case <-time.After(chaincodeSupport.ccStartupTimeout):
        err = fmt.Errorf("Timeout expired while starting chaincode %s(networkid:%s,peerid:%s,tx:%s)", canName, chaincodeSupport.peerNetworkID, chaincodeSupport.peerID, cccid.TxID)
    }
    if err != nil {
        chaincodeLogger.Debugf("stopping due to error while launching %s", err)
        errIgnore := chaincodeSupport.Stop(ctxt, cccid, cds)
        if errIgnore != nil {
            chaincodeLogger.Debugf("error on stop %s(%s)", errIgnore, err)
        }
    }
    return err
}

```
- VMCProcess主要过程是开启协程，尝试获得container的锁，执行startImageReq的do函数，然后释放锁.在do中主要是调用了start函数来启动相应的container。对于执行环境为system(即system chaincode)和执行环境为docker(即用户级chaincode),分别对应不同的start函数。
core/container/controller.go

```go
func VMCProcess(ctxt context.Context, vmtype string, req VMCReqIntf) (interface{}, error) {
    v := vmcontroller.newVM(vmtype)

    if v == nil {
        return nil, fmt.Errorf("Unknown VM type %s", vmtype)
    }

    c := make(chan struct{})
    var resp interface{}
    go func() {
        defer close(c)

        id, err := v.GetVMName(req.getCCID())
        if err != nil {
            resp = VMCResp{Err: err}
            return
        }
        vmcontroller.lockContainer(id)
        resp = req.do(ctxt, v)
        vmcontroller.unlockContainer(id)
    }()

    select {
    case <-c:
        return resp, nil
    case <-ctxt.Done():
        //TODO cancel req.do ... (needed) ?
        <-c
        return nil, ctxt.Err()
    }
}
```

```go
func (si StartImageReq) do(ctxt context.Context, v api.VM) VMCResp {
    var resp VMCResp

    if err := v.Start(ctxt, si.CCID, si.Args, si.Env, si.Builder, si.PrelaunchFunc); err != nil {
        resp = VMCResp{Err: err}
    } else {
        resp = VMCResp{}
    }

    return resp
}
```
- 对于用户级chaincode. Start函数中，主要是docker启动的操作。在启动之前检查container是否已经在运行，如果是则通过vm.stopInternal清理、停止、并移除container。然后尝试创建container,对于image不存在的情况在失败后会首先尝试创建image，然后重新创建container.之后对创建好的container加上需要的attach(这里的主要是log的处理)，最后启动container.
core/container/dockercontroller/dockercontroller.go

```go
func (vm *DockerVM) Start(ctxt context.Context, ccid ccintf.CCID,
    args []string, env []string, builder container.BuildSpecFactory, prelaunchFunc container.PrelaunchFunc) error {
    imageID, err := vm.GetVMName(ccid)
    if err != nil {
        return err
    }

    client, err := vm.getClientFnc()
    if err != nil {
        dockerLogger.Debugf("start - cannot create client %s", err)
        return err
    }

    containerID := strings.Replace(imageID, ":", "_", -1)
    attachStdout := viper.GetBool("vm.docker.attachStdout")

    //stop,force remove if necessary
    dockerLogger.Debugf("Cleanup container %s", containerID)
    //处理container已经存在的情况，将container停止并移除
    vm.stopInternal(ctxt, client, containerID, 0, false, false)

    dockerLogger.Debugf("Start container %s", containerID)
    //创建容器
    err = vm.createContainer(ctxt, client, imageID, containerID, args, env, attachStdout)
    if err != nil {
        //镜像不存在，则创建镜像并重试，其它错误则直接返回错误
        if err == docker.ErrNoSuchImage {
            if builder != nil {
                dockerLogger.Debugf("start-could not find image <%s> (container id <%s>), because of <%s>..."+
                    "attempt to recreate image", imageID, containerID, err)

                reader, err1 := builder()
                if err1 != nil {
                    dockerLogger.Errorf("Error creating image builder for image <%s> (container id <%s>), "+
                        "because of <%s>", imageID, containerID, err1)
                }
                //部署镜像，该过程涉及docker镜像创建和部署过程，此处略
                if err1 = vm.deployImage(client, ccid, args, env, reader); err1 != nil {
                    return err1
                }

                dockerLogger.Debug("start-recreated image successfully")
                //创建容器
                if err1 = vm.createContainer(ctxt, client, imageID, containerID, args, env, attachStdout); err1 != nil {
                    dockerLogger.Errorf("start-could not recreate container post recreate image: %s", err1)
                    return err1
                }
            } else {
                dockerLogger.Errorf("start-could not find image <%s>, because of %s", imageID, err)
                return err
            }
        } else {
            dockerLogger.Errorf("start-could not recreate container <%s>, because of %s", containerID, err)
            return err
        }
    }

    if attachStdout {
        /*该部分主要是关于docker 容器的log的，忽略*/
    }

    if prelaunchFunc != nil {
        if err = prelaunchFunc(); err != nil {
            return err
        }
    }
    //Start Container
    err = client.StartContainer(containerID, nil)
    if err != nil {
        dockerLogger.Errorf("start-could not start container: %s", err)
        return err
    }

    dockerLogger.Debugf("Started container %s", containerID)
    return nil
}
```
- 对于system chaincode.会启动协程执行launchInProc完成chaincode的launch工作。
core/container/inproccontroller/inproccontroller.go

```go
//Start starts a previously registered system codechain
func (vm *InprocVM) Start(ctxt context.Context, ccid ccintf.CCID, args []string, env []string, builder container.BuildSpecFactory, prelaunchFunc container.PrelaunchFunc) error {
    path := ccid.ChaincodeSpec.ChaincodeId.Path
    ipctemplate := typeRegistry[path]
    if ipctemplate == nil {
        return fmt.Errorf(fmt.Sprintf("%s not registered", path))
    }

    instName, _ := vm.GetVMName(ccid)

    ipc, err := vm.getInstance(ctxt, ipctemplate, instName, args, env)

    if err != nil {
        return fmt.Errorf(fmt.Sprintf("could not create instance for %s", instName))
    }

    if ipc.running {
        return fmt.Errorf(fmt.Sprintf("chaincode running %s", path))
    }

    ccSupport, ok := ctxt.Value(ccintf.GetCCHandlerKey()).(ccintf.CCSupport)
    if !ok || ccSupport == nil {
        return fmt.Errorf("in-process communication generator not supplied")
    }

    if prelaunchFunc != nil {
        if err = prelaunchFunc(); err != nil {
            return err
        }
    }

    ipc.running = true

    go func() {
        defer func() {
            if r := recover(); r != nil {
                inprocLogger.Criticalf("caught panic from chaincode  %s", instName)
            }
        }()
        ipc.launchInProc(ctxt, instName, args, env, ccSupport)
    }()

    return nil
}
```