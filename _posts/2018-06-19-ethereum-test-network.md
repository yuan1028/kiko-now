---
published: true
disqus: true
layout: post
mathjax: true
title: 搭建以太坊私链环境
tags:
  - Ethereum
---

## 1. 编译geth
**geth**是以太坊开发中最常用的工具，是官方客户端。可以运行以太坊节点、创建和管理账户、发送交易、挖矿、部署智能合约等。

由于我们在以太坊的基础上进行了一些代码的改动。所以环境搭建的时候建议采用直接从源码编译的方式。

从源码编译geth的过程：

- 查看当前go版本，确认go版本1.9及以上。
```sh
go version
```
- git clone 以太坊项目源码，地址：https://github.com/ethereum/go-ethereum.git ，之后我们自己项目的代码地址会另外提供。
```sh
git clone https://github.com/ethereum/go-ethereum.git
```
- 进入到目录$GOPATH/src/github.com/ethereum/go-ethereum目录下
```sh
make geth
```
- 编译成功则geth的可执行文件就会在以太坊目录的build/bin/目录下，为了方便使用，把该路径放入到PATH中。
```sh
export PATH=$PATH:$GOPATH/src/github.com/ethereum/go-ethereum/build/bin/
```

## 2. 编写创世区块配置
可以从网上随便贴一个创世区块的配置，比方说下面的配置，保存在genesis.json文件中（文件名随意）。
其中

- config为私链的一些相关配置，详见params/config.go中的ChainConfig结构体。修改共识模式等，都是需要修改config中的字段的。
- alloc为预设账号和账号的以太币数量，比方说账号A预设1000以太坊，就在alloc中相应设置。
- 其他字段为区块header中的一些字段，可以暂且不管。

```json
{
    "config":{
		"chainId":1999,
		"homesteadBlock":0,
		"eip155Block":0,
		"eip158Block":0
    },
    "nonce": "0x0",
    "timestamp": "0x00",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "0x00",
    "gasLimit": "0x8000000",
    "difficulty": "0x4000",
    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0x0000000000000000000000000000000000000000",
    "alloc": {     }
}
```

也可以使用官方的puppeth工具来生成创世区块的配置文件

- 使用前，首先在以太坊源码目录中，进行工具的编译,可以把所有工具都编译出来
```sh
make all 
```
或者如果只是要单独make puppeth的话
```sh
build/env.sh go run build/ci.go install ./cmd/puppeth
```
- 使用puppeth按步骤生成创世区块的配置文件

```
[**********@******]$ puppeth
+-----------------------------------------------------------+
| Welcome to puppeth, your Ethereum private network manager |
|                                                           |
| This tool lets you create a new Ethereum network down to  |
| the genesis block, bootnodes, miners and ethstats servers |
| without the hassle that it would normally entail.         |
|                                                           |
| Puppeth uses SSH to dial in to remote servers, and builds |
| its network components out of Docker containers using the |
| docker-compose toolset.                                   |
+-----------------------------------------------------------+

Please specify a network name to administer (no spaces or hyphens, please)
>
```
> 输入测试网络的名字，比方说test

```
> test

Sweet, you can set this via --network=test next time!

INFO [06-19|14:46:08] Administering Ethereum network           name=test
WARN [06-19|14:46:08] No previous configurations found         path=/Users/yuanzhenxia/.puppeth/test

What would you like to do? (default = stats)
 1. Show network stats
 2. Configure new genesis
 3. Track new remote server
 4. Deploy network components
>
```
> 配置新的创世区块，选择2

```
> 2

Which consensus engine to use? (default = clique)
 1. Ethash - proof-of-work
 2. Clique - proof-of-authority
>
```
> 选择共识模式，其中Ethash是PoW共识，Clique是PoA共识，这里可以先选择1

```
> 1

Which accounts should be pre-funded? (advisable at least one)
> 0x
```
> 选择预设账户，这个地方如果已经有账户，可以设，也可以直接按回车。

```
> 0x

Specify your chain/network ID if you want an explicit one (default = random)
>
```
> 选择网络id，可以输入一个整数，也可以直接回车，这一步结束，就是配置完成了。

```
>
INFO [06-19|14:50:49] Configured new genesis block

What would you like to do? (default = stats)
 1. Show network stats
 2. Manage existing genesis
 3. Track new remote server
 4. Deploy network components
```
> 配置已经完成，接下来选择2管理创世块配置

```
> 2

 1. Modify existing fork rules
 2. Export genesis configuration
 3. Remove genesis configuration
>
```
> 选择2来导出配置文件

