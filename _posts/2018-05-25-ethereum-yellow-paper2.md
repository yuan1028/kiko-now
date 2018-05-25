---
published: true
disqus: true
layout: post
mathjax: true
title: 以太坊黄皮书详解（二）
tags:
  - Ethereum
---
## 交易执行

```go
/*
The State Transitioning Model

A state transition is a change made when a transaction is applied to the current world state
The state transitioning model does all all the necessary work to work out a valid new state root.

1) Nonce handling
2) Pre pay gas
3) Create a new state object if the recipient is \0*32
4) Value transfer
== If contract creation ==
  4a) Attempt to run transaction data
  4b) If valid, use result as code for the new state object
== end ==
5) Run Script section
6) Derive new state root
*/
//gp 中一开始有gasLimit数量的gas
type StateTransition struct {
	gp         *GasPool
	msg        Message
	gas        uint64
	gasPrice   *big.Int
	initialGas uint64
	value      *big.Int
	data       []byte
	state      vm.StateDB
	evm        *vm.EVM
}
```

```go
// TransitionDb will transition the state by applying the current message and
// returning the result including the the used gas. It returns an error if it
// failed. An error indicates a consensus issue.
func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
	if err = st.preCheck(); err != nil {
		return
	}
	msg := st.msg
	sender := vm.AccountRef(msg.From())
	homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
	contractCreation := msg.To() == nil

	// Pay intrinsic gas
	// 预付gas，也就是g0
	gas, err := IntrinsicGas(st.data, contractCreation, homestead)
	if err != nil {
		return nil, 0, false, err
	}
	if err = st.useGas(gas); err != nil {
		return nil, 0, false, err
	}

	var (
		evm = st.evm
		// vm errors do not effect consensus and are therefor
		// not assigned to err, except for insufficient balance
		// error.
		vmerr error
	)
	if contractCreation {
		ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
	} else {
		// Increment the nonce for the next transaction
		st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
		ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
	}
	if vmerr != nil {
		log.Debug("VM returned with error", "err", vmerr)
		// The only possible consensus-error would be if there wasn't
		// sufficient balance to make the transfer happen. The first
		// balance transfer may never fail.
		if vmerr == vm.ErrInsufficientBalance {
			return nil, 0, false, vmerr
		}
	}
	st.refundGas()
	st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))

	return ret, st.gasUsed(), vmerr != nil, err
}
```

```go
func (st *StateTransition) preCheck() error {
	// Make sure this transaction's nonce is correct.
	if st.msg.CheckNonce() {
		nonce := st.state.GetNonce(st.msg.From())
		if nonce < st.msg.Nonce() {
			return ErrNonceTooHigh
		} else if nonce > st.msg.Nonce() {
			return ErrNonceTooLow
		}
	}
	return st.buyGas()
}
```

```go
func (st *StateTransition) buyGas() error {
	mgval := new(big.Int).Mul(new(big.Int).SetUint64(st.msg.Gas()), st.gasPrice)
	if st.state.GetBalance(st.msg.From()).Cmp(mgval) < 0 {
		return errInsufficientBalanceForGas
	}
	if err := st.gp.SubGas(st.msg.Gas()); err != nil {
		return err
	}
	st.gas += st.msg.Gas()

	st.initialGas = st.msg.Gas()
	st.state.SubBalance(st.msg.From(), mgval)
	return nil
}
```

```go
func (st *StateTransition) refundGas() {
	// Apply refund counter, capped to half of the used gas.
	refund := st.gasUsed() / 2
	if refund > st.state.GetRefund() {
		refund = st.state.GetRefund()
	}
	st.gas += refund

	// Return ETH for remaining gas, exchanged at the original rate.
	remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
	st.state.AddBalance(st.msg.From(), remaining)

	// Also return remaining gas to the block gas counter so it is
	// available for the next transaction.
	st.gp.AddGas(st.gas)
}

// gasUsed returns the amount of gas used up by the state transition.
func (st *StateTransition) gasUsed() uint64 {
	return st.initialGas - st.gas
}
```
$$
\begin{equation}
\boldsymbol{\sigma}' = \Upsilon(\boldsymbol{\sigma}, T)
\tag{51}
\end{equation}
$$

