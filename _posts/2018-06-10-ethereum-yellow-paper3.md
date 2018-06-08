---
published: true
disqus: true
layout: post
mathjax: true
title: 以太坊黄皮书详解（三）
tags:
  - Ethereum
---
## 六、虚拟机的执行模型
上文三四五部分都是从流程上对交易执行（包括合约创建和合约调用）进行了介绍。本章节介绍以太坊中的虚拟机执行的流程。以太坊虚拟机EVM是图灵完备虚拟机器。EVM存在而典型图灵完备机器不存在的唯一限制就是EVM本质上是被gas束缚。因此，可以完成的计算总量本质上是被提供的gas总量限制的。

### 6.1 基本模型
![]({{ site.baseurl }}/images/evm.JPEG)
- EVM是基于栈（先进后出）的架构。EVM中每个堆栈项的大小为256位，堆栈有一个最大的大小，为1024位。
- EVM有内存，项目按照可寻址字节数组来存储。内存是易失性的，也就是数据是不持久的。
- EVM也有一个存储器。不像内存，存储器是非易失性的，并作为系统状态的一部分进行维护。EVM分开保存程序代码，在虚拟ROM 中只能通过特殊指令来访问。
- EVM同样有属于它自己的语言：“EVM字节码”，在以太坊上运行的智能合约时，通常都是用高级语言例如Solidity来编写代码。然后将它编译成EVM可以理解的EVM字节码。

```go
// EVM is the Ethereum Virtual Machine base object and provides
// the necessary tools to run a contract on the given state with
// the provided context. It should be noted that any error
// generated through any of the calls should be considered a
// revert-state-and-consume-all-gas operation, no checks on
// specific errors should ever be performed. The interpreter makes
// sure that any errors generated are to be considered faulty code.
//
// The EVM should never be reused and is not thread safe.
type EVM struct {
	// Context provides auxiliary blockchain related information
	Context
	// StateDB gives access to the underlying state
	StateDB StateDB
	// Depth is the current call stack
	depth int

	// chainConfig contains information about the current chain
	chainConfig *params.ChainConfig
	// chain rules contains the chain rules for the current epoch
	chainRules params.Rules
	// virtual machine configuration options used to initialise the
	// evm.
	vmConfig Config
	// global (to this context) ethereum virtual machine
	// used throughout the execution of the tx.
	interpreter *Interpreter
	// abort is used to abort the EVM calling operations
	// NOTE: must be set atomically
	abort int32
	// callGasTemp holds the gas available for the current call. This is needed because the
	// available gas is calculated in gasCall* according to the 63/64 rule and later
	// applied in opCall*.
	callGasTemp uint64
}

// Interpreter is used to run Ethereum based contracts and will utilise the
// passed environment to query external sources for state information.
// The Interpreter will run the byte code VM based on the passed
// configuration.
type Interpreter struct {
	evm      *EVM
	cfg      Config
	gasTable params.GasTable
	intPool  *intPool  //栈

	readOnly   bool   // Whether to throw on stateful modifications
	returnData []byte // Last CALL's return data for subsequent reuse
}
```
### 6.2 费用
以太坊虚拟机执行过程中，主要有3类费用。

- 执行过程中的运算费用。
- 创建或者调用其他合约消耗的费用。
- 新增的存储的费用。

### 6.3 运行环境

合约执行过程中的运行环境包括：系统状态$\sigma$,可用gas值g，以及其他包含在I中的一些值，在第四和第五部分也都有涉及到。

- $I_a$为合约执行的当前账户。对于合约创建则为新创建的合约账户，如果是合约调用，则为接收者账户。
- $I_o$为原始调用者，即该条交易的发送者。
- $I_p$为gas价格。
- $I_d$为合约执行的输入数据。
- $I_s$为合约当前调用者，如果是条简单的交易，则为交易的发送者。
- $I_v$为value，单位为Wei，即要转移给当前账户的以太币。
- $I_b$需要被执行的机器码。合约创建的时候该处即为初始化合约的字节码。
- $I_H$当前区块的header。
- $I_e$当前的栈深度。
- $I_w$相关权限。