```
> 2

Which file to save the genesis into? (default = test.json)
>
```
> 输入文件名，默认名称为第一步输入的网络名称跟.json。

```
Which file to save the genesis into? (default = test.json)
>
INFO [06-19|14:53:48] Exported existing genesis block
```
> ctrl+c退出puppeth命令行

## 3. 初始化并启动节点
### 3.1 初始化节点
在以太坊节点运行之前，需要运行geth init命令进行初始化操作。其中***.json即为第二步中的生成的创世区块的配置文件。初始化的主要作用是把配置在json文件中的内容，转化成以太坊链的配置，并生成0号区块创世块，将其写到数据库中(可以看下指定目录下应该多了个geth文件夹，进入geth文件夹后可以看到chaindata和lightchaindata两个文件夹)。

```sh
geth --datadir=[yourdir] init ***.json
```
*注意到命令中--datadir参数，geth命令又非常多的参数，详见geth help的内容。该处--datadir是将以太坊私链的内容存放到指定的目录的意思。如果不指定，默认是在$HOME/.ethereum目录下。*
### 3.2 启动节点
使用geth命令启动节点,只需要简单的使用geth命令就可以启动节点，如果要自己的需求，只需要跟相应的参数即可
```sh
geth --datadir=[yourdir]
```
### 3.3 启动web3
如果像3.2中启动节点，则只能看到节点的日志，无法和节点进行交互。我们还需要有个交互用的客户端，以太坊中支持多种交互的方式，其中web3是最常用的方式。

- 启动节点的时候启动web3,可以通过添加console命令在启动节点的时候，直接启动web3
```sh
geth --datadir=[yourdir] console
```
- 如果节点已经启动，可以通过attach命令来启动web3
```sh
geth --datadir=[yourdir] attach
```

## 4. 挖矿
在挖矿之前必须首先有账号来作为挖矿的收益者。

### 4.1 新建账号
新建账号有两种方法：

- web3中使用personal.newAccount()来新建账号，按提示输入密码，即可创建账号。
```js
> personal.newAccount()
Passphrase:
Repeat passphrase:
"0x****************************************" //这个就是此次新建的地址
```
- 使用geth的account new命令来新建账号，同样按提示输密码。
```
[****@****]$ geth --datadir=./ account new
INFO [06-19|19:51:23] Maximum peer count                       ETH=25 LES=0 total=25
Your new account is locked with a password. Please give a password. Do not forget this password.
Passphrase:
Repeat passphrase:
Address: {****************************************}
```

### 4.2 解锁账号
以太坊中要挖矿，或者要发送交易的时候，账号需要是解锁状态，否则会提示相应的错误。
使用web3的personal.unlockAccount方法,输入密码后可以解锁账号。
```js
> personal.unlockAccount(eth.accounts[0])
Unlock account 0x****************************************
Passphrase:
true
```
默认是使用eth.accounts中的第一个账号做挖矿者的，如果想要使用其他账号作为矿工，需要使用miner.setEtherbase()进行设置，然后记得解锁相应的账号。
```js
miner.setEtherbase(eth.accounts[1])
```

### 4.3 开始挖矿
使用miner.start()可以开始挖矿。
```js
>miner.start()
```
可以通过eth.blockNumber看到区块号在增加
```js
>eth.blockNumber
```

### 4.4 停止挖矿
使用miner.stop()可以停止挖矿。
```js
miner.stop()
```

## 5. 发送交易
在没有安装其他智能合约的情况下，以太坊也可以使用eth.sendTransaction进行基本的以太币转账操作(发送者账号必须是解锁状态)，如下命令就是账号0向账号1转1个以太币。
```js
eth.sendTransaction({from:eth.accounts[0],to:eth.accounts[1],value:web3.toWei(1,"ether")})
```

## 6. 多个节点的情况
多个节点组成一个以太坊网络需要保证其创世区块是一致的。

- 把创世块的配置文件***.json，拷贝到多台机器，或者多个文件夹。（datadir指定的文件夹只能被一个节点访问，如果同时在一个文件夹试图启动多个节点，会报错）
- 每个文件夹目录下，分别进行geth init 操作。

接下来是启动节点，并使得节点间是相互连通的，这个有两种方法。

### 6.1 通过bootnodes来保证联通。