$$
\begin{equation}
A \equiv (A_{\mathbf{s}}, A_{\mathbf{l}}, A_{\mathbf{t}}, A_{\mathrm{r}})
\tag{52}
\end{equation}
$$

$$
\begin{equation}
A^0 \equiv (\varnothing,(), \varnothing, 0)
\tag{53}
\end{equation}
$$

```go
// IntrinsicGas computes the 'intrinsic gas' for a message with the given data.
func IntrinsicGas(data []byte, contractCreation, homestead bool) (uint64, error) {
	// Set the starting gas for the raw transaction
	var gas uint64
	if contractCreation && homestead {
		gas = params.TxGasContractCreation
	} else {
		gas = params.TxGas
	}
	// Bump the required gas by the amount of transactional data
	if len(data) > 0 {
		// Zero and non-zero bytes are priced differently
		var nz uint64
		for _, byt := range data {
			if byt != 0 {
				nz++
			}
		}
		// Make sure we don't exceed uint64 for all data combinations
		if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
			return 0, vm.ErrOutOfGas
		}
		gas += nz * params.TxDataNonZeroGas

		z := uint64(len(data)) - nz
		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, vm.ErrOutOfGas
		}
		gas += z * params.TxDataZeroGas
	}
	return gas, nil
}

```
$$
\begin{align}
 g_0 \equiv {} & \sum_{i \in T_{\mathbf{i}}, T_{\mathbf{d}}} \begin{cases} {G_{\mathrm{txdatazero}}} & \text{if} \quad i = 0 \\  {G_{\mathrm{txdatanonzero}}} & \text{otherwise} \end{cases} \tag{54}\\
{} & + \begin{cases} {G_{\mathrm{txcreate}}} & \text{if} \quad T_{\mathrm{t}} = \varnothing \\ 0 & \text{otherwise} \end{cases} \tag{55}\\
{} & + {G_{\mathrm{transaction}}} \tag{56}
\end{align}
$$

$$
\begin{equation}
v_0 \equiv {T_{\mathrm{g}}} {T_{\mathrm{p}}} + {T_{\mathrm{v}}}
\tag{57}
\end{equation}
$$

$$
\begin{equation}
\begin{array}[t]{rcl}
S(T) & \neq & \varnothing \quad \wedge \\
\boldsymbol{\sigma}[S(T)] & \neq & \varnothing \quad \wedge \\
{}T_{\mathrm{n}} & = & \boldsymbol{\sigma}[S(T)]_{\mathrm{n}} \quad \wedge \\
g_0 & \leqslant & T_{\mathrm{g}} \quad \wedge \\
v_0 & \leqslant & \boldsymbol{\sigma}[S(T)]_{\mathrm{b}} \quad \wedge \\
T_{\mathrm{g}} & \leqslant & {B_{H}}_{\mathrm{l}} - {\ell}(B_{\mathbf{R}})_{\mathrm{u}}
\end{array}
\tag{58}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_0  \equiv  \boldsymbol{\sigma} \quad \text{except:} 
\tag{59}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_0[S(T)]_{\mathrm{b}}  \equiv  \boldsymbol{\sigma}[S(T)]_{\mathrm{b}} - T_{\mathrm{g}} T_{\mathrm{p}} 
\tag{60}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_0[S(T)]_{\mathrm{n}}  \equiv  \boldsymbol{\sigma}[S(T)]_{\mathrm{n}} + 1
\tag{61}
\end{equation}
$$