执行模型可以用公式121表示。其中子状态如公式122所示。（该处子状态定义和之前是相同的）
$$
\begin{equation}
(\boldsymbol{\sigma}', g', A, \mathbf{o}) \equiv \Xi(\boldsymbol{\sigma}, g, I)
\tag{121}
\end{equation}
$$
$$
\begin{equation}
A \equiv (\mathbf{s}, \mathbf{l}, \mathbf{t}, r)
\tag{122}
\end{equation}
$$
### 6.4 执行过程
- 执行刚开始时，内存和堆栈都是空的，程序计数器为0。
- 然后EVM开始递归的执行交易，为每个循环计算系统状态和机器状态。系统状态也就是以太坊的全局状态(global state)。机器状态包含：可获取的gas，程序计数器，内存的内容，内存中字的活跃数，堆栈的内容。
- 堆栈中的项从栈顶被删除（POP）或者添加（PUSH）。
- 每个循环，剩余的gas都会被减少相应的量，程序计数器也会增加。

在每个循环的结束，都有四种可能性：

- 机器到达异常状态（err != nil 的情况。例如 gas不足，无效指令，堆栈项不足，堆栈项会溢出1024，无效的JUMP/JUMPI目的地等等）因此停止，并丢弃任何的更改进入后续处理下一个循环。
- 机器到达了受控停止（到达执行过程的终点，halts或者revert），整个结束了，机器就会产生一个合成状态，执行之后的剩余gas、产生的子状态、以及组合输出。
- 假设执行没有遇到异常状态，继续循环执行下一步。


