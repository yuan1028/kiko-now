---
published: true
disqus: true
layout: post
mathjax: true
title: 以太坊中web3调用逻辑剖析
tags:
  - Ethereum
---

## 写在篇头

本篇要介绍的内容大概是回答以下几个问题。

1. geth是怎样或者使用何种技术在终端中实现了一个javascript的运行环境的。
1. 在终端中输入的一个命令是如何调到以太坊的底层函数，从而拿到想要的结果的。
1. 在终端中使用web3接口，和使用jsonrpc有什么区别。
1. 如何增加自己的web3接口，需要进行哪些修改。

## 1. JSRE(javascript runtime environment)
以太坊实现了一个javascript的运行环境即JSRE，可以在console这种交互模式或者script这种非交互模式中使用。详细使用见[JavaScript Console](https://github.com/ethereum/go-ethereum/wiki/JavaScript-Console)
- 交互模式，其可以通过console和attach子命令来启动。console是在启动geth的时候在启动完节点后打开终端。attach是在一个运行中的geth实例上开启一个终端。
```sh
$geth console
$geth attach
```
- 非交互模式，是指使用JSRE来执行文件。console和attach子命令都可以接受--exec的参数。
```sh
$geth --exec "eth.blockNumber" attach
$geth --exec 'loadScript("/tmp/checkbalances.js")' attach
```

### 1.2 非交互模式（script）使用
