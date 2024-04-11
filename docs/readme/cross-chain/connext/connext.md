---
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Connext



## connext概述

### 什么是connext？

![connext\_\_Logo+%2B+WhiteText+MultiColor.png](<connext 374baea3a2724d87b71c4831cd58ea49/connext\_\_Logo2BWhiteTextMultiColor.png>)

Connext 是一个网络，旨在在链与 L2 之间进行快速、无需信任的通信。它的主要功能是链之间的代币交换和数据传输，而与其他互操作性系统不同的是，Connext 在不引入新的信任假设或外部验证者的情况下实现了这一点。

Connext 的目标是实现低成本、无信任和普遍性。它通过高资本效率的代币转移，使其成为用户最便宜的桥接基础设施。通过底层链的保护，交易由底层链保护，无需信任第三方。同时，Connext 可以部署到任何类型的链或 L2 系统上，并以相同的方式工作。

该系统的概述如下：Connext 的网络迭代采用了 `nxtp`，这是一个轻量级的通用跨链传输协议。`Nxtp` 由一个简单的合约组成，它使用锁定模式来准备和履行交易。在这个系统中，有两个主要的参与角色：路由器和用户。路由器参与定价拍卖并提供交易流动性，而用户通过使用 `nxtp` 用户端 SDK，在链上寻找路由器并进行交易。

## 主要合约

![contract.jpg](<connext 374baea3a2724d87b71c4831cd58ea49/contract.jpg>)

## 交易过程

![tx.jpg](<connext 374baea3a2724d87b71c4831cd58ea49/tx.jpg>)

## 交易生命周期

![HighLevelFlow-8fa010ecd5303fc6b12c9ecb54e5a83b.png](<connext 374baea3a2724d87b71c4831cd58ea49/HighLevelFlow-8fa010ecd5303fc6b12c9ecb54e5a83b.png>)

### **交易的三个阶段**