```go
// Run loops and evaluates the contract's code with the given input data and returns
// the return byte-slice and an error if one occurred.
//
// It's important to note that any errors returned by the interpreter should be
// considered a revert-and-consume-all-gas operation except for
// errExecutionReverted which means revert-and-keep-gas-left.
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	// Increment the call depth which is restricted to 1024
	in.evm.depth++
	defer func() { in.evm.depth-- }()

	// Reset the previous call's return data. It's unimportant to preserve the old buffer
	// as every returning call will return new data anyway.
	in.returnData = nil

	// Don't bother with the execution if there's no code.
	if len(contract.Code) == 0 {
		return nil, nil
	}

	//机器状态包含：可获取的gas，程序计数器pc，内存的内容mem，内存中字的活跃数，堆栈的内容stack。
	var (
		op    OpCode        // current opcode
		mem   = NewMemory() // bound memory
		stack = newstack()  // local stack
		// For optimisation reason we're using uint64 as the program counter.
		// It's theoretically possible to go above 2^64. The YP defines the PC
		// to be uint256. Practically much less so feasible.
		pc   = uint64(0) // program counter
		cost uint64
		// copies used by tracer
		pcCopy  uint64 // needed for the deferred Tracer
		gasCopy uint64 // for Tracer to log gas remaining before execution
		logged  bool   // deferred Tracer should ignore already logged steps
	)
	contract.Input = input

	if in.cfg.Debug {
		defer func() {
			if err != nil {
				if !logged {
					in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
				} else {
					in.cfg.Tracer.CaptureFault(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
				}
			}
		}()
	}
	// 循环直到三种结束状态中的一种。
	// 1.异常情况，各种不合法的情况出现时，或者执行出错，会直接return
	// 2.执行中返回REVERT错误。
	// 3.执行中中断。 
	// The Interpreter main run loop (contextual). This loop runs until either an
	// explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
	// the execution of one of the operations or until the done flag is set by the
	// parent context.
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		if in.cfg.Debug {
			// Capture pre-execution values for tracing.
			logged, pcCopy, gasCopy = false, pc, contract.Gas
		}

		// Get the operation from the jump table and validate the stack to ensure there are
		// enough stack items available to perform the operation.
		// 获取当前要执行的操作
		op = contract.GetOp(pc)
		// 确认操作是否在操作集合表中
		operation := in.cfg.JumpTable[op]
		if !operation.valid {
			return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
		}
		// 确保目前栈满足操作要求，比方说有些操作是二元的，那么栈中至少要有两个数据。
		if err := operation.validateStack(stack); err != nil {
			return nil, err
		}
		// If the operation is valid, enforce and write restrictions
		if err := in.enforceRestrictions(op, operation, stack); err != nil {
			return nil, err
		}

		var memorySize uint64
		// calculate the new memory size and expand the memory to fit
		// the operation
		// 计算新操作需要的内存空间
		if operation.memorySize != nil {
			memSize, overflow := bigUint64(operation.memorySize(stack))
			if overflow {
				return nil, errGasUintOverflow
			}
			// memory is expanded in words of 32 bytes. Gas
			// is also calculated in words.
			if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
				return nil, errGasUintOverflow
			}
		}
		// consume the gas and return an error if not enough gas is available.
		// cost is explicitly set so that the capture state defer method can get the proper cost
		// 消耗gas
		cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
		if err != nil || !contract.UseGas(cost) {
			return nil, ErrOutOfGas
		}
		if memorySize > 0 {
			mem.Resize(memorySize)
		}

		if in.cfg.Debug {
			in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
			logged = true
		}

		// execute the operation
		// 执行具体的操作，比方说加法，就是从栈中取出两个值后相加
		res, err := operation.execute(&pc, in.evm, contract, mem, stack)
		// verifyPool is a build flag. Pool verification makes sure the integrity
		// of the integer pool by comparing values to a default value.
		if verifyPool {
			verifyIntegerPool(in.intPool)
		}
		// if the operation clears the return data (e.g. it has returning data)
		// set the last return to the result of the operation.
		if operation.returns {
			in.returnData = res
		}

		switch {
		case err != nil:
			return nil, err
		case operation.reverts:
			return res, errExecutionReverted
		case operation.halts:
			return res, nil
		case !operation.jumps:
			pc++ //计数器加一，接着执行
		}
	}
	return nil, nil
}

```

### 6.4.1 整体流程的形式化表示
虚拟机行执行合约的整体流程可以看作是输入系统状态$\sigma$,可用gas值g，输入项I，可以获取到的账户集合T。经过虚拟机函数$\Xi$后，得到新的系统状态$\sigma'$,执行剩余可用gas$\mu_g'$,交易子状态A和组合输出$\mathbf{o}$。如公式123所示。

执行的过程是一个迭代执行的过程，定义X为该迭代函数。如公式124所示。

其中机器状态$\mu$,包括：

- 可用gas值g，初始值即为输入的可用gas值，公式125。
- 程序计数器pc，初始值为0，公式126。
- 内存内容m，初始为全0，公式127。
- 内存内容中活跃的字数i，初始为0，公式128。
- 栈s，初始为空的栈，公式129。
- 组合输出，初始为空，公式130.

$$
\begin{eqnarray}
\Xi(\boldsymbol{\sigma}, g, I, T) & \equiv & (\boldsymbol{\sigma}', \boldsymbol{\mu}'_{\mathrm{g}}, A, \mathbf{o}) \tag{123}\\
(\boldsymbol{\sigma}', \boldsymbol{\mu}', A, ..., \mathbf{o}) & \equiv & X\big((\boldsymbol{\sigma}, \boldsymbol{\mu}, A^0, I)\big) \tag{124}\\
\boldsymbol{\mu}_{\mathrm{g}} & \equiv & g \tag{125}\\
\boldsymbol{\mu}_{\mathrm{pc}} & \equiv & 0 \tag{126}\\
\boldsymbol{\mu}_{\mathbf{m}} & \equiv & (0, 0, ...) \tag{127}\\
\boldsymbol{\mu}_{\mathrm{i}} & \equiv & 0 \tag{128}\\
\boldsymbol{\mu}_{\mathbf{s}} & \equiv & () \tag{129}\\
\boldsymbol{\mu}_{\mathbf{o}} & \equiv & () \tag{130}
\end{eqnarray}
$$


