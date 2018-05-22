
---
layout: post
title: Hyperledger fabric 1.0 代码解析 之 verify and commit block
tags:
  - Fabric
---
## peer节点对区块进行验证
1.peer从orderer那里得到block之后会首先通过VerifyBlock进行验证。该部分主要是从block的层面进行的验证，并不会对其中的交易进行验证。

```go
// VerifyBlock returns nil if the block is properly signed, and the claimed seqNum is the
// sequence number that the block's header contains.
// else returns error
func (s *mspMessageCryptoService) VerifyBlock(chainID common.ChainID, seqNum uint64, signedBlock []byte) error {
    // - Convert signedBlock to common.Block.
    //基础验证
    block, err := utils.GetBlockFromBlockBytes(signedBlock)
    if err != nil {
        return fmt.Errorf("Failed unmarshalling block bytes on channel [%s]: [%s]", chainID, err)
    }
    //header不为空
    if block.Header == nil {
        return fmt.Errorf("Invalid Block on channel [%s]. Header must be different from nil.", chainID)
    }
    //blockNum是否鱼seqNum一致
    blockSeqNum := block.Header.Number
    if seqNum != blockSeqNum {
        return fmt.Errorf("Claimed seqNum is [%d] but actual seqNum inside block is [%d]", seqNum, blockSeqNum)
    }
    //chainId是否正确
    // - Extract channelID and compare with chainID
    channelID, err := utils.GetChainIDFromBlock(block)
    if err != nil {
        return fmt.Errorf("Failed getting channel id from block with id [%d] on channel [%s]: [%s]", block.Header.Number, chainID, err)
    }

    if channelID != string(chainID) {
        return fmt.Errorf("Invalid block's channel id. Expected [%s]. Given [%s]", chainID, channelID)
    }

    // - Unmarshal medatada
    if block.Metadata == nil || len(block.Metadata.Metadata) == 0 {
        return fmt.Errorf("Block with id [%d] on channel [%s] does not have metadata. Block not valid.", block.Header.Number, chainID)
    }
    //取出metadata数据
    metadata, err := utils.GetMetadataFromBlock(block, pcommon.BlockMetadataIndex_SIGNATURES)
    if err != nil {
        return fmt.Errorf("Failed unmarshalling medatata for signatures [%s]", err)
    }
    //Data.Hash是否与Header中纪录的DataHash一致
    // - Verify that Header.DataHash is equal to the hash of block.Data
    // This is to ensure that the header is consistent with the data carried by this block
    if !bytes.Equal(block.Data.Hash(), block.Header.DataHash) {
        return fmt.Errorf("Header.DataHash is different from Hash(block.Data) for block with id [%d] on channel [%s]", block.Header.Number, chainID)
    }

    // - Get Policy for block validation

    // Get the policy manager for channelID
    // 验证channel的policy是否满足
    cpm, ok := s.channelPolicyManagerGetter.Manager(channelID)
    if cpm == nil {
        return fmt.Errorf("Could not acquire policy manager for channel %s", channelID)
    }
    // ok is true if it was the manager requested, or false if it is the default manager
    mcsLogger.Debugf("Got policy manager for channel [%s] with flag [%s]", channelID, ok)

    // Get block validation policy
    policy, ok := cpm.GetPolicy(policies.BlockValidation)
    // ok is true if it was the policy requested, or false if it is the default policy
    mcsLogger.Debugf("Got block validation policy for channel [%s] with flag [%s]", channelID, ok)

    // - Prepare SignedData
    signatureSet := []*pcommon.SignedData{}
    for _, metadataSignature := range metadata.Signatures {
        shdr, err := utils.GetSignatureHeader(metadataSignature.SignatureHeader)
        if err != nil {
            return fmt.Errorf("Failed unmarshalling signature header for block with id [%d] on channel [%s]: [%s]", block.Header.Number, chainID, err)
        }
        signatureSet = append(
            signatureSet,
            &pcommon.SignedData{
                Identity:  shdr.Creator,
                Data:      util.ConcatenateBytes(metadata.Value, metadataSignature.SignatureHeader, block.Header.Bytes()),
                Signature: metadataSignature.Signature,
            },
        )
    }

    // - Evaluate policy
    return policy.Evaluate(signatureSet)
}

```