$$
\begin{equation}
(\boldsymbol{\sigma}_{P}, g', A, z) \equiv \begin{cases}
\Lambda_{4}(\boldsymbol{\sigma}_0, S(T), T_{\mathrm{o}}, g, &\\ \quad\quad T_{\mathrm{p}}, T_{\mathrm{v}}, T_{\mathbf{i}}, 0, \top) & \text{if} \quad T_{\mathrm{t}} = \varnothing \\
\Theta_{4}(\boldsymbol{\sigma}_0, S(T), T_{\mathrm{o}}, T_{\mathrm{t}}, T_{\mathrm{t}}, &\\ \quad\quad g, T_{\mathrm{p}}, T_{\mathrm{v}}, T_{\mathrm{v}}, T_{\mathbf{d}}, 0, \top) & \text{otherwise}
\end{cases}
\tag{62}
\end{equation}
$$

$$
\begin{equation}
g \equiv T_{\mathrm{g}} - g_0
\tag{63}
\end{equation}
$$

$$
\begin{equation}
	A'_{r} \equiv A_{r} + \sum_{i \in A_{s}} R_{selfdestruct}
\tag{64}
\end{equation}
$$

$$
\begin{equation}
g^* \equiv g' + \min \left\{ \Big\lfloor \dfrac{T_{\mathrm{g}} - g'}{2} \Big\rfloor, {A'_{\mathrm{r}}} \right\}
\tag{65}
\end{equation}
$$

$$
\begin{eqnarray}
 \boldsymbol{\sigma}^* & \equiv & \boldsymbol{\sigma}_{P} \quad \text{except} \tag{66}\\
 \boldsymbol{\sigma}^*[S(T)]_{\mathrm{b}} & \equiv & \boldsymbol{\sigma}_{P}[S(T)]_{\mathrm{b}} + g^* T_{\mathrm{p}} \tag{67}\\
 \boldsymbol{\sigma}^*[m]_{\mathrm{b}} & \equiv & \boldsymbol{\sigma}_{P}[m]_{\mathrm{b}} + (T_{\mathrm{g}} - g^*) T_{\mathrm{p}} \tag{68}\\
m & \equiv & {B_{H}}_{\mathrm{c}} \tag{69}
\end{eqnarray}
$$

$$
\begin{eqnarray}
 \boldsymbol{\sigma}' & \equiv & \boldsymbol{\sigma}^* \quad \text{except} \tag{70}\\
 {}\forall i \in A_{\mathbf{s}}: \boldsymbol{\sigma}'[i] & = & \varnothing \tag{71}\\
{}\forall i \in A_{\mathbf{t}}: \boldsymbol{\sigma}'[i] & = & \varnothing \quad\text{if}\quad \mathtt{\tiny DEAD}(\boldsymbol{\sigma}^*\kern -2pt, i) \tag{72}
\end{eqnarray}
$$

$$
\begin{eqnarray}
\Upsilon^{\mathrm{g}}(\boldsymbol{\sigma}, T) & \equiv & T_{\mathrm{g}} - g^* \tag{73}\\
\Upsilon^{\mathbf{l}}(\boldsymbol{\sigma}, T) & \equiv & {A_{\mathbf{l}}} \tag{74}\\
\Upsilon^{\mathrm{z}}(\boldsymbol{\sigma}, T) & \equiv & z \tag{75}
\end{eqnarray}
$$

$$
\begin{equation}
(\boldsymbol{\sigma}', g', A, z, \mathbf{o}) \equiv \Lambda(\boldsymbol{\sigma}, s, o, g, p, v, \mathbf{i}, e, w)
\tag{76}
\end{equation}
$$

$$
\begin{equation}
a \equiv \mathcal{B}_{96..255}\Big(\mathtt{\tiny KEC}\Big(\mathtt{\tiny RLP}\big(\;(s, \boldsymbol{\sigma}[s]_{\mathrm{n}} - 1)\;\big)\Big)\Big)
\tag{77}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}^* \equiv \boldsymbol{\sigma} \quad \text{except:}
\tag{78}
\end{equation}
$$