1. `Router` 出价拍卖：用户向 Connext 网络广播他们所需的路由器。路由器则以密封的出价回应，承诺在一定的时间和价格范围内完成交易。
2. 准备阶段包括以下步骤：
   1. 拍卖完成后，准备交易。
   2. 用户提交一笔交易到发送方链上的 `TransactionManager` 合约，包含路由器的签名出价。这笔交易将用户在发送方链上的资金（发送金额）锁定。
   3.  当路由器检测到一个包含他们签名的链上准备事件时，它会向接收方链上的 `TransactionManager` 提交相同的交易，并锁定相应数量的流动资金。在接收方链上锁定的金额是发送金额减去拍卖费，这样就可以激励路由器完成交易。

       ```solidity
       function prepare(
           PrepareArgs calldata args
         ) external payable override nonReentrant returns (TransactionData memory) {
           // Sanity check: user is sensible
           require(args.invariantData.user != address(0), "#P:009");

           // Sanity check: router is sensible
           require(args.invariantData.router != address(0), "#P:001");

           // Router is approved *on both chains*
           require(isRouterOwnershipRenounced() || approvedRouters[args.invariantData.router], "#P:003");

           // Sanity check: sendingChainFallback is sensible
           require(args.invariantData.sendingChainFallback != address(0), "#P:010");

           // Sanity check: valid fallback
           require(args.invariantData.receivingAddress != address(0), "#P:026");

           // Make sure the chains are different
           require(args.invariantData.sendingChainId != args.invariantData.receivingChainId, "#P:011");

           // Make sure the chains are relevant
           uint256 _chainId = getChainId();
           require(args.invariantData.sendingChainId == _chainId || args.invariantData.receivingChainId == _chainId, "#P:012");

           { // Expiry scope
             // Make sure the expiry is greater than min
             uint256 buffer = args.expiry - block.timestamp;
             require(buffer >= MIN_TIMEOUT, "#P:013");

             // Make sure the expiry is lower than max
             require(buffer <= MAX_TIMEOUT, "#P:014");
           }

           // Make sure the hash is not a duplicate
           bytes32 digest = keccak256(abi.encode(args.invariantData));
           require(variantTransactionData[digest] == bytes32(0), "#P:015");

           // NOTE: the `encodedBid` and `bidSignature` are simply passed through
           //       to the contract emitted event to ensure the availability of
           //       this information. Their validity is asserted offchain, and
           //       is out of scope of this contract. They are used as inputs so
           //       in the event of a router or user crash, they may recover the
           //       correct bid information without requiring an offchain store.

           // Amount actually used (if fee-on-transfer will be different than
           // supplied)
           uint256 amount = args.amount;

           // First determine if this is sender side or receiver side
           if (args.invariantData.sendingChainId == _chainId) {
             // Check the sender is correct
             require(msg.sender == args.invariantData.initiator, "#P:039");

             // Sanity check: amount is sensible
             // Only check on sending chain to enforce router fees. Transactions could
             // be 0-valued on receiving chain if it is just a value-less call to some
             // `IFulfillHelper`
             require(args.amount > 0, "#P:002");

             // Assets are approved
             // NOTE: Cannot check this on receiving chain because of differing
             // chain contexts
             require(isAssetOwnershipRenounced() || approvedAssets[args.invariantData.sendingAssetId], "#P:004");

             // This is sender side prepare. The user is beginning the process of
             // submitting an onchain tx after accepting some bid. They should
             // lock their funds in the contract for the router to claim after
             // they have revealed their signature on the receiving chain via
             // submitting a corresponding `fulfill` tx

             // Validate correct amounts on msg and transfer from user to
             // contract
             amount = transferAssetToContract(
               args.invariantData.sendingAssetId,
               args.amount
             );

             // Store the transaction variants. This happens after transferring to
             // account for fee on transfer tokens
             variantTransactionData[digest] = hashVariantTransactionData(
               amount,
               args.expiry,
               block.number
             );
           } else {
             // This is receiver side prepare. The router has proposed a bid on the
             // transfer which the user has accepted. They can now lock up their
             // own liquidity on th receiving chain, which the user can unlock by
             // calling `fulfill`. When creating the `amount` and `expiry` on the
             // receiving chain, the router should have decremented both. The
             // expiry should be decremented to ensure the router has time to
             // complete the sender-side transaction after the user completes the
             // receiver-side transactoin. The amount should be decremented to act as
             // a fee to incentivize the router to complete the transaction properly.

             // Check that the callTo is a contract
             // NOTE: This cannot happen on the sending chain (different chain
             // contexts), so a user could mistakenly create a transfer that must be
             // cancelled if this is incorrect
             require(args.invariantData.callTo == address(0) || Address.isContract(args.invariantData.callTo), "#P:031");

             // Check that the asset is approved
             // NOTE: This cannot happen on both chains because of differing chain
             // contexts. May be possible for user to create transaction that is not
             // prepare-able on the receiver chain.
             require(isAssetOwnershipRenounced() || approvedAssets[args.invariantData.receivingAssetId], "#P:004");

             // Check that the caller is the router
             require(msg.sender == args.invariantData.router, "#P:016");

             // Check that the router isnt accidentally locking funds in the contract
             require(msg.value == 0, "#P:017");

             // Check that router has liquidity
             uint256 balance = routerBalances[args.invariantData.router][args.invariantData.receivingAssetId];
             require(balance >= amount, "#P:018");

             // Store the transaction variants
             variantTransactionData[digest] = hashVariantTransactionData(
               amount,
               args.expiry,
               block.number
             );

             // Decrement the router liquidity
             // using unchecked because underflow protected against with require
             unchecked {
               routerBalances[args.invariantData.router][args.invariantData.receivingAssetId] = balance - amount;
             }
           }

           // Emit event
           TransactionData memory txData = TransactionData({
             receivingChainTxManagerAddress: args.invariantData.receivingChainTxManagerAddress,
             user: args.invariantData.user,
             router: args.invariantData.router,
             initiator: args.invariantData.initiator,
             sendingAssetId: args.invariantData.sendingAssetId,
             receivingAssetId: args.invariantData.receivingAssetId,
             sendingChainFallback: args.invariantData.sendingChainFallback,
             callTo: args.invariantData.callTo,
             receivingAddress: args.invariantData.receivingAddress,
             callDataHash: args.invariantData.callDataHash,
             transactionId: args.invariantData.transactionId,
             sendingChainId: args.invariantData.sendingChainId,
             receivingChainId: args.invariantData.receivingChainId,
             amount: amount,
             expiry: args.expiry,
             preparedBlockNumber: block.number
           });

           emit TransactionPrepared(
             txData.user,
             txData.router,
             txData.transactionId,
             txData,
             msg.sender,
             args
           );

           return txData;
         }

       // 输入的参数
        struct PrepareArgs {
           InvariantTransactionData invariantData;
           uint256 amount;
           uint256 expiry;
           bytes encryptedCallData;
           bytes encodedBid;
           bytes bidSignature;
           bytes encodedMeta;
         }
           // Holds all data that is constant between sending and
         // receiving chains. The hash of this is what gets signed
         // to ensure the signature can be used on both chains.
         struct InvariantTransactionData {
           address receivingChainTxManagerAddress; //接收链的合约地址
           address user; //用户地址
           address router; //路由器地址
           address initiator; // msg.sender of sending side
           address sendingAssetId;
           address receivingAssetId;
           address sendingChainFallback; // funds sent here on cancel
           address receivingAddress;
           address callTo;
           uint256 sendingChainId;
           uint256 receivingChainId;
           bytes32 callDataHash; // hashed to prevent free option
           bytes32 transactionId;
         }
       ```