## peer端进行区块提交
经过第一步验证的区块，会进入到提交区块的阶段。
1.commitBlock主要调用committer的Commit方法提交区块。

```go
func (s *GossipStateProviderImpl) commitBlock(block *common.Block) error {

    if err := s.committer.Commit(block); err != nil {
        logger.Errorf("Got error while committing(%s)", err)
        return err
    }

    // Update ledger level within node metadata
    nodeMetastate := NewNodeMetastate(block.Header.Number)
    // Decode nodeMetastate to byte array
    b, err := nodeMetastate.Bytes()
    if err == nil {
        s.gossip.UpdateChannelMetadata(b, common2.ChainID(s.chainID))
    } else {

        logger.Errorf("Unable to serialize node meta nodeMetastate, error = %s", err)
    }

    logger.Warningf("Channel [%s]: Created block [%d] with %d transaction(s)",
        s.chainID, block.Header.Number, len(block.Data.Data))

    return nil
    /*
        s.asynCommitBlock(block,seqNum)
        return nil
    */
}
```
2.commiter主要是调用Validate对区块进行进一步验证，主要是调用lscc验证。如果是configBlock，需要更新CSCC。然后通过调用ledger的Commit进行真正的写块和写数据库的操作。最后调用SendProducerBlockEvent方法，向客户端以及SDK等发送新区块的事件。

