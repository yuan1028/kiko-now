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