$$
\begin{eqnarray}
 \boldsymbol{\sigma}^*[a] &=& \big( 1, v + v', \mathtt{\tiny {TRIE}}(\varnothing), \mathtt{\tiny KEC}\big(()\big) \big) \tag{79}\\
 \boldsymbol{\sigma}^*[s] &=& \begin{cases}
\varnothing & \text{if}\ \boldsymbol{\sigma}[s] = \varnothing \ \wedge\ v = 0 \\
\mathbf{a}^* & \text{otherwise}
\end{cases} \tag{80}\\
\mathbf{a}^* &\equiv& (\boldsymbol{\sigma}[s]_{\mathrm{n}}, \boldsymbol{\sigma}[s]_{\mathrm{b}} - v, \boldsymbol{\sigma}[s]_{\mathbf{s}}, \boldsymbol{\sigma}[s]_{\mathrm{c}})
\tag{81}
\end{eqnarray}
$$

$$
\begin{equation}
v' \equiv \begin{cases}
0 & \text{if} \quad \boldsymbol{\sigma}[a] = \varnothing\\
\boldsymbol{\sigma}[a]_{\mathrm{b}} & \text{otherwise}
\end{cases}
\tag{82}
\end{equation}
$$

$$
\begin{equation}
(\boldsymbol{\sigma}^{**}, g^{**}, A, \mathbf{o}) \equiv \Xi(\boldsymbol{\sigma}^*, g, I, \{s, a\}) 
\tag{83}
\end{equation}
$$

$$
\begin{eqnarray}
 I_{\mathrm{a}} & \equiv & a \tag{84}\\
 I_{\mathrm{o}} & \equiv & o \tag{85}\\
 I_{\mathrm{p}} & \equiv & p \tag{86}\\
 I_{\mathbf{d}} & \equiv & () \tag{87}\\
 I_{\mathrm{s}} & \equiv & s \tag{88}\\
 {I_{\mathrm{v}}} & \equiv & v \tag{89}\\
 I_{\mathbf{b}} & \equiv & \mathbf{i} \tag{90}\\
 I_{\mathrm{e}} & \equiv & e \tag{91}\\
I_{\mathrm{w}} & \equiv & w \tag{92}
\end{eqnarray}
$$

$$
\begin{equation}
c \equiv G_{codedeposit} \times |\mathbf{o}|
\tag{93}
\end{equation}
$$

$$
\begin{align}
\quad g' &\equiv \begin{cases}
0 & \text{if} \quad F \\
g^{**} - c & \text{otherwise} \\
\end{cases} \tag{94}\\
\quad \boldsymbol{\sigma}' &\equiv  \begin{cases}
\boldsymbol{\sigma} & \text{if} \quad F \\
\boldsymbol{\sigma}^{**} \quad \text{except:} & \\
\quad\boldsymbol{\sigma}'[a] = \varnothing & \text{if} \quad \mathtt{\tiny DEAD}(\boldsymbol{\sigma}^{**}, a) \\
\boldsymbol{\sigma}^{**} \quad \text{except:} & \\
\quad\boldsymbol{\sigma}'[a]_{\mathrm{c}} = {\small KEC}(\mathbf{o}) & \text{otherwise}
\end{cases} \tag{95}\\
\quad z &\equiv \begin{cases}
0 & \text{if} \quad \boldsymbol{\sigma}^{**} = \varnothing \lor g^{**} < c \\
1 & \text{otherwise}
\end{cases} \tag{96}\\
F &\equiv \big((\boldsymbol{\sigma}^{**} = \varnothing \ \wedge\ \mathbf{o} = \varnothing) \vee\  g^{**} < c \ \vee\  |\mathbf{o}| > 24576\big)
\tag{97}
\end{align}
$$

$$
\begin{equation}
(\boldsymbol{\sigma}', g', A, z, \mathbf{o}) \equiv \Theta(\boldsymbol{\sigma}, s, o, r, c, g, p, v, \tilde{v}, \mathbf{d}, e, w)
\tag{98}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_1[r]_{\mathrm{b}} \equiv \boldsymbol{\sigma}[r]_{\mathrm{b}} + v \quad\wedge\quad \boldsymbol{\sigma}_1[s]_{\mathrm{b}} \equiv \boldsymbol{\sigma}[s]_{\mathrm{b}} - v
\tag{99}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_1 \equiv \boldsymbol{\sigma}_1' \quad \text{except:} 
\tag{100}
\end{equation}
$$

$$
\begin{equation}
\boldsymbol{\sigma}_1[s] \equiv \begin{cases}
\varnothing & \text{if}\ \boldsymbol{\sigma}_1'[s] = \varnothing \ \wedge\ v = 0 
\mathbf{a}_1 &\text{otherwise}
\end{cases}
\tag{101}
\end{equation}
$$

$$
\begin{equation}
\mathbf{a}_1 \equiv \left(\boldsymbol{\sigma}_1'[s]_{\mathrm{n}}, \boldsymbol{\sigma}_1'[s]_{\mathrm{b}} - v, \boldsymbol{\sigma}_1'[s]_{\mathbf{s}}, \boldsymbol{\sigma}_1'[s]_{\mathrm{c}}\right)
\tag{102}
\end{equation}
$$

$$
\begin{equation}
\text{and}\quad \boldsymbol{\sigma}_1' \equiv \boldsymbol{\sigma} \quad \text{except:} 
\tag{103}
\end{equation}
$$

$$
\begin{equation}
\begin{cases}
\boldsymbol{\sigma}_1'[r] \equiv (0, v, \mathtt{\tiny TRIE}(\varnothing), \mathtt{\tiny KEC}(())) & \text{if} \quad \boldsymbol{\sigma}[r] = \varnothing \wedge v \neq 0 \\
\boldsymbol{\sigma}_1'[r] \equiv \varnothing & \text{if}\quad \boldsymbol{\sigma}[r] = \varnothing \wedge v = 0 \\
\boldsymbol{\sigma}_1'[r] \equiv \mathbf{a}_1' & \text{otherwise}
\end{cases}
\tag{104}
\end{equation}
$$

$$
\begin{equation}
\mathbf{a}_1' \equiv (\boldsymbol{\sigma}[r]_{\mathrm{n}}, \boldsymbol{\sigma}[r]_{\mathrm{b}} + v, \boldsymbol{\sigma}[r]_{\mathbf{s}}, \boldsymbol{\sigma}[r]_{\mathrm{c}})
\tag{105}
\end{equation}
$$

$$
\begin{eqnarray}
\boldsymbol{\sigma}' & \equiv & \begin{cases}
\boldsymbol{\sigma} & \text{if} \quad \boldsymbol{\sigma}^{**} = \varnothing \\
\boldsymbol{\sigma}^{**} & \text{otherwise}
\end{cases} \tag{106}\\
g' & \equiv & \begin{cases}
0 & \text{if} \quad \boldsymbol{\sigma}^{**} = \varnothing \ \wedge \\
&\quad \mathbf{o} = \varnothing \\
g^{**} & \text{otherwise}
\end{cases} \tag{107}\\ 
z & \equiv & \begin{cases}
0 & \text{if} \quad \boldsymbol{\sigma}^{**} = \varnothing \\
1 & \text{otherwise}
\end{cases} \tag{108}\\
(\boldsymbol{\sigma}^{**}, g^{**},A, \mathbf{o}) & \equiv & \Xi \tag{109}\\
I_{\mathrm{a}} & \equiv & r \tag{110}\\
I_{\mathrm{o}} & \equiv & o \tag{111}\\
I_{\mathrm{p}} & \equiv & p \tag{112}\\
I_{\mathbf{d}} & \equiv & \mathbf{d} \tag{113}\\
I_{\mathrm{s}} & \equiv & s \tag{114}\\
I_{\mathrm{v}} & \equiv & \tilde{v} \tag{115}\\
I_{\mathrm{e}} & \equiv & e \tag{116}\\
I_{\mathrm{w}} & \equiv & w \tag{117}\\
\mathbf{t} & \equiv & \{s, r\} \tag{118}\\
\end{eqnarray}
$$

$$
\begin{equation}
\Xi \equiv \begin{cases}
\Xi_{\mathtt{ECREC}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 1 \\
\Xi_{\mathtt{SHA256}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 2 \\
\Xi_{\mathtt{RIP160}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 3 \\
\Xi_{\mathtt{ID}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 4 \\
\Xi_{\mathtt{EXPMOD}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 5 \\
\Xi_{\mathtt{BN\_ADD}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 6 \\
\Xi_{\mathtt{BN\_MUL}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 7 \\
\Xi_{\mathtt{SNARKV}}(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{if} \quad r = 8 \\
\Xi(\boldsymbol{\sigma}_1, g, I, \mathbf{t}) & \text{otherwise} \end{cases}
\tag{119}
\end{equation}
$$

$$
\begin{equation}
\text{Let} \; \mathtt{\tiny KEC}(I_{\mathbf{b}}) = \boldsymbol{\sigma}[c]_{\mathrm{c}}
\tag{120}
\end{equation}
$$

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

$$
\begin{eqnarray}
\Xi(\boldsymbol{\sigma}, g, I, T) & \equiv & (\boldsymbol{\sigma}', \boldsymbol{\mu}'_{\mathrm{g}}, A, \mathbf{o}) \\
(\boldsymbol{\sigma}', \boldsymbol{\mu}', A, ..., \mathbf{o}) & \equiv & X\big((\boldsymbol{\sigma}, \boldsymbol{\mu}, A^0, I)\big) \\
\boldsymbol{\mu}_{\mathrm{g}} & \equiv & g \\
\boldsymbol{\mu}_{\mathrm{pc}} & \equiv & 0 \\
\boldsymbol{\mu}_{\mathbf{m}} & \equiv & (0, 0, ...) \\
\boldsymbol{\mu}_{\mathrm{i}} & \equiv & 0 \\
\boldsymbol{\mu}_{\mathbf{s}} & \equiv & () \\
\boldsymbol{\mu}_{\mathbf{o}} & \equiv & ()
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
\end{equation}
$$

$$
\begin{eqnarray}
\mathbf{o} & \equiv & H(\boldsymbol{\mu}, I) \\
(a, b, c, d) \cdot e & \equiv & (a, b, c, d, e) \\
\boldsymbol{\mu}' & \equiv & \boldsymbol{\mu}\ \text{except:} \\
\boldsymbol{\mu}'_{\mathrm{g}} & \equiv & \boldsymbol{\mu}_{\mathrm{g}} - C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I)
\end{eqnarray}
$$

$$
\begin{equation}\label{eq:currentoperation}
w \equiv \begin{cases} I_{\mathbf{b}}[\boldsymbol{\mu}_{\mathrm{pc}}] & \text{if} \quad \boldsymbol{\mu}_{\mathrm{pc}} < \lVert I_{\mathbf{b}} \rVert \\
{\small STOP} & \text{otherwise}
\end{cases}
\end{equation}
$$

$$
\begin{equation} 
Z(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \equiv
\begin{array}[t]{l}
\boldsymbol{\mu}_g < C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \quad \vee \\
\mathbf{\delta}_w = \varnothing \quad \vee \\
\lVert\boldsymbol{\mu}_\mathbf{s}\rVert < \mathbf{\delta}_w \quad \vee \\
( w = {\small JUMP} \quad \wedge \quad \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) ) \quad \vee \\
( w = {\small JUMPI} \quad \wedge \quad \boldsymbol{\mu}_\mathbf{s}[1] \neq 0 \quad \wedge \\
\quad \boldsymbol{\mu}_\mathbf{s}[0] \notin D(I_\mathbf{b}) ) \quad \vee \\
( w = {\small RETURNDATACOPY} \wedge \\ \quad \boldsymbol{\mu}_{\mathbf{s}}[1] + \boldsymbol{\mu}_{\mathbf{s}}[2] > \lVert\boldsymbol{\mu}_{\mathbf{o}}\rVert) \quad \vee \\
\lVert\boldsymbol{\mu}_\mathbf{s}\rVert - \mathbf{\delta}_w + \mathbf{\alpha}_w > 1024 \vee
 (\neg I_{\mathrm{w}} \wedge W(w, \boldsymbol{\mu}))
\end{array}
\end{equation}
$$

$$
\begin{equation}
W(w, \boldsymbol{\mu}) \equiv \\ 
\begin{array}[t]{l}
w \in \{ {\small CREATE}, {\small SSTORE}, {\small SELFDESTRUCT}\} \ \vee \\
{\small LOG0} \le w \wedge w \le {\small LOG4} \quad \vee \\
w \in \{ {\small CALL}, {\small CALLCODE}\} \wedge \boldsymbol{\mu}_{\mathbf{s}}[2] \neq 0
\end{array}
\end{equation}
$$

$$
\begin{equation}
D(\mathbf{c}) \equiv D_{J}(\mathbf{c}, 0)
\end{equation}
$$

$$
\begin{equation}
D_{J}(\mathbf{c}, i) \equiv \begin{cases}
\{\} & \text{if} \quad i \geqslant |\mathbf{c}|  \\
\{ i \} \cup D_{J}(\mathbf{c}, N(i, \mathbf{c}[i])) & \\
\quad\quad\text{if} \quad \mathbf{c}[i] = {\small JUMPDEST} \\
D_{J}(\mathbf{c}, N(i, \mathbf{c}[i])) & \text{otherwise} \\
\end{cases}
\end{equation}
$$

$$
\begin{equation}
N(i, w) \equiv \begin{cases}
i + w - {\small PUSH1} + 2 & \\
\quad\quad\text{if} \quad w \in [{\small PUSH1}, {\small PUSH32}] \\
i + 1 & \text{otherwise} \end{cases}
\end{equation}
$$

$$
\begin{equation}
H(\boldsymbol{\mu}, I) \equiv \begin{cases}
H_{ {\tiny RETURN}}(\boldsymbol{\mu}) \ \text{if} \quad w \in \{{\small {RETURN}}, {\small REVERT}\} &\\
() \quad\quad\ \text{if} \quad w \in \{ {\small {STOP}}, {\small {SELFDESTRUCT}} \} &\\
\varnothing \quad\quad\ \text{otherwise}&
\end{cases}
\end{equation}
$$

$$
\begin{eqnarray}
O\big((\boldsymbol{\sigma}, \boldsymbol{\mu}, A, I)\big) & \equiv & (\boldsymbol{\sigma}', \boldsymbol{\mu}', A', I) \\
\Delta & \equiv & \mathbf{\alpha}_{w} - \mathbf{\delta}_{w} \\
\lVert\boldsymbol{\mu}'_{\mathbf{s}}\rVert & \equiv & \lVert\boldsymbol{\mu}_{\mathbf{s}}\rVert + \Delta \\
\quad \forall x \in [\mathbf{\alpha}_{w}, \lVert\boldsymbol{\mu}'_{\mathbf{s}}\rVert): \boldsymbol{\mu}'_{\mathbf{s}}[x] & \equiv & \boldsymbol{\mu}_{\mathbf{s}}[x-\Delta]
\end{eqnarray}
$$

$$
\begin{eqnarray}
\quad \boldsymbol{\mu}'_{g} & \equiv & \boldsymbol{\mu}_{g} - C(\boldsymbol{\sigma}, \boldsymbol{\mu}, I) \label{eq:mu_pc}\\
\quad \boldsymbol{\mu}'_{\mathrm{pc}} & \equiv & \begin{cases}
{J_{\text{JUMP}}}(\boldsymbol{\mu}) & \text{if} \quad w = {\small JUMP} \\
{J_{\text{JUMPI}}}(\boldsymbol{\mu}) & \text{if} \quad w = {\small JUMPI} \\
N(\boldsymbol{\mu}_{\mathrm{pc}}, w) & \text{otherwise}
\end{cases}
\end{eqnarray}
$$

$$
\begin{eqnarray}
\boldsymbol{\mu}'_{\mathbf{m}} & \equiv & \boldsymbol{\mu}_{\mathbf{m}} \\
\boldsymbol{\mu}'_{\mathrm{i}} & \equiv & \boldsymbol{\mu}_{\mathrm{i}} \\
A' & \equiv & A \\
\boldsymbol{\sigma}' & \equiv & \boldsymbol{\sigma}
\end{eqnarray}
$$

$$
\begin{eqnarray}
B_{\mathrm{t}} & \equiv & B'_{\mathrm{t}} + B_{\mathrm{d}} \\
B' & \equiv & P(B_{H})
\end{eqnarray}
$$

$$
\begin{equation}
\lVert B_{\mathbf{U}} \rVert \leqslant 2 \bigwedge_{\mathbf{U} \in B_{\mathbf{U}}} {V({\mathbf{U}}})\; \wedge \; k({\mathbf{U}}, P(\mathbf{B}_{\mathbf{H}})_{\mathbf{H}}, 6)
\end{equation}
$$

$$
\begin{equation}
k(U, H, n) \equiv \begin{cases} false & \text{if} \quad n = 0 \\
s(U, H) &\\
\quad \vee \; k(U, P(H)_{H}, n - 1) & \text{otherwise}
\end{cases}
\end{equation}
$$

$$
\begin{equation}
s(U, H) \equiv (P(H) = P(U)\; \wedge \; H \neq U \; \wedge \; U \notin B(H)_{\mathbf{U}})
\end{equation}
$$

$$
\begin{equation}
{B_{H}}_{\mathrm{g}} = {\ell}({\mathbf{R})_{\mathrm{u}}}
\end{equation}
$$

$$
\begin{eqnarray}
\\ \nonumber
\Omega(B, \boldsymbol{\sigma}) & \equiv & \boldsymbol{\sigma}': \boldsymbol{\sigma}' = \boldsymbol{\sigma} \quad \text{except:} \\
\qquad\boldsymbol{\sigma}'[{\mathbf{B}_{H}}_{\mathrm{c}}]_{\mathrm{b}} & = & \boldsymbol{\sigma}[{\mathbf{B}_{H}}_{\mathrm{c}}]_{\mathrm{b}} + \left(1 + \frac{\lVert \mathbf{B}_{\mathbf{U}}\rVert}{32}\right)R_{\mathrm{block}} \\
\qquad\forall_{\mathbf{U} \in \mathbf{B}_{\mathbf{U}}}: \\ \nonumber
\boldsymbol{\sigma}'[\mathbf{U}_{\mathrm{c}}] & = & \begin{cases}
\varnothing &\text{if}\ \boldsymbol{\sigma}[\mathbf{U}_{\mathrm{c}}] = \varnothing\ \wedge\ R = 0 \\
\mathbf{a}' &\text{otherwise}
\end{cases} \\
\mathbf{a}' &\equiv& (\boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{n}}, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{b}} + R, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathbf{s}}, \boldsymbol{\sigma}[U_{\mathrm{c}}]_{\mathrm{c}}) \\
R & \equiv & \left(1 + \frac{1}{8} (U_{\mathrm{i}} - {B_{H}}_{\mathrm{i}})\right) R_{\mathrm{block}}
\end{eqnarray}
$$

$$
\begin{equation}
\text{Let} \quad R_{\mathrm{block}} = 3 \times 10^{18}
\end{equation}
$$

$$
\begin{equation}
\Gamma(B) \equiv \begin{cases}
\boldsymbol{\sigma}_0 kern 10pc \ \text{if} \quad P(B_{H}) = \varnothing \\
\boldsymbol{\sigma}_{\mathrm{i}}: \mathtt{\small {TRIE}}(L_{S}(\boldsymbol{\sigma}_{\mathrm{i}})) = {P(B_{H})_{H}}_{\mathrm{r}} \quad\text{otherwise}
\end{cases}
\end{equation}
$$

$$
\begin{eqnarray}
\Phi(B) & \equiv & B': \quad B' = B^* \quad \text{except:} \\
B'_{\mathrm{n}} & = & n: \quad x \leqslant \frac{2^{256}}{H_{\mathrm{d}}} \\
B'_{\mathrm{m}} & = & m \quad \text{with } (x, m) = \mathtt{PoW}(B^*_{\cancel {n}}, n, \mathbf{d}) \\
B^* & \equiv & B \quad \text{except:} \quad {B^*_{\mathrm{r}}} = {r}({\Pi}(\Gamma(B), B))
\end{eqnarray}
$$

$$
\begin{equation}
\boldsymbol{\sigma}[n] = \begin{cases} {\Gamma}(B) & \text{if} \quad n < 0 \\ 
{\Upsilon}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n]) & \text{otherwise} \end{cases}
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathrm{u}} = \begin{cases} 0 & \text{if} \quad n < 0 \\
\begin{array}[b]{l}
\Upsilon^g(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n])\\ \quad + \mathbf{R}[n-1]_{\mathrm{u}}
\end{array}
 & \text{otherwise} \end{cases}
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathbf{l}} =
\Upsilon^{\mathbf{l}}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n])
\end{equation}
$$

$$
\begin{equation}
\mathbf{R}[n]_{\mathrm{z}} =
\Upsilon^{\mathrm{z}}(\boldsymbol{\sigma}[n - 1], B_{\mathbf{T}}[n])
\end{equation}
$$

$$
\begin{equation}
\Pi(\boldsymbol{\sigma}, B) \equiv {\Omega}(B, \ell(\boldsymbol{\sigma}))
\end{equation}
$$

$$
\begin{equation}
m = {H_{\mathrm{m}}} \quad \wedge \quad n \leqslant \frac{2^{256}}{H_{\mathrm{d}}} \quad \text{with} \quad (m, n) = \mathtt{PoW}({H_{\cancel {n}}}, H_{\mathrm{n}}, \mathbf{d})
\end{equation}
$$