```go
// Commit commits block to into the ledger
// Note, it is important that this always be called serially
func (lc *LedgerCommitter) Commit(block *common.Block) error {

    // Validate and mark invalid transactions
    logger.Debug("Validating block")
    //[hzyangwenlongTODO] validate
    if err := lc.validator.Validate(block); err != nil {
        return err
    }

    // Updating CSCC with new configuration block
    if utils.IsConfigBlock(block) {
        logger.Debug("Received configuration update, calling CSCC ConfigUpdate")
        if err := lc.eventer(block); err != nil {
            return fmt.Errorf("Could not update CSCC with new configuration update due to %s", err)
        }
    }

    if err := lc.ledger.Commit(block); err != nil {
        return err
    }

    // send block event *after* the block has been committed
    if err := producer.SendProducerBlockEvent(block); err != nil {
        logger.Errorf("Error publishing block %d, because: %v", block.Header.Number, err)
    }
    go write2Log(block)
    return nil
}
```
2.1 Validate过程，主要是调用ValidateTransaction进行基本的格式检查。格式检查正确后调用VSCCValidateTx进行背书验证，检查交易是否符合chaincode初始化是要求的背书条件。
```go
func (v *txValidator) Validate(block *common.Block) error {
    logger.Debug("START Block Validation")
    defer logger.Debug("END Block Validation")
    // Initialize trans as valid here, then set invalidation reason code upon invalidation below
    txsfltr := ledgerUtil.NewTxValidationFlags(len(block.Data.Data))
    // txsChaincodeNames records all the invoked chaincodes by tx in a block
    txsChaincodeNames := make(map[int]*sysccprovider.ChaincodeInstance)
    // upgradedChaincodes records all the chaincodes that are upgrded in a block
    txsUpgradedChaincodes := make(map[int]*sysccprovider.ChaincodeInstance)

    for tIdx, d := range block.Data.Data {
        if d != nil {
            if env, err := utils.GetEnvelopeFromBlock(d); err != nil {
                logger.Warningf("Error getting tx from block(%s)", err)
                txsfltr.SetFlag(tIdx, peer.TxValidationCode_INVALID_OTHER_REASON)
            } else if env != nil {
                // validate the transaction: here we check that the transaction
                // is properly formed, properly signed and that the security
                // chain binding proposal to endorsements to tx holds. We do
                // NOT check the validity of endorsements, though. That's a
                // job for VSCC below
                logger.Debug("Validating transaction peer.ValidateTransaction()")
                var payload *common.Payload
                var err error
                var txResult peer.TxValidationCode
                //对交易进行一些格式上的检查，比方说header是否正确，creator签名是否正确，交易类型是否正确，交易的参数等
                if payload, txResult = validation.ValidateTransaction(env); txResult != peer.TxValidationCode_VALID {
                    logger.Errorf("Invalid transaction with index %d", tIdx)
                    txsfltr.SetFlag(tIdx, txResult)
                    continue
                }

                chdr, err := utils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
                if err != nil {
                    logger.Warningf("Could not unmarshal channel header, err %s, skipping", err)
                    txsfltr.SetFlag(tIdx, peer.TxValidationCode_INVALID_OTHER_REASON)
                    continue
                }

                channel := chdr.ChannelId
                logger.Debugf("Transaction is for chain %s", channel)

                if !v.chainExists(channel) {
                    logger.Errorf("Dropping transaction for non-existent chain %s", channel)
                    txsfltr.SetFlag(tIdx, peer.TxValidationCode_TARGET_CHAIN_NOT_FOUND)
                    continue
                }

                if common.HeaderType(chdr.Type) == common.HeaderType_ENDORSER_TRANSACTION {
                    // Check duplicate transactions
                    txID := chdr.TxId
                    if _, err := v.support.Ledger().GetTransactionByID(txID); err == nil {
                        logger.Error("Duplicate transaction found, ", txID, ", skipping")
                        txsfltr.SetFlag(tIdx, peer.TxValidationCode_DUPLICATE_TXID)
                        continue
                    }

                    // Validate tx with vscc and policy
                    logger.Debug("Validating transaction vscc tx validate")

                    err, cde := v.vscc.VSCCValidateTx(payload, d, env)
                    if err != nil {

                        txID := txID
                        logger.Errorf("VSCCValidateTx for transaction txId = %s returned error %s", txID, err)
                        txsfltr.SetFlag(tIdx, cde)
                        continue
                    }

                    invokeCC, upgradeCC, err := v.getTxCCInstance(payload)
                    if err != nil {
                        logger.Errorf("Get chaincode instance from transaction txId = %s returned error %s", txID, err)
                        txsfltr.SetFlag(tIdx, peer.TxValidationCode_INVALID_OTHER_REASON)
                        continue
                    }
                    txsChaincodeNames[tIdx] = invokeCC
                    if upgradeCC != nil {
                        logger.Infof("Find chaincode upgrade transaction for chaincode %s on chain %s with new version %s", upgradeCC.ChaincodeName, upgradeCC.ChainID, upgradeCC.ChaincodeVersion)
                        txsUpgradedChaincodes[tIdx] = upgradeCC
                    }

                } else if common.HeaderType(chdr.Type) == common.HeaderType_CONFIG {
                    configEnvelope, err := configtx.UnmarshalConfigEnvelope(payload.Data)
                    if err != nil {
                        err := fmt.Errorf("Error unmarshaling config which passed initial validity checks: %s", err)
                        logger.Critical(err)
                        return err
                    }

                    if err := v.support.Apply(configEnvelope); err != nil {
                        err := fmt.Errorf("Error validating config which passed initial validity checks: %s", err)
                        logger.Critical(err)
                        return err
                    }
                    logger.Debugf("config transaction received for chain %s", channel)
                } else {
                    logger.Warningf("Unknown transaction type [%s] in block number [%d] transaction index [%d]",
                        common.HeaderType(chdr.Type), block.Header.Number, tIdx)
                    txsfltr.SetFlag(tIdx, peer.TxValidationCode_UNKNOWN_TX_TYPE)
                    continue
                }

                if _, err := proto.Marshal(env); err != nil {
                    logger.Warningf("Cannot marshal transaction due to %s", err)
                    txsfltr.SetFlag(tIdx, peer.TxValidationCode_MARSHAL_TX_ERROR)
                    continue
                }
                // Succeeded to pass down here, transaction is valid
                txsfltr.SetFlag(tIdx, peer.TxValidationCode_VALID)
            } else {
                logger.Warning("Nil tx from block")
                txsfltr.SetFlag(tIdx, peer.TxValidationCode_NIL_ENVELOPE)
            }
        }
    }
    // Initialize metadata structure
    utils.InitBlockMetadata(block)

    block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsfltr

    return nil
}
```