- 先启动一个节点。
- 查看这个节点的enode(admin.nodeInfo.enode)，这个可以看作是节点的地址。
```js
> admin.nodeInfo.enode
"enode://bafe93edee0c5cfab6a3d12927553abe6f9a3cf31b56b58d65c05e52edf184dcf88f8a2a3d84f3911bc2233c0140188e1e8717c26ff98a0268298c3a933c1855@[::]:30303"
```
- 其他节点启动的时候，添加--bootnodes参数,并跟上刚才查看到的enode内容。如果是在同一台机器上不同节点要设置成不同的端口号，默认的端口号为30303，端口号通过--port来设置。
```sh
geth --datadir=./ --bootnodes=enode://bafe93edee0c5cfab6a3d12927553abe6f9a3cf31b56b58d65c05e52edf184dcf88f8a2a3d84f3911bc2233c0140188e1e8717c26ff98a0268298c3a933c1855@127.0.0.1:30303 --port=31303 console
```

### 6.2 通过admin.addPeer来保证联通
如果节点已经启动，或者临时有新的节点想要加入进网络，都可以使用admin.addPeer的方法来添加，参数为上一小节介绍的enode。
```js
admin.addPeer("enode://bafe93edee0c5cfab6a3d12927553abe6f9a3cf31b56b58d65c05e52edf184dcf88f8a2a3d84f3911bc2233c0140188e1e8717c26ff98a0268298c3a933c1855@127.0.0.1:30303")
```
可以通过admin.peers查看节点情况。
```js
admin.peers
```
## 7. 常用的web3指令
web3的详细API参考[JavaScript API · ethereum/wiki Wiki](https://github.com/ethereum/wiki/wiki/JavaScript-API) ，本文中只介绍涉及到的API。

### 7.1 eth相关命令
eth相关命令，主要是用来查看链当前状态，发送交易给链等操作。在web3窗口输入eth可以看到。
其中常用的有：

- eth.accounts,查看当前节点下有哪些账号，相应的账户信息在指定目录的keystore目录下。
- eth.blockNumber,查看当前节点的最新的区块号。
- eth.getBlock(),参数为区块号或者区块hash，可以查看区块的内容。例如eth.getBlock(0)。
- eth.getBalance(),参数为账号地址，可以查看账户的余额（单位为：Wei，一个以太币1 ether=10^18 Wei）。例如eth.getBalance(eth.accounts[0]).
- eth.getTransaction(),参数为transaction hash，可以查看交易详情。
- eth.sendTransaction(),发送交易，直接使用就是转以太币，如果是有合约，用合约调，就是运行合约相应的方法。例如从账户1转1以太币到账户2，eth.sendTransaction({from:eth.accounts[0],to:eth.accounts[1],value:web3.toWei(1,"ether")})
- eth.mining，查看当前节点是否在挖矿。
- eth.syncing,查看当前节点是否在同步区块。

### 7.2 miner相关命令
miner即为以太坊中矿工相关的命令。其中常用的有：

- miner.start()启动矿工开始挖矿。
- miner.stop()矿工停止挖矿。
- miner.setEtherbase()设置矿工账号。

### 7.3 admin相关命令
其中比较常用的有：

- admin.nodeInfo，查看当前节点信息。其中的enode比较重要。
- admin.peers,查看当前节点联通的节点的信息，可以查看网络中节点是否互联。
- admin.addPeer([enode]),添加节点.

### 7.4 personal相关命令
其中比较常用的有：

- personal.newAccount()新建账号。
- personal.unlockAccount()解锁账号，参数为账号的Address。

### 7.5 txpool相关命令
txpool主要是用来查看当前交易池状态的命令，当交易发送之后，会首先到交易池中，之后才会被矿工挖矿，然后被打到区块中之后才会从交易池中删除。

### 7.6 其他工具类命令
- web3.fromAscii将字符串转化为16进制形式的字符数组。如 web3.fromAscii("abc")返回"0x616263"。
- web3.fromDecimal将十进制的数字转化为16进制形式的字符数组。如 web3.fromDecimal(100)返回"0x64"。
- web3.fromWei将以Wei为单位的以太币转化为以ether（通常所说的1以太币指的是1 ether）为单位的以太币，如web3.fromWei(1000000000000000000)返回"1"。
- web3.sha3将数据进行hash，如：web3.sha3("100")返回"0x8c18210df0d9514f2d2e5d8ca7c100978219ee80d3968ad850ab5ead208287b3"。
- web3.toAscii,将16进制形式的字符数组转化为ascii码的字符数组，如 web3.toAscii("0x616263")返回"abc"。
- web3.toDecimal将16进制形式的字符数组转化为十进制的数字。如 web3.toDecimal("0x64")返回100。
- web3.toWei将以太币转化为以Wei为单元，如web3.toWei(1)返回"1000000000000000000"。