3. 在履行阶段，步骤如下：
   1. 用户在检测到接收方链上的 `TransactionPrepared` 事件后，签署一条消息并将其发送给中继者，中继者将获取提交费用。
   2. 中继者（通常是另一个路由器）随后将消息提交给 TransactionManager，以完成用户在接收方链上的交易，并索取由路由器锁定的资金。使用中继器是为了让用户在接收链上提交具有任意 `calldata` 的交易，而无需 gas 来完成。
   3. 路由器随后提交相同的签名信息，并在发送方完成交易，解锁金额。

```solidity
function fulfill(
    FulfillArgs calldata args
  ) external override nonReentrant returns (TransactionData memory) {
    // Get the hash of the invariant tx data. This hash is the same
    // between sending and receiving chains. The variant data is stored
    // in the contract when `prepare` is called within the mapping.

    { // scope: validation and effects
      bytes32 digest = hashInvariantTransactionData(args.txData);

      // Make sure that the variant data matches what was stored
      require(variantTransactionData[digest] == hashVariantTransactionData(
        args.txData.amount,
        args.txData.expiry,
        args.txData.preparedBlockNumber
      ), "#F:019");

      // Make sure the expiry has not elapsed
      require(args.txData.expiry >= block.timestamp, "#F:020");

      // Make sure the transaction wasn't already completed
      require(args.txData.preparedBlockNumber > 0, "#F:021");

      // Check provided callData matches stored hash
      require(keccak256(args.callData) == args.txData.callDataHash, "#F:024");

      // To prevent `fulfill` / `cancel` from being called multiple times, the
      // preparedBlockNumber is set to 0 before being hashed. The value of the
      // mapping is explicitly *not* zeroed out so users who come online without
      // a store can tell the difference between a transaction that has not been
      // prepared, and a transaction that was already completed on the receiver
      // chain.
      variantTransactionData[digest] = hashVariantTransactionData(
        args.txData.amount,
        args.txData.expiry,
        0
      );
    }

    // Declare these variables for the event emission. Are only assigned
    // IFF there is an external call on the receiving chain
    bool success;
    bool isContract;
    bytes memory returnData;

    uint256 _chainId = getChainId();

    if (args.txData.sendingChainId == _chainId) {
      // The router is completing the transaction, they should get the
      // amount that the user deposited credited to their liquidity
      // reserves.

      // Make sure that the user is not accidentally fulfilling the transaction
      // on the sending chain
      require(msg.sender == args.txData.router, "#F:016");

      // Validate the user has signed
      require(
        recoverFulfillSignature(
          args.txData.transactionId,
          args.relayerFee,
          args.txData.receivingChainId,
          args.txData.receivingChainTxManagerAddress,
          args.signature
        ) == args.txData.user, "#F:022"
      );

      // Complete tx to router for original sending amount
      routerBalances[args.txData.router][args.txData.sendingAssetId] += args.txData.amount;

    } else {
      // Validate the user has signed, using domain of contract
      require(
        recoverFulfillSignature(
          args.txData.transactionId,
          args.relayerFee,
          _chainId,
          address(this),
          args.signature
        ) == args.txData.user, "#F:022"
      );

      // Sanity check: fee <= amount. Allow `=` in case of only
      // wanting to execute 0-value crosschain tx, so only providing
      // the fee amount
      require(args.relayerFee <= args.txData.amount, "#F:023");

      (success, isContract, returnData) = _receivingChainFulfill(
        args.txData,
        args.relayerFee,
        args.callData
      );
    }

    // Emit event
    emit TransactionFulfilled(
      args.txData.user,
      args.txData.router,
      args.txData.transactionId,
      args,
      success,
      isContract,
      returnData,
      msg.sender
    );

    return args.txData;
  }

```