2.2 提交区块，包括写入区块数据，写入statedb，写入hsitorydb。

```go
// Commit commits the valid block (returned in the method RemoveInvalidTransactionsAndPrepare) and related state changes
func (l *kvLedger) Commit(block *common.Block) error {
    var err error
    blockNo := block.Header.Number
    //判断是否有读写集冲突，准备好需要写statedb的内容
    logger.Debugf("Channel [%s]: Validating block [%d]", l.ledgerID, blockNo)
    err = l.txtmgmt.ValidateAndPrepare(block, true)
    if err != nil {
        return err
    }
    //写入到区块中
    logger.Debugf("Channel [%s]: Committing block [%d] to storage", l.ledgerID, blockNo)
    if err = l.blockStore.AddBlock(block); err != nil {
        return err
    }
    logger.Infof("Channel [%s]: Created block [%d] with %d transaction(s)", l.ledgerID, block.Header.Number, len(block.Data.Data))

    logger.Debugf("Channel [%s]: Committing block [%d] transactions to state database", l.ledgerID, blockNo)
    // statedb更新
    if err = l.txtmgmt.Commit(); err != nil {
        panic(fmt.Errorf(`Error during commit to txmgr:%s`, err))
    }

    // History database could be written in parallel with state and/or async as a future optimization
    // 更新历史
    if ledgerconfig.IsHistoryDBEnabled() {
        logger.Debugf("Channel [%s]: Committing block [%d] transactions to history database", l.ledgerID, blockNo)
        if err := l.historyDB.Commit(block); err != nil {
            panic(fmt.Errorf(`Error during commit to history db:%s`, err))
        }
    }

    return nil
}
```
2.2.1验证区块中的每条交易，并确定有哪些是要写入更新的。

```go
// ValidateAndPrepare implements method in interface `txmgmt.TxMgr`
func (txmgr *LockBasedTxMgr) ValidateAndPrepare(block *common.Block, doMVCCValidation bool) error {
    logger.Debugf("Validating new block with num trans = [%d]", len(block.Data.Data))
    batch, err := txmgr.validator.ValidateAndPrepareBatch(block, doMVCCValidation)
    if err != nil {
        return err
    }
    txmgr.currentBlock = block
    txmgr.batch = batch
    return err
}
```
验证每条交易的读写集，确定是否有读写集冲突。如果有冲突则该条交易会被标记invalid。

