---
title: EthHash PoW Consensus - Block building and the filling up of transactions
date: 2026-09-28
categories: [ethereum, consensus]
tags: [ethhash, geth, mining, pow, blockbuilding]    
pin: true
description: The complete process on how a block gets built on geth.
---

In my previous post <https://pvnotpv.github.io/posts/ethhash/>, it was regarding the algorithmic part of EthHash, but the whole process of creating a new block, adding transactions to it, and how the entire process works is pretty much a league of its own, and we'd be going deep into that.

I'm kind of really excited to write this post because this is exactly where we'd be going deep into the actual blockchain. I mean, we'd be going to see the actual incrementing of the block number from the parent and how new transactions are added to the chain!

<img width="1200" height="542" alt="image" src="https://github.com/user-attachments/assets/1c9aaf22-c00c-42d7-8f1a-b642a7fdedc1" />

So here's a mapping I made of the entire mining process on geth:

<img width="2523" height="1529" alt="image" src="https://github.com/user-attachments/assets/13b00d2c-5c2d-49ee-9b58-110c36a1f970" />

To get a perfect version of Geth that doesn't include many of the beacon chain methods, make sure to go with v1.10.14.

When going through such a huge codebase like Geth, with multiple packages like networking, tries, txpool, and state dbs, you'd wonder how the whole thing is connected, even more importantly, which is that one core thing that drives all of this. In simple words, how does the blockchain keep on running and keep producing blocks? This is exactly where the miner package comes along, and the whole process is really interesting.

Now the above diagram may seems like a lot but we'd be going into it section by section ;)

<img width="335" height="145" alt="image" src="https://github.com/user-attachments/assets/e2ff4031-4fdc-4f10-95a5-bbedd1d7de12" />

Let's directly jump into the worker object which is the main struct which implements the whole mining process:

### Worker

<img width="1190" height="915" alt="image" src="https://github.com/user-attachments/assets/4a70e8c2-e0c1-4e15-ab4c-cfc7b395cbef" />

The worker is what implements the multiple loops, which'd keep on listening for events, transactions, mining, and creating new blocks. And the different channels on how these loops communicate with each other—I was really having a hard time figuring out the whole thing due to the multiple channels and the reason why I started making diagrams to not confuse with the whole thing.

```go
// worker is the main object which takes care of submitting new work to consensus engine
// and gathering the sealing result.
type worker struct {
	config      *Config
	chainConfig *params.ChainConfig
	engine      consensus.Engine
	eth         Backend
	chain       *core.BlockChain
	merger      *consensus.Merger

	// Feeds
	pendingLogsFeed event.Feed

	// Subscriptions
	mux          *event.TypeMux
	txsCh        chan core.NewTxsEvent
	txsSub       event.Subscription
	chainHeadCh  chan core.ChainHeadEvent
	chainHeadSub event.Subscription
	chainSideCh  chan core.ChainSideEvent
	chainSideSub event.Subscription

	// Channels
	newWorkCh          chan *newWorkReq
	taskCh             chan *task
	resultCh           chan *types.Block
	startCh            chan struct{}
	exitCh             chan struct{}
	resubmitIntervalCh chan time.Duration
	resubmitAdjustCh   chan *intervalAdjust
```

worker has a method named newWorker() which starts all the loops for the worker, which is of course a goroutine of its own and keeps on running doing certain tasks.

```go
	// Subscribe NewTxsEvent for tx pool
	worker.txsSub = eth.TxPool().SubscribeNewTxsEvent(worker.txsCh)
	// Subscribe events for blockchain
	worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
	worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)

	// Sanitize recommit interval if the user-specified one is too short.
	recommit := worker.config.Recommit
	if recommit < minRecommitInterval {
		log.Warn("Sanitizing miner recommit interval", "provided", recommit, "updated", minRecommitInterval)
		recommit = minRecommitInterval
	}

	worker.wg.Add(4)
	go worker.mainLoop()
	go worker.newWorkLoop(recommit)
	go worker.resultLoop()
	go worker.taskLoop()

	// Submit first work to initialize pending state.
	if init {
		worker.startCh <- struct{}{}
	}
	return worker

```

It subscribes to new transactions, head block events, and the side chain events are sent during reorgs.

<img width="833" height="554" alt="image" src="https://github.com/user-attachments/assets/80ed7875-c876-405d-a060-7a6bbf10a343" />

So there are 4 loops doing their own thing, and the best loop to start with is the workLoop()