如果交易在一个固定的到期时间内没有完成，它就可以被取消。 如果交易还未到期，用户只能取消接收方的`prepare`，而`router`只能取消发送方的`prepare`。

## connext的架构

![Architecture-6db700297f1357f8b5459ce1df5d2ef7.png](<connext 374baea3a2724d87b71c4831cd58ea49/Architecture-6db700297f1357f8b5459ce1df5d2ef7.png>)

* **Contracts**：这个部分负责管理网络中的资金，根据用户和路由器提交的数据来控制资金的锁定和解锁。
* **Subgraph**：这个功能通过缓存链上的数据和事件，让查询和响应更加高效。
* **TxService**：负责向区块链发送交易，而且可以处理重试等问题。
* **Messaging**：通过Nats系统来准备、发送和监听消息数据。
* **Router**：监听来自消息服务和子图的事件，然后将交易发送给TxService来处理。
* **SDK**：提供创建拍卖、监听事件以及在用户端创建交易等功能。

## connext的安全性

通常情况下，Connext采用与其他锁定系统如HTLCs相同的核心安全模型。不同于其他系统的是，Connext受到自由选择的影响较小，因为所有跨链的互动都是用1:1的资产（例如，从Optimism的ETH到Arbitrum的ETH）。

HTLCs（Hash Time Locked Contracts）：HTLC是一种付款方式，它使用哈希锁和时间锁。接收方需要在截止日期前通过生成加密的付款证明来确认收到付款，或者放弃索取付款的能力，将其返还给付款人。

### 风险

#### 资金损失

除去黑客或用户错误的可能性，用户在这个系统中没有办法损失资金。为了确保他们的安全，用户只需要确保目的地链上的相应准备的交易与他们在原始链上发送的交易数据相同。

如果链受到攻击，或者在转移过程中，`router`长时间完全离线，就有可能导致资金损失。当用户在目的链上完成他们的交易后，`router`在交易到期前要在原点链上提交相应的`fulfill`交易，以索取他们的资金。如果`router`未能做到这一点，原生链上的交易将自动退回给用户。

#### `router`拒绝服务

`router`可能通过锁定用户资金，导致用户在过期前无法使用该资金。

`connext`正在实施一个惩罚机制，惩罚没有完成所承诺的交易的`router`

`nxtp`由一个简单的合约组成，它使用锁定模式来准备和履行交易。



## 总结

Connext 是一个跨链互操作性协议，它允许在兼容EVM的区块链之间进行快速、非托管的资金转移和合约调用。这里是关于 Connext 合约的一些要点总结：

* **互操作性协议**：Connext 旨在统一跨链流动性，使得资金可以在不同的区块链之间自由流动。
* **流动性网络**：它由节点（路由器）组成的链下点对点网络，这些节点持有发送和接收链上的资产“库存”。
* **资金转移机制**：用户在发送链上“锁定”资金，并在接收链上“解锁”相应金额（减去费用）。路由器在交易开始时在接收链上为用户锁定资金，在发送链上索取锁定的资金。
* **非托管跨链传输协议 (NXTP)**：Connext 的 NXTP 是一种不添加新信任假设的通用互操作性解决方案，它最小化了用户对路由器的信任需求。
* **可扩展性**：Connext 可以轻松地将新链加入网络，从而快速扩展到其他生态系统。
* **资金效率**：与安全模型脱钩的资本锁定，允许网络安全提高经济吞吐量而没有博弈论上限。
* **用户体验**：由于验证发生在流动性网络本地，跨链传输速度很快，且与L1和L2无关。
* **流动性再平衡**：Connext 在 AMM 曲线上动态定价流动性，以套利激励来重新平衡网络流动性。

Connext 的这些特点使其在跨链转移和合约调用方面处于有利位置，为用户提供了一个安全、快速且资金效率高的跨链解决方案。

> 参考链接：[https://zhuanlan.zhihu.com/p/420858923](https://zhuanlan.zhihu.com/p/420858923)