```go
// ValidateAndPrepareBatch implements method in Validator interface
func (v *Validator) ValidateAndPrepareBatch(block *common.Block, doMVCCValidation bool) (*statedb.UpdateBatch, error) {
    logger.Debugf("New block arrived for validation:%#v, doMVCCValidation=%t", block, doMVCCValidation)
    updates := statedb.NewUpdateBatch()
    logger.Debugf("Validating a block with [%d] transactions", len(block.Data.Data))

    // Committer validator has already set validation flags based on well formed tran checks
    txsFilter := util.TxValidationFlags(block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER])

    // Precaution in case committer validator has not added validation flags yet
    if len(txsFilter) == 0 {
        txsFilter = util.NewTxValidationFlags(len(block.Data.Data))
        block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsFilter
    }

    for txIndex, envBytes := range block.Data.Data {
        if txsFilter.IsInvalid(txIndex) {
            // Skiping invalid transaction
            logger.Warningf("Block [%d] Transaction index [%d] marked as invalid by committer. Reason code [%d]",
                block.Header.Number, txIndex, txsFilter.Flag(txIndex))
            continue
        }

        env, err := putils.GetEnvelopeFromBlock(envBytes)
        if err != nil {
            return nil, err
        }

        payload, err := putils.GetPayload(env)
        if err != nil {
            return nil, err
        }

        chdr, err := putils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
        if err != nil {
            return nil, err
        }

        txType := common.HeaderType(chdr.Type)

        if txType != common.HeaderType_ENDORSER_TRANSACTION {
            logger.Debugf("Skipping mvcc validation for Block [%d] Transaction index [%d] because, the transaction type is [%s]",
                block.Header.Number, txIndex, txType)
            continue
        }
        //验证区块中的交易
        txRWSet, txResult, err := v.validateEndorserTX(envBytes, doMVCCValidation, updates)

        if err != nil {
            return nil, err
        }

        txsFilter.SetFlag(txIndex, txResult)

        //txRWSet != nil => t is valid
        if txRWSet != nil {
            //验证通过的交易，将需要更新的东西都加入到Batch中，这个之后会批量一起写入数据库。
            committingTxHeight := version.NewHeight(block.Header.Number, uint64(txIndex))
            addWriteSetToBatch(txRWSet, committingTxHeight, updates)
            txsFilter.SetFlag(txIndex, peer.TxValidationCode_VALID)
        }

        if txsFilter.IsValid(txIndex) {
            logger.Debugf("Block [%d] Transaction index [%d] TxId [%s] marked as valid by state validator",
                block.Header.Number, txIndex, chdr.TxId)
        } else {
            logger.Warningf("Block [%d] Transaction index [%d] TxId [%s] marked as invalid by state validator. Reason code [%d]",
                block.Header.Number, txIndex, chdr.TxId, txsFilter.Flag(txIndex))
        }
    }
    block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsFilter
    return updates, nil
}

```
读集验证的具体过程，大致就是去state db中查看读集中的字段，当前的Version（实际上就是上次写入时的blockNum和TranNum组合起来的），如果不一致则冲突，标记读写冲突的失败标记。