$$
\begin{equation} \label{eq:X-def}
X\big( (\boldsymbol{\sigma}, \boldsymbol{\mu}, A, I) \big) \equiv \begin{cases}
\big(\varnothing, \boldsymbol{\mu}, A^0, I, \varnothing\big) & \text{if} \quad Z(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \\
\big(\varnothing, \boldsymbol{\mu}', A^0, I, \mathbf{o}\big) & \text{if} \quad w = {\small REVERT} \\
O(\boldsymbol{\sigma}, \boldsymbol{\mu}, A, I) \cdot \mathbf{o} & \text{if} \quad \mathbf{o} \neq \varnothing \\
X\big(O(\boldsymbol{\sigma}, \boldsymbol{\mu}, A, I)\big) & \text{otherwise} \\
\end{cases}
\tag{131}
\end{equation}
$$

$$
\begin{eqnarray}
\mathbf{o} & \equiv & H(\boldsymbol{\mu}, I) \tag{132}\\
(a, b, c, d) \cdot e & \equiv & (a, b, c, d, e) \tag{133}\\
\boldsymbol{\mu}' & \equiv & \boldsymbol{\mu}\ \text{except:} \tag{134}\\
\boldsymbol{\mu}'_{\mathrm{g}} & \equiv & \boldsymbol{\mu}_{\mathrm{g}} - C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \tag{135}
\end{eqnarray}
$$

迭代函数X定义如公式131-135所示。

- 如果迭代过程中遇到异常，即$Z(\boldsymbol{\sigma}, \boldsymbol{\mu}, I)$,异常的具体情况见下一小节。则迭代结束，返回空的状态集$\varnothing$，原始机器状态$\boldsymbol{\mu}$，空的子状态$A^0$,空的组合输出$\varnothing$。
- 如果执行中隐式中断，当前要执行的操作为revert，则迭代结束，返回空的状态集$\varnothing$，临时机器状态$\boldsymbol{\mu'}$（扣除了gas消耗公式134-135），空的子状态$A^0$,组合输出$\mathbf{o}$。
- 如果执行过程正常，且达到了受控停止的状态，即组合输出$\mathbf{o}$不为空，则返回执行结果以及组合输出。组合输出由$H(\boldsymbol{\mu}, I)$确定公式132，详解见后文6.4.4。
- 如果执行过程正常，且组合输出为空，继续进行下一轮迭代。

当前要被执行的操作定义如公式136.

- 当程序计数器小于虚拟机字节码时，当前要执行的操作为字节码中对应操作。
- 否则为STOP

$$
\begin{equation}\label{eq:currentoperation}
w \equiv \begin{cases} I_{\mathbf{b}}[\boldsymbol{\mu}_{\mathrm{pc}}] & \text{if} \quad \boldsymbol{\mu}_{\mathrm{pc}} < \lVert I_{\mathbf{b}} \rVert \\
{\small STOP} & \text{otherwise}
\end{cases}
\tag{136}
\end{equation}
$$

### 6.4.2异常的形式化定义
虚拟机执行过程中的异常有以下几种情况,如公式137：
$$
\begin{equation} 
Z(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \equiv
\begin{array}[t]{l}
\boldsymbol{\mu}_g < C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \quad \vee \\
\mathbf{\delta}_w = \varnothing \quad \vee \\
\lVert\boldsymbol{\mu}_\mathbf{s}\rVert < \mathbf{\delta}_w \quad \vee \\
( w = {\small JUMP} \quad \wedge \quad \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) ) \quad \vee \\
( w = {\small JUMPI} \quad \wedge \quad \boldsymbol{\mu}_\mathbf{s}[1] \neq 0 \quad \wedge 
\quad \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) ) \quad \vee \\
( w = {\small RETURNDATACOPY} \quad \wedge  \quad \boldsymbol{\mu}_{\mathbf{s}}[1] + \boldsymbol{\mu}_{\mathbf{s}}[2] > \lVert\boldsymbol{\mu}_{\mathbf{o}}\rVert) \quad \vee \\
\lVert\boldsymbol{\mu}_\mathbf{s}\rVert - \mathbf{\delta}_w + \mathbf{\alpha}_w > 1024 \quad \vee \\
 (\neg I_{\mathrm{w}} \wedge W(w, \boldsymbol{\mu}))
\end{array}
\tag{137}
\end{equation}
$$

$$
\begin{equation}
W(w, \boldsymbol{\mu}) \equiv  
\begin{array}[t]{l}
w \in \{ {\small CREATE}, {\small SSTORE}, {\small SELFDESTRUCT}\} \ \vee \\
{\small LOG0} \le w \wedge w \le {\small LOG4} \quad \vee \\
w \in \{ {\small CALL}, {\small CALLCODE}\} \wedge \boldsymbol{\mu}_{\mathbf{s}}[2] \neq 0
\end{array}
\tag{138}
\end{equation}
$$

- gas值不足以支付花费。即$\boldsymbol{\mu}_g < C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I)$。
```go
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	//...
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		//...
		// gas值不足以支付花费
		cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
		if err != nil || !contract.UseGas(cost) {
			return nil, ErrOutOfGas
		}
		//...
	}
}
```
- 当前的操作是无效的。$\mathbf{\delta}_w = \varnothing$。当前操作w对应的字节码为空。
```go
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	//...
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		//...
		// 获取当前要执行的操作
		op = contract.GetOp(pc)
		// 确认操作是否在操作集合表中
		operation := in.cfg.JumpTable[op]
		if !operation.valid {
			return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
		}
		//...
	}
}
```
- 当前栈不合法（比方说二元操作，栈里只有一个数据）$\lVert\boldsymbol{\mu}_\mathbf{s}\rVert < \mathbf{\delta}_w$
```go
func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	//...
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		//...
		// 确保目前栈满足操作要求，比方说有些操作是二元的，那么栈中至少要有两个数据。
		if err := operation.validateStack(stack); err != nil {
			return nil, err
		}
		// If the operation is valid, enforce and write restrictions
		if err := in.enforceRestrictions(op, operation, stack); err != nil {
			return nil, err
		}
		//...
	}
}
```
- 当前操作为跳转，但是跳转到的地方不合法$( w = {\small JUMP} \wedge  \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) )$,
$(w = {\small JUMPI} \wedge \boldsymbol{\mu}_\mathbf{s}[1] \neq 0 \wedge  \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) )$
```go
//跳转操作
func opJump(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
	pos := stack.pop()
	if !contract.jumpdests.has(contract.CodeHash, contract.Code, pos) {
		nop := contract.GetOp(pos.Uint64())
		return nil, fmt.Errorf("invalid jump destination (%v) %v", nop, pos)
	}
	*pc = pos.Uint64()

	evm.interpreter.intPool.put(pos)
	return nil, nil
}

func opJumpi(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
	pos, cond := stack.pop(), stack.pop()
	if cond.Sign() != 0 {
		if !contract.jumpdests.has(contract.CodeHash, contract.Code, pos) {
			nop := contract.GetOp(pos.Uint64())
			return nil, fmt.Errorf("invalid jump destination (%v) %v", nop, pos)
		}
		*pc = pos.Uint64()
	} else {
		*pc++
	}

	evm.interpreter.intPool.put(pos, cond)
	return nil, nil
}
```
- 当前操作为返回数据操作，但是返回的数据长度超过了允许长度。$( w = {\small RETURNDATACOPY} \wedge  \boldsymbol{\mu}_{\mathbf{s}}[1] + \boldsymbol{\mu}_{\mathbf{s}}[2] > \lVert\boldsymbol{\mu}_{\mathbf{o}}\rVert)$
```go
func opReturnDataCopy(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
	var (
		memOffset  = stack.pop()
		dataOffset = stack.pop()
		length     = stack.pop()

		end = evm.interpreter.intPool.get().Add(dataOffset, length)
	)
	defer evm.interpreter.intPool.put(memOffset, dataOffset, length, end)
	//判断返回数据长度是否超过允许长度
	if end.BitLen() > 64 || uint64(len(evm.interpreter.returnData)) < end.Uint64() {
		return nil, errReturnDataOutOfBounds
	}
	memory.Set(memOffset.Uint64(), length.Uint64(), evm.interpreter.returnData[dataOffset.Uint64():end.Uint64()])

	return nil, nil
}

```
- 新的栈的大小大于1024,$\lVert\boldsymbol{\mu}_\mathbf{s}\rVert - \mathbf{\delta}_w + \mathbf{\alpha}_w > 1024$
- 没用权限却尝试非静态调用$\neg I_{\mathrm{w}} \wedge W(w, \boldsymbol{\mu})$。其中非静态调用有，CREATE，SSTORE，SELFDESTRUCT，LOG0，LOG1，LOG2，LOG3，LOG4，以及CALL和CALLCHAINCODE，如公式138。