### workLoop

```go
// newWorkLoop is a standalone goroutine to submit new mining work upon received events.
func (w *worker) newWorkLoop(recommit time.Duration) {
	defer w.wg.Done()
	var (
...

	// commit aborts in-flight transaction execution with given signal and resubmits a new one.
	commit := func(noempty bool, s int32) {
		if interrupt != nil {
			atomic.StoreInt32(interrupt, s)
		}
		interrupt = new(int32)
		select {
		case w.newWorkCh <- &newWorkReq{interrupt: interrupt, noempty: noempty, timestamp: timestamp}:
		case <-w.exitCh:
			return
		}
		timer.Reset(recommit)
		atomic.StoreInt32(&w.newTxs, 0)
	}

...


	for {
		select {
		case <-w.startCh:
			clearPending(w.chain.CurrentBlock().NumberU64())
			timestamp = time.Now().Unix()
			commit(false, commitInterruptNewHead)

		case head := <-w.chainHeadCh:
			clearPending(head.Block.NumberU64())
			timestamp = time.Now().Unix()
			commit(false, commitInterruptNewHead)

		case <-timer.C:
			// If mining is running resubmit a new work cycle periodically to pull in
			// higher priced transactions. Disable this overhead for pending blocks.
			if w.isRunning() && (w.chainConfig.Clique == nil || w.chainConfig.Clique.Period > 0) {
				// Short circuit if no new transaction arrives.
				if atomic.LoadInt32(&w.newTxs) == 0 {
					timer.Reset(recommit)
					continue
				}
				commit(true, commitInterruptResubmit)
			}


```