```go
//validate endorser transaction
func (v *Validator) validateEndorserTX(envBytes []byte, doMVCCValidation bool, updates *statedb.UpdateBatch) (*rwsetutil.TxRwSet, peer.TxValidationCode, error) {
    // extract actions from the envelope message
    //从envlope中获取payload
    respPayload, err := putils.GetActionFromEnvelope(envBytes)
    if err != nil {
        return nil, peer.TxValidationCode_NIL_TXACTION, nil
    }

    //preparation for extracting RWSet from transaction
    txRWSet := &rwsetutil.TxRwSet{}

    // Get the Result from the Action
    // and then Unmarshal it into a TxReadWriteSet using custom unmarshalling
    //拿到读写集
    if err = txRWSet.FromProtoBytes(respPayload.Results); err != nil {
        return nil, peer.TxValidationCode_INVALID_OTHER_REASON, nil
    }

    txResult := peer.TxValidationCode_VALID

    //mvccvalidation, may invalidate transaction
    if doMVCCValidation {
        //验证读集
        if txResult, err = v.validateTx(txRWSet, updates); err != nil {
            return nil, txResult, err
        } else if txResult != peer.TxValidationCode_VALID {
            txRWSet = nil
        }
    }

    return txRWSet, txResult, err
}
func (v *Validator) validateTx(txRWSet *rwsetutil.TxRwSet, updates *statedb.UpdateBatch) (peer.TxValidationCode, error) {
	for _, nsRWSet := range txRWSet.NsRwSets {
		ns := nsRWSet.NameSpace
    //验证读集，看读集是否冲突
		if valid, err := v.validateReadSet(ns, nsRWSet.KvRwSet.Reads, updates); !valid || err != nil {
			if err != nil {
				return peer.TxValidationCode(-1), err
			}
			return peer.TxValidationCode_MVCC_READ_CONFLICT, nil
		}
		if valid, err := v.validateRangeQueries(ns, nsRWSet.KvRwSet.RangeQueriesInfo, updates); !valid || err != nil {
			if err != nil {
				return peer.TxValidationCode(-1), err
			}
			return peer.TxValidationCode_PHANTOM_READ_CONFLICT, nil
		}
	}
	return peer.TxValidationCode_VALID, nil
}

func (v *Validator) validateReadSet(ns string, kvReads []*kvrwset.KVRead, updates *statedb.UpdateBatch) (bool, error) {
	for _, kvRead := range kvReads {
		if valid, err := v.validateKVRead(ns, kvRead, updates); !valid || err != nil {
			return valid, err
		}
	}
	return true, nil
}

// validateKVRead performs mvcc check for a key read during transaction simulation.
// i.e., it checks whether a key/version combination is already updated in the statedb (by an already committed block)
// or in the updates (by a preceding valid transaction in the current block)
func (v *Validator) validateKVRead(ns string, kvRead *kvrwset.KVRead, updates *statedb.UpdateBatch) (bool, error) {
	if updates.Exists(ns, kvRead.Key) {
		return false, nil
	}
	versionedValue, err := v.db.GetState(ns, kvRead.Key)
	if err != nil {
		return false, nil
	}
	var committedVersion *version.Height
	if versionedValue != nil {
		committedVersion = versionedValue.Version
	}

	if !version.AreSame(committedVersion, rwsetutil.NewVersion(kvRead.Version)) {
		logger.Debugf("Version mismatch for key [%s:%s]. Committed version = [%s], Version in readSet [%s]",
			ns, kvRead.Key, committedVersion, kvRead.Version)
		return false, nil
	}
	return true, nil
}

func (v *Validator) validateRangeQueries(ns string, rangeQueriesInfo []*kvrwset.RangeQueryInfo, updates *statedb.UpdateBatch) (bool, error) {
	for _, rqi := range rangeQueriesInfo {
		if valid, err := v.validateRangeQuery(ns, rqi, updates); !valid || err != nil {
			return valid, err
		}
	}
	return true, nil
}

// validateRangeQuery performs a phatom read check i.e., it
// checks whether the results of the range query are still the same when executed on the
// statedb (latest state as of last committed block) + updates (prepared by the writes of preceding valid transactions
// in the current block and yet to be committed as part of group commit at the end of the validation of the block)
func (v *Validator) validateRangeQuery(ns string, rangeQueryInfo *kvrwset.RangeQueryInfo, updates *statedb.UpdateBatch) (bool, error) {
	logger.Debugf("validateRangeQuery: ns=%s, rangeQueryInfo=%s", ns, rangeQueryInfo)

	// If during simulation, the caller had not exhausted the iterator so
	// rangeQueryInfo.EndKey is not actual endKey given by the caller in the range query
	// but rather it is the last key seen by the caller and hence the combinedItr should include the endKey in the results.
	includeEndKey := !rangeQueryInfo.ItrExhausted

	combinedItr, err := newCombinedIterator(v.db, updates,
		ns, rangeQueryInfo.StartKey, rangeQueryInfo.EndKey, includeEndKey)
	if err != nil {
		return false, err
	}
	defer combinedItr.Close()
	var validator rangeQueryValidator
	if rangeQueryInfo.GetReadsMerkleHashes() != nil {
		logger.Debug(`Hashing results are present in the range query info hence, initiating hashing based validation`)
		validator = &rangeQueryHashValidator{}
	} else {
		logger.Debug(`Hashing results are not present in the range query info hence, initiating raw KVReads based validation`)
		validator = &rangeQueryResultsValidator{}
	}
	validator.init(rangeQueryInfo, combinedItr)
	return validator.validate()
}

```
2.2.2 调用AddBlock将区块写入到区块文件