### 6.4.3 跳转表的形式化定义

$$
\begin{equation}
D(\mathbf{c}) \equiv D_{J}(\mathbf{c}, 0)
\tag{139}
\end{equation}
$$

$$
\begin{equation}
D_{J}(\mathbf{c}, i) \equiv \begin{cases}
\{\} & \text{if} \quad i \geqslant |\mathbf{c}|  \\
\{ i \} \cup D_{J}(\mathbf{c}, N(i, \mathbf{c}[i])) & \text{if} \quad \mathbf{c}[i] = {\small JUMPDEST} \\
D_{J}(\mathbf{c}, N(i, \mathbf{c}[i])) & \text{otherwise} \\
\end{cases}
\tag{140}
\end{equation}
$$

$$
\begin{equation}
N(i, w) \equiv \begin{cases}
i + w - {\small PUSH1} + 2  
& \text{if} \quad w \in [{\small PUSH1}, {\small PUSH32}] \\
i + 1 & \text{otherwise} \end{cases}
\tag{141}
\end{equation}
$$
公式139-141为对跳转表的定义。

- 跳转到c，指跳转到c的第0个指令。公式140.
- 跳转到c的第i个指令，如果i大于等于c的长度，则跳转到空，否则跳转到i的下一个有效指令的位置。公式140.

