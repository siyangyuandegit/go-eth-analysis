## 分析以太坊源码

## 一笔交易

交易的入口

```go
位于core/state_processor.go中的Process函数
该函数会返回收据、日志、使用的gas，如果gas不够，会返回error
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
  ...
  for i, tx := range block.Transactions() {
		// 将tx转化为消息
		msg, err := TransactionToMessage(tx, types.MakeSigner(p.config, header.Number), header.BaseFee)
		if err != nil {
			return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
		}
		statedb.SetTxContext(tx.Hash(), i)
		// 开始执行交易，如果某个交易失败，revert
		receipt, err := applyTransaction(msg, p.config, gp, statedb, blockNumber, blockHash, tx, usedGas, vmenv)
		if err != nil {
			return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
		}
		receipts = append(receipts, receipt)
		allLogs = append(allLogs, receipt.Logs...)
	}
...
```

接下来看applyTransaction函数

```go
func applyTransaction(msg *Message, config *params.ChainConfig, gp *GasPool, statedb *state.StateDB, blockNumber *big.Int, blockHash common.Hash, tx *types.Transaction, usedGas *uint64, evm *vm.EVM) (*types.Receipt, error) {
	// Create a new context to be used in the EVM environment.
	txContext := NewEVMTxContext(msg)
	evm.Reset(txContext, statedb)

	// Apply the transaction to the current state (included in the env).
	// 这里是真正执行交易的逻辑
	result, err := ApplyMessage(evm, msg, gp)
	if err != nil {
		return nil, err
	}
...
}
```

然后是ApplyMessage

```go
func ApplyMessage(evm *vm.EVM, msg *Message, gp *GasPool) (*ExecutionResult, error) {
		// 这里的NewStateTransition会返回一个StateTransition的指针，然后执行该对象的TransitionDb函数
	return NewStateTransition(evm, msg, gp).TransitionDb()
}
```

TransitionDb函数会真正开始执行交易，将应用当前msg改变state，并返回evm执行后的结果(used gas包含返还的gas，returndata 从evm返回的数据，concrete execution error具体的执行错误：ErrOutOfGas, ErrExecutionReverted) 如果遇到共识问题，直接返回错误 evm执行结果为空

```go
func (st *StateTransition) TransitionDb() (*ExecutionResult, error) {
  ...
  这里是一些共识的检查机制，以及gas的消耗返还等，具体看注释
  ...
  // 分为创建合约和普通调用
	if contractCreation {
		ret, _, st.gasRemaining, vmerr = st.evm.Create(sender, msg.Data, st.gasRemaining, msg.Value)
	} else {
		// Increment the nonce for the next transaction
		// 将账户的nonce+1
		st.state.SetNonce(msg.From, st.state.GetNonce(sender.Address())+1)
		ret, st.gasRemaining, vmerr = st.evm.Call(sender, st.to(), msg.Data, st.gasRemaining, msg.Value)
	}
}
```

到这里，就进入了虚拟机中，开始call、callcode、staticcall、delegatecall、create等一些列操作

```go
// 在这里说明一下call、callcode、staticcall、delegatecall的区别
// 要说明这个区别就要先看Contract的结构
type Contract struct {
	CallerAddress common.Address // msg.sender
	caller        ContractRef  // 调用该合约的上一个调用者
	self          ContractRef  // 改变谁的存储
	...
}
// 这四个操作里边，只有call可以真正的发送eth，callcode虽然接受value，但是并不会真正发送
// call一个合约，修改的是合约的上下文

func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
  ...
  // 这里只看引起他们不同的主要差别
  code := evm.StateDB.GetCode(addr)
		if len(code) == 0 {
			ret, err = nil, nil // gas is unchanged
		} else {
			addrCopy := addr
			// 生成一个contract对象，caller是入参的caller，contract object是被调用者的引用
			// 这里返回的合约对象是addr的合约
			contract := NewContract(caller, AccountRef(addrCopy), value, gas)
			// 设置该合约对象的的code为被调用者的code，state也是被调用合约的state
			contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), code)
			ret, err = evm.interpreter.Run(contract, input, false)
			gas = contract.Gas
		}
  ...
}

func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
	...
  	addrCopy := addr
		// 这里就跟跟call的区别，这里的contract对象，caller还是caller，但是合约对象换成了caller的合约
		// 在newcontract中，返回的contract object是第二个参数，所以这里返回的是caller的合约本身
		// 也就是说callcode用的是addr的code，但是改变的是caller合约的状态
		contract := NewContract(caller, AccountRef(caller.Address()), value, gas)
		contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), evm.StateDB.GetCode(addrCopy))
		ret, err = evm.interpreter.Run(contract, input, false)
		gas = contract.Gas
  ...
}

func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
	...
  	addrCopy := addr
		// 这里跟callcode不同的是调用了AsDelegate函数，这个函数将合约的calladdress和value设置成上一级的calladdress和value
		// 说人话，就是callcode的msg.sender是上一级调用者，delegatecall的msg.sender是最初的调用者
		contract := NewContract(caller, AccountRef(caller.Address()), nil, gas).AsDelegate()
		contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), evm.StateDB.GetCode(addrCopy))
		ret, err = evm.interpreter.Run(contract, input, false)
		gas = contract.Gas
  ...
}


func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
	...
  	addrCopy := addr
		contract := NewContract(caller, AccountRef(addrCopy), new(big.Int), gas)
		contract.SetCallCode(&addrCopy, evm.StateDB.GetCodeHash(addrCopy), evm.StateDB.GetCode(addrCopy))
		// 这里设置了true，意味readonly，如果该函数修改了storage会revert
		ret, err = evm.interpreter.Run(contract, input, true)
		gas = contract.Gas
  ...
}
```



然后就是run

```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
	// Increment the call depth which is restricted to 1024
	// 整个以太坊代码中只有这里做了evm.depth++的操作，然后限制了执行run的次数不能超过1024
	in.evm.depth++
	defer func() { in.evm.depth-- }()
  ...
  	// 如果有的话，开始执行opcode，遇到call家四兄弟或者create/create2，depth就会+1，执行操作的时候会判断depth是否超过1024，超过会revert
 	... 
}
```