```go
// AddBlock adds a new block
func (store *fsBlockStore) AddBlock(block *common.Block) error {
    return store.fileMgr.addBlock(block)
}
func (mgr *blockfileMgr) addBlock(block *common.Block) error {
    if block.Header.Number != mgr.getBlockchainInfo().Height {
        return fmt.Errorf("Block number should have been %d but was %d", mgr.getBlockchainInfo().Height, block.Header.Number)
    }
    blockBytes, info, err := serializeBlock(block)
    if err != nil {
        return fmt.Errorf("Error while serializing block: %s", err)
    }
    blockHash := block.Header.Hash()
    //Get the location / offset where each transaction starts in the block and where the block ends
    txOffsets := info.txOffsets
    currentOffset := mgr.cpInfo.latestFileChunksize
    if err != nil {
        return fmt.Errorf("Error while serializing block: %s", err)
    }
    blockBytesLen := len(blockBytes)
    blockBytesEncodedLen := proto.EncodeVarint(uint64(blockBytesLen))
    totalBytesToAppend := blockBytesLen + len(blockBytesEncodedLen)

    //Determine if we need to start a new file since the size of this block
    //exceeds the amount of space left in the current file
    if currentOffset+totalBytesToAppend > mgr.conf.maxBlockfileSize {
        mgr.moveToNextFile()
        currentOffset = 0
    }
    //append blockBytesEncodedLen to the file
    err = mgr.currentFileWriter.append(blockBytesEncodedLen, false)
    if err == nil {
        //append the actual block bytes to the file
        err = mgr.currentFileWriter.append(blockBytes, true)
    }
    if err != nil {
        truncateErr := mgr.currentFileWriter.truncateFile(mgr.cpInfo.latestFileChunksize)
        if truncateErr != nil {
            panic(fmt.Sprintf("Could not truncate current file to known size after an error during block append: %s", err))
        }
        return fmt.Errorf("Error while appending block to file: %s", err)
    }

    //Update the checkpoint info with the results of adding the new block
    currentCPInfo := mgr.cpInfo
    newCPInfo := &checkpointInfo{
        latestFileChunkSuffixNum: currentCPInfo.latestFileChunkSuffixNum,
        latestFileChunksize:      currentCPInfo.latestFileChunksize + totalBytesToAppend,
        isChainEmpty:             false,
        lastBlockNumber:          block.Header.Number}
    //save the checkpoint information in the database
    if err = mgr.saveCurrentInfo(newCPInfo, false); err != nil {
        truncateErr := mgr.currentFileWriter.truncateFile(currentCPInfo.latestFileChunksize)
        if truncateErr != nil {
            panic(fmt.Sprintf("Error in truncating current file to known size after an error in saving checkpoint info: %s", err))
        }
        return fmt.Errorf("Error while saving current file info to db: %s", err)
    }

    //Index block file location pointer updated with file suffex and offset for the new block
    blockFLP := &fileLocPointer{fileSuffixNum: newCPInfo.latestFileChunkSuffixNum}
    blockFLP.offset = currentOffset
    // shift the txoffset because we prepend length of bytes before block bytes
    for _, txOffset := range txOffsets {
        txOffset.loc.offset += len(blockBytesEncodedLen)
    }
    //save the index in the database
    mgr.index.indexBlock(&blockIdxInfo{
        blockNum: block.Header.Number, blockHash: blockHash,
        flp: blockFLP, txOffsets: txOffsets, metadata: block.Metadata})

    //update the checkpoint info (for storage) and the blockchain info (for APIs) in the manager
    mgr.updateCheckpoint(newCPInfo)
    mgr.updateBlockchainInfo(blockHash, block)

    return nil
}
```

2.2.3 调用txtmgmt的Commit更新statedb