其中$N(i,w)$指下一个有效的操作指令的位置，因为是找有效的指令，和不是数据，所以遇到push指令的时候需要跳过push进去的数据。

- 如果当前命令为PUSHX，则跳转到i+X+1（因为栈里接下来全是push进去的数据，寻找指令，要跳过这些数据）。
- 否则为i + 1.

### 6.4.4 正常的可控中断的形式化定义
正常的可控中断有以下三种：

- 当前的操作为RETURN或者REVERT，相应的进行返回。
- 当前操作为STOP或者SELFDESTRUCT。
- 其他情况返回空。

*注：这里STOP或者SELFDESTRUCT返回的并不是空，所以在6.1中也是o不为空的情况。*

$$
\begin{equation}
H(\boldsymbol{\mu}, I) \equiv \begin{cases}
H_{ {\tiny RETURN}}(\boldsymbol{\mu}) \ \text{if} \quad w \in \{{\small {RETURN}}, {\small REVERT}\} &\\
() \quad\quad\ \text{if} \quad w \in \{ {\small {STOP}}, {\small {SELFDESTRUCT}} \} &\\
\varnothing \quad\quad\ \text{otherwise}&
\end{cases}
\tag{142}
\end{equation}
$$

### 6.5 执行的生命周期的形式化表示
$$
\begin{eqnarray}
O\big((\boldsymbol{\sigma}, \boldsymbol{\mu}, A, I)\big) & \equiv & (\boldsymbol{\sigma}', \boldsymbol{\mu}', A', I) \tag{143}\\
\Delta & \equiv & \mathbf{\alpha}_{w} - \mathbf{\delta}_{w} \tag{144}\\
\lVert\boldsymbol{\mu}'_{\mathbf{s}}\rVert & \equiv & \lVert\boldsymbol{\mu}_{\mathbf{s}}\rVert + \Delta \tag{145}\\
\quad \forall x \in [\mathbf{\alpha}_{w}, \lVert\boldsymbol{\mu}'_{\mathbf{s}}\rVert): \boldsymbol{\mu}'_{\mathbf{s}}[x] & \equiv & \boldsymbol{\mu}_{\mathbf{s}}[x-\Delta] \tag{146}
\end{eqnarray}
$$

$$
\begin{eqnarray}
\quad \boldsymbol{\mu}'_{g} & \equiv & \boldsymbol{\mu}_{g} - C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \label{eq:mu_pc} \tag{147} \\
\quad \boldsymbol{\mu}'_{\mathrm{pc}} & \equiv & \begin{cases}
{J_{\text{JUMP}}}(\boldsymbol{\mu}) & \text{if} \quad w = {\small JUMP} \\
{J_{\text{JUMPI}}}(\boldsymbol{\mu}) & \text{if} \quad w = {\small JUMPI} \\
N(\boldsymbol{\mu}_{\mathrm{pc}}, w) & \text{otherwise}
\end{cases}
\tag{148}
\end{eqnarray}
$$

$$
\begin{eqnarray}
\boldsymbol{\mu}'_{\mathbf{m}} & \equiv & \boldsymbol{\mu}_{\mathbf{m}} \tag{149}\\
\boldsymbol{\mu}'_{\mathrm{i}} & \equiv & \boldsymbol{\mu}_{\mathrm{i}} \tag{150}\\
A' & \equiv & A \tag{151}\\
\boldsymbol{\sigma}' & \equiv & \boldsymbol{\sigma} \tag{152}
\end{eqnarray}
$$


$$
\begin{eqnarray}
B_{\mathrm{t}} & \equiv & B'_{\mathrm{t}} + B_{\mathrm{d}} \tag{153}\\
B' & \equiv & P(B_{H})
\tag{154}
\end{eqnarray}
$$

$$
\begin{equation}
\lVert B_{\mathbf{U}} \rVert \leqslant 2 \bigwedge_{\mathbf{U} \in B_{\mathbf{U}}} {V({\mathbf{U}}})\; \wedge \; k({\mathbf{U}}, P(\mathbf{B}_{\mathbf{H}})_{\mathbf{H}}, 6)
\tag{155}
\end{equation}
$$

$$
\begin{equation}
k(U, H, n) \equiv \begin{cases} false & \text{if} \quad n = 0 \\
s(U, H) \; \vee \; k(U, P(H)_{H}, n - 1) & \text{otherwise}
\end{cases}
\tag{156}
\end{equation}
$$

$$
\begin{equation}
s(U, H) \equiv (P(H) = P(U)\; \wedge \; H \neq U \; \wedge \; U \notin B(H)_{\mathbf{U}})
\tag{157}
\end{equation}
$$

$$
\begin{equation}
{B_{H}}_{\mathrm{g}} = {\ell}({\mathbf{R})_{\mathrm{u}}}
\tag{158}
\end{equation}
$$

$$
\begin{eqnarray}
\Omega(B, \boldsymbol{\sigma}) & \equiv & \boldsymbol{\sigma}': \boldsymbol{\sigma}' = \boldsymbol{\sigma} \quad \text{except:} \tag{159}\\
\qquad\boldsymbol{\sigma}'[{\mathbf{B}_{H}}_{\mathrm{c}}]_{\mathrm{b}} & = & \boldsymbol{\sigma}[{\mathbf{B}_{H}}_{\mathrm{c}}]_{\mathrm{b}} + \left(1 + \frac{\lVert \mathbf{B}_{\mathbf{U}}\rVert}{32}\right)R_{\mathrm{block}} \tag{160}\\
\qquad\forall_{\mathbf{U} \in \mathbf{B}_{\mathbf{U}}}: \tag{161}\\ \nonumber
\boldsymbol{\sigma}'[\mathbf{U}_{\mathrm{c}}] & = & \begin{cases}
\varnothing &\text{if}\ \boldsymbol{\sigma}[\mathbf{U}_{\mathrm{c}}] = \varnothing\ \wedge\ R = 0 \\
\mathbf{a}' &\text{otherwise}
\end{cases} \\
\mathbf{a}' &\equiv& (\boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{n}}, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{b}} + R, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathbf{s}}, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{c}}) \tag{162}\\
R & \equiv & \left(1 + \frac{1}{8} (U_{\mathrm{i}} - {B_{H}}_{\mathrm{i}})\right) R_{\mathrm{block}} \tag{163}
\end{eqnarray}
$$

$$
\begin{equation}
\text{Let} \quad R_{\mathrm{block}} = 3 \times 10^{18}
\tag{164}
\end{equation}
$$

$$
\begin{equation}
\Gamma(B) \equiv \begin{cases}
\boldsymbol{\sigma}_0 & \text{if} \quad P(B_{H}) = \varnothing \\
\boldsymbol{\sigma}_{\mathrm{i}}: \mathtt{\small {TRIE}}(L_{S}(\boldsymbol{\sigma}_{\mathrm{i}})) = {P(B_{H})_{H}}_{\mathrm{r}} & \text{otherwise}
\end{cases}
\tag{165}
\end{equation}
$$

$$
\begin{eqnarray}
\Phi(B) & \equiv & B': \quad B' = B^* \quad \text{except:} \tag{166}\\
B'_{\mathrm{n}} & = & n: \quad x \leqslant \frac{2^{256}}{H_{\mathrm{d}}} \tag{167}\\
B'_{\mathrm{m}} & = & m \quad \text{with } (x, m) = \mathtt{PoW}(B^*_{\cancel {n}}, n, \mathbf{d}) \tag{168}\\
B^* & \equiv & B \quad \text{except:} \quad {B^*_{\mathrm{r}}} = {r}({\Pi}(\Gamma(B), B)) \tag{169}
\end{eqnarray}
$$

$$
\begin{equation}
\boldsymbol{\sigma}[n] = \begin{cases} {\Gamma}(B) & \text{if} \quad n < 0 \\ 
{\Upsilon}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n]) & \text{otherwise} \end{cases}
\tag{170}
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathrm{u}} = \begin{cases} 0 & \text{if} \quad n < 0 \\
\begin{array}[b]{l}
\Upsilon^g(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n]) + \mathbf{R}[n-1]_{\mathrm{u}}
\end{array}
 & \text{otherwise} \end{cases}
 \tag{171}
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathbf{l}} =
\Upsilon^{\mathbf{l}}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n])
\tag{172}
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathrm{z}} =
\Upsilon^{\mathrm{z}}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n])
\tag{173}
\end{equation}
$$

$$
\begin{equation}
\Pi(\boldsymbol{\sigma}, B) \equiv {\Omega}(B, \ell(\boldsymbol{\sigma}))
\tag{174}
\end{equation}
$$

$$
\begin{equation}
m = {H_{\mathrm{m}}} \quad \wedge \quad n \leqslant \frac{2^{256}}{H_{\mathrm{d}}} \quad \text{with} \quad (m, n) = \mathtt{PoW}({H_{\cancel {n}}}, H_{\mathrm{n}}, \mathbf{d})
\tag{175}
\end{equation}
$$