(I've removed certain stuffs to focus on the main parts.)

<img width="653" height="498" alt="image" src="https://github.com/user-attachments/assets/e0bc760f-f07a-4eed-bb9a-296397daaea2" />

From the code we can see that workLoop() is what does action based on events being received; when a new head block is received, it creates a new work request, or when something is sent to the start channel, a new work request is being sent.

At the end of the newWorker() method, we can see these lines:

```go
	// Submit first work to initialize pending state.
	if init {
		worker.startCh <- struct{}{}
	}
```

So when a new worker is created , if init is set to true , the whole process starts right away.

<img width="1315" height="434" alt="image" src="https://github.com/user-attachments/assets/06cc78f8-a1a4-4b2b-a7d1-0c24a987acfc" />

```go
// newWorkReq represents a request for new sealing work submitting with relative interrupt notifier.
type newWorkReq struct {
	interrupt *int32
	noempty   bool
	timestamp int64
}
```
With the interrupts being:

```go
const (
	commitInterruptNone int32 = iota
	commitInterruptNewHead
	commitInterruptResubmit
)
```

The struct would start making a lot of sense once we get to the sealing part.

Currently the workLoop has received a request in the start channel and it sends a new work to the "newWork" channel

```go
		case w.newWorkCh <- &newWorkReq{interrupt: interrupt, noempty: noempty, timestamp: timestamp}:

```

### mainLoop

The mainLoop is what listens to the "newWork channel":

```go
// mainLoop is a standalone goroutine to regenerate the sealing task based on the received event.
func (w *worker) mainLoop() {
	defer w.wg.Done()
	defer w.txsSub.Unsubscribe()
	defer w.chainHeadSub.Unsubscribe()
	defer w.chainSideSub.Unsubscribe()
	defer func() {
		if w.current != nil && w.current.state != nil {
			w.current.state.StopPrefetcher()
		}
	}()

	for {
		select {
		case req := <-w.newWorkCh:
			w.commitNewWork(req.interrupt, req.noempty, req.timestamp)

```

<img width="768" height="436" alt="image" src="https://github.com/user-attachments/assets/e9bf5879-6580-4f40-a632-f8cdbd22150a" />

### Header creation and transactions

The commitNewWork function is called with the request.

This is pretty much the most important function of the whole process.

```go
// commitNewWork generates several new sealing tasks based on the parent block.
func (w *worker) commitNewWork(interrupt *int32, noempty bool, timestamp int64) {
	w.mu.RLock()
	defer w.mu.RUnlock()

	tstart := time.Now()
	parent := w.chain.CurrentBlock()

	if parent.Time() >= uint64(timestamp) {
		timestamp = int64(parent.Time() + 1)
	}
	num := parent.Number()
	header := &types.Header{
		ParentHash: parent.Hash(),
		Number:     num.Add(num, common.Big1),
		GasLimit:   core.CalcGasLimit(parent.GasLimit(), w.config.GasCeil),
		Extra:      w.extra,
		Time:       uint64(timestamp),
	}
```

Here we can see a header being created with the number being incremented from the parent! Idk how insane that sounds, but damn.

The Prepare method of the engine being called here, which sets the difficulty of the block for ethash:

```go
	if err := w.engine.Prepare(w.chain, header); err != nil {
```

Then we can see the pending transactions being taken in the function:

```go
	// Fill the block with all available pending transactions.
	pending := w.eth.TxPool().Pending(true)
```

```go
	// Split the pending transactions into locals and remotes
	localTxs, remoteTxs := make(map[common.Address]types.Transactions), pending
	for _, account := range w.eth.TxPool().Locals() {
		if txs := remoteTxs[account]; len(txs) > 0 {
			delete(remoteTxs, account)
			localTxs[account] = txs
		}
	}

```

Local transactions being the txs sent from the running node.

```go
	if len(remoteTxs) > 0 {
		txs := types.NewTransactionsByPriceAndNonce(w.current.signer, remoteTxs, header.BaseFee)
		if w.commitTransactions(txs, w.coinbase, interrupt) {
			return
		}
	}
```

<img width="633" height="569" alt="image" src="https://github.com/user-attachments/assets/b484eab5-c023-40f2-a650-47fcf3c12d0a" />

commitTransactions function which iteratively calls commitTransaction for each function wherein the actual execution takes place.

### Transaction execution

```go

func (w *worker) commitTransactions(txs *types.TransactionsByPriceAndNonce, coinbase common.Address, interrupt *int32) bool {
	// Short circuit if current is nil
	if w.current == nil {
		return true
	}

```

```go
func (w *worker) commitTransactions(txs *types.TransactionsByPriceAndNonce, coinbase common.Address, interrupt *int32) bool {
	// Short circuit if current is nil
	if w.current == nil {
		return true
		
		...
		
	logs, err := w.commitTransaction(tx, coinbase)
```

<img width="775" height="276" alt="image" src="https://github.com/user-attachments/assets/cc456adf-05ea-4c17-84fe-c52798e029aa" />

```go

func (w *worker) commitTransaction(tx *types.Transaction, coinbase common.Address) ([]*types.Log, error) {
	snap := w.current.state.Snapshot()

	receipt, err := core.ApplyTransaction(w.chainConfig, w.chain, &coinbase, w.current.gasPool, w.current.state, w.current.header, tx, &w.current.header.GasUsed, *w.chain.GetVMConfig())
	if err != nil {
		w.current.state.RevertToSnapshot(snap)
		return nil, err
	}
	w.current.txs = append(w.current.txs, tx)
	w.current.receipts = append(w.current.receipts, receipt)

	return receipt.Logs, nil
}
```

The ApplyTransaction function is what creates the evm context and runs the transaction.


```go
// ApplyTransaction attempts to apply a transaction to the given state database
// and uses the input parameters for its environment. It returns the receipt
// for the transaction, gas used and an error if the transaction failed,
// indicating the block was invalid.
func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, error) {
	msg, err := tx.AsMessage(types.MakeSigner(config, header.Number), header.BaseFee)
	if err != nil {
		return nil, err
	}
	// Create a new context to be used in the EVM environment
	blockContext := NewEVMBlockContext(header, bc, author)
	vmenv := vm.NewEVM(blockContext, vm.TxContext{}, statedb, config, cfg)
	return applyTransaction(msg, config, bc, author, gp, statedb, header.Number, header.Hash(), tx, usedGas, vmenv)
}
```

Then in the commitNewWork() function we can see commit() being called:

### Creation of a block

<img width="628" height="470" alt="image" src="https://github.com/user-attachments/assets/e728a613-b18c-4e90-bdd5-7b0c6af5dc7f" />

```go
// commit runs any post-transaction state modifications, assembles the final block
// and commits new work if consensus engine is running.
func (w *worker) commit(uncles []*types.Header, interval func(), update bool, start time.Time) error {
	// Deep copy receipts here to avoid interaction between different tasks.
	receipts := copyReceipts(w.current.receipts)
	s := w.current.state.Copy()
	block, err := w.engine.FinalizeAndAssemble(w.chain, w.current.header, s, w.current.txs, uncles, receipts)
	if err != nil {
		return err
	}
	
	...
	
	}
		select {
		case w.taskCh <- &task{receipts: receipts, state: s, block: block, createdAt: time.Now()}:
			w.unconfirmed.Shift(block.NumberU64() - 1)
			log.Info("Commit new mining work", "number", block.Number(), "sealhash", w.engine.SealHash(block.Header()),
				"uncles", len(uncles), "txs", w.current.tcount,
				"gas", block.GasUsed(), "fees", totalFees(block, receipts),
				"elapsed", common.PrettyDuration(time.Since(start)))

		case <-w.exitCh:
			log.Info("Worker has exited")
		}
	}


```

This is where the actual block creation takes place, for ethhash:

```go
// Finalize implements consensus.Engine, accumulating the block and uncle rewards,
// setting the final state on the header
func (ethash *Ethash) Finalize(chain consensus.ChainHeaderReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header) {
	// Accumulate any block and uncle rewards and commit the final state root
	accumulateRewards(chain.Config(), state, header, uncles)
	header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))
}

// FinalizeAndAssemble implements consensus.Engine, accumulating the block and
// uncle rewards, setting the final state and assembling the block.
func (ethash *Ethash) FinalizeAndAssemble(chain consensus.ChainHeaderReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
	// Finalize block
	ethash.Finalize(chain, header, state, txs, uncles)

	// Header seems complete, assemble into a block and return
	return types.NewBlock(header, txs, uncles, receipts, trie.NewStackTrie(nil)), nil
}
```

The state root is being set and a new block is created with the header and the transactions! 

So basically we've created a block now!

Now in commit() we can see this line:

```go
		case w.taskCh <- &task{receipts: receipts, state: s, block: block, createdAt: time.Now()}:
```

Where a new task is created with the block and sent to the task channel.

### taskLoop

Up next we have the task loop which listens to the task channel, where the actual mining process happens!

<img width="620" height="546" alt="image" src="https://github.com/user-attachments/assets/71c341be-abb7-467c-867a-cf0cf03c1678" />

```go

// taskLoop is a standalone goroutine to fetch sealing task from the generator and
// push them to consensus engine.
func (w *worker) taskLoop() {
	defer w.wg.Done()
	var (
		stopCh chan struct{}
		prev   common.Hash
	)
...

	for {
		select {
		case task := <-w.taskCh:
			if w.newTaskHook != nil {
				w.newTaskHook(task)
			}
			// Reject duplicate sealing work due to resubmitting.
			sealHash := w.engine.SealHash(task.block.Header())
			if sealHash == prev {
				continue
			}
			// Interrupt previous sealing operation
			interrupt()
			stopCh, prev = make(chan struct{}), sealHash

			if w.skipSealHook != nil && w.skipSealHook(task) {
				continue
			}
			w.pendingMu.Lock()
			w.pendingTasks[sealHash] = task
			w.pendingMu.Unlock()

			if err := w.engine.Seal(w.chain, task.block, w.resultCh, stopCh); err != nil {
				log.Warn("Block sealing failed", "err", err)
				w.pendingMu.Lock()
				delete(w.pendingTasks, sealHash)
				w.pendingMu.Unlock()
			}
		case <-w.exitCh:
			interrupt()
			return
		}
```

```go
			if err := w.engine.Seal(w.chain, task.block, w.resultCh, stopCh); err != nil {
```

The sealing method is where the whole block mining process happens, which I've explained in my previous post.

<https://pvnotpv.github.io/posts/ethhash/>

### Mining in action

Now let's see the actual mining process in action! Here I've created a private network:

```bash
pv@arch ~/t/go-ethereum ((v1.10.14))> build/bin/geth --datadir .private-net/pow-node-clean --networkid 1337 --nodiscover --http --http.addr 127.0.0.1 --http.port 8545 --http.api eth,net,web3,miner --mine --miner.threads 1 --miner.etherbase 0x3D27412aC6D1bB84E9621bcE0D95207f3dC2214C
```

First we can see the generation of the DAG:

<img width="1044" height="488" alt="image" src="https://github.com/user-attachments/assets/1b411d13-6e14-4c5c-9f77-fc2a9d5f4784" />

Then the whole mining process beings:

<img width="1424" height="599" alt="image" src="https://github.com/user-attachments/assets/1d829930-01ef-4e5b-b537-631c02b5d53f" />

```bash
pv@arch ~> curl -s -X POST http://127.0.0.1:8545 \
                 -H 'Content-Type: application/json' \
                 --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
{"jsonrpc":"2.0","id":1,"result":"0x19"}
pv@arch ~>

```