```go
// Commit implements method in interface `txmgmt.TxMgr`
func (txmgr *LockBasedTxMgr) Commit() error {
    logger.Debugf("Committing updates to state database")
    txmgr.commitRWLock.Lock()
    defer txmgr.commitRWLock.Unlock()
    logger.Debugf("Write lock acquired for committing updates to state database")
    if txmgr.batch == nil {
        panic("validateAndPrepare() method should have been called before calling commit()")
    }
    defer func() { txmgr.batch = nil }()
    if err := txmgr.db.ApplyUpdates(txmgr.batch,
        version.NewHeight(txmgr.currentBlock.Header.Number, uint64(len(txmgr.currentBlock.Data.Data)-1))); err != nil {
        return err
    }
    logger.Debugf("Updates committed to state database")
    return nil
}
```

2.2.4 更新historydb

```go
// Commit implements method in HistoryDB interface
func (historyDB *historyDB) Commit(block *common.Block) error {

    blockNo := block.Header.Number
    //Set the starting tranNo to 0
    var tranNo uint64

    dbBatch := leveldbhelper.NewUpdateBatch()

    logger.Debugf("Channel [%s]: Updating history database for blockNo [%v] with [%d] transactions",
        historyDB.dbName, blockNo, len(block.Data.Data))

    // Get the invalidation byte array for the block
    txsFilter := util.TxValidationFlags(block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER])
    // Initialize txsFilter if it does not yet exist (e.g. during testing, for genesis block, etc)
    if len(txsFilter) == 0 {
        txsFilter = util.NewTxValidationFlags(len(block.Data.Data))
        block.Metadata.Metadata[common.BlockMetadataIndex_TRANSACTIONS_FILTER] = txsFilter
    }

    // write each tran's write set to history db
    for _, envBytes := range block.Data.Data {

        // If the tran is marked as invalid, skip it
        if txsFilter.IsInvalid(int(tranNo)) {
            logger.Debugf("Channel [%s]: Skipping history write for invalid transaction number %d",
                historyDB.dbName, tranNo)
            tranNo++
            continue
        }

        env, err := putils.GetEnvelopeFromBlock(envBytes)
        if err != nil {
            return err
        }

        payload, err := putils.GetPayload(env)
        if err != nil {
            return err
        }

        chdr, err := putils.UnmarshalChannelHeader(payload.Header.ChannelHeader)
        if err != nil {
            return err
        }

        if common.HeaderType(chdr.Type) == common.HeaderType_ENDORSER_TRANSACTION {

            // extract actions from the envelope message
            respPayload, err := putils.GetActionFromEnvelope(envBytes)
            if err != nil {
                return err
            }

            //preparation for extracting RWSet from transaction
            txRWSet := &rwsetutil.TxRwSet{}

            // Get the Result from the Action and then Unmarshal
            // it into a TxReadWriteSet using custom unmarshalling
            if err = txRWSet.FromProtoBytes(respPayload.Results); err != nil {
                return err
            }
            // for each transaction, loop through the namespaces and writesets
            // and add a history record for each write
            for _, nsRWSet := range txRWSet.NsRwSets {
                ns := nsRWSet.NameSpace

                for _, kvWrite := range nsRWSet.KvRwSet.Writes {
                    writeKey := kvWrite.Key

                    //composite key for history records is in the form ns~key~blockNo~tranNo
                    compositeHistoryKey := historydb.ConstructCompositeHistoryKey(ns, writeKey, blockNo, tranNo)

                    // No value is required, write an empty byte array (emptyValue) since Put() of nil is not allowed
                    dbBatch.Put(compositeHistoryKey, emptyValue)
                }
            }

        } else {
            logger.Debugf("Skipping transaction [%d] since it is not an endorsement transaction\n", tranNo)
        }
        tranNo++
    }

    // add savepoint for recovery purpose
    height := version.NewHeight(blockNo, tranNo)
    dbBatch.Put(savePointKey, height.ToBytes())

    // write the block's history records and savepoint to LevelDB
    if err := historyDB.db.WriteBatch(dbBatch, false); err != nil {
        return err
    }

    logger.Debugf("Channel [%s]: Updates committed to history database for blockNo [%v]", historyDB.dbName, blockNo)
    return nil
}

```