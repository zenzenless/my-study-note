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

![connext\_\_Logo+%2B+WhiteText+MultiColor.png](<../../readme/cross-chain/connext/connext 374baea3a2724d87b71c4831cd58ea49/connext\_\_Logo2BWhiteTextMultiColor.png>)

Connext 是一个用于在链和 L2 之间进行快速、无需信任的通信的网络

它主要实现了链之间的代币交换及数据传输，与大多数其他互操作性系统不同，Connext是在没有引入任何新的`信任假设`或`外部验证器`的情况下实现了这一点。

### connext的目标

* 低成本

​ 转移具有很高的资本效率，使 Connext 成为用户最便宜的桥接基础设施

* 无信任

​ 通过 Connext 进行的交易由底层链保护，无需信任第三方

* 普遍性

​ Connext 可以部署到任何类型的链或 L2 系统上并以相同的方式工作

### connext系统概述

#### 它是如何工作的？

Connext的这个网络迭代采用了`nxtp`，这是一个轻量级的通用跨链传输协议。

`nxtp`由一个简单的合约组成，它使用锁定模式来准备和履行交易。

参与的角色：

* `router`：参与定价拍卖和提供交易流动性。
* 用户：通过使用`nxtp`用户端`sdk` ，寻找`router`并进行链上交易。

主要合约：

![contract.jpg](<../../readme/cross-chain/connext/connext 374baea3a2724d87b71c4831cd58ea49/contract.jpg>)

交易过程：

![tx.jpg](<../../readme/cross-chain/connext/connext 374baea3a2724d87b71c4831cd58ea49/tx.jpg>)

#### 交易生命周期

![HighLevelFlow-8fa010ecd5303fc6b12c9ecb54e5a83b.png](<../../readme/cross-chain/connext/connext 374baea3a2724d87b71c4831cd58ea49/HighLevelFlow-8fa010ecd5303fc6b12c9ecb54e5a83b.png>)

**交易分为三个阶段**

1. Router 出价拍卖： 用户向connext的网络广播，表示他们想要的`router`。`router`以密封的出价回应，其中包含在一定时间和价格范围内完成交易的承诺。
2.  准备阶段 一旦拍卖完成，就可以准备交易。用户向`发送方`链上的`TransactionManager`合约提交一笔交易，其中包含`router`的签名出价。这个交易锁定了用户在`发送方`链上的资金(发送金额)。

    `router`在检测到一个包含他们签名的链上`prepare`事件后，`router`向`接收方`链上的 `TransactionManager` 提交相同的交易，并锁定`router`相应数量的流动资金。在接收方链上锁定的金额是`发送金额-拍卖费`，以此激励`router`完成交易。

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
3. 履行阶段 在检测到`接收方`链上的`TransactionPrepared`事件后，用户签名一个消息并将其发送给中继者，中继者将获得提交费用。中继者（通常是另一个`router`）然后将消息提交给`TransactionManager`，以完成用户在接收方链上的交易，并索取由路由器锁定的资金。这里使用中继器是为了让用户在接收链上提交具有任意calldata的交易，而不需要`gas`来完成。`router`然后提交相同的签名信息，并在发送方完成交易，解锁金额。

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

#### connext的架构

![Architecture-6db700297f1357f8b5459ce1df5d2ef7.png](<../../readme/cross-chain/connext/connext 374baea3a2724d87b71c4831cd58ea49/Architecture-6db700297f1357f8b5459ce1df5d2ef7.png>)

* **Contracts** ：为所有网络参与者持有资金，并根据用户和路由器提交的数据进行锁定/解锁
* **Subgraph**：通过缓存链上数据和事件实现可扩展的查询/响应。
* **TxService** ：向链上发送交易（有重试等）
* **Messaging** ：通过Nats准备、发送和监听消息数据。
* **Router**：监听来自消息服务和子图的事件，然后将交易分派给txService
* **SDK** : 创建拍卖，监听事件并在用户端创建交易。

#### connext的安全

一般来说，Connext采用与其他锁定系统如HTLCs相同的核心安全模型。与其他系统不同的是，Connext不太容易受到自由选择的影响，因为所有跨链的互动都是用1:1的资产（比如`Optimism` 的ETH到`Arbitrum`的ETH）。

`HTLCs (Hash Time Locked Contract)`：HTLC是一类使用哈希锁和时间锁的付款方式，要求付款的接收方通过生成加密的付款证明来确认在截止日期前收到付款，或者放弃索取付款的能力，将其返还给付款人。

#### 风险

#### 资金损失

除去黑客或用户错误的可能性，用户在这个系统中没有办法损失资金。为了确保他们的安全，用户只需要确保目的地链上的相应准备的交易与他们在原始链上发送的交易数据相同。

如果链受到攻击，或者在转移过程中，`router`长时间完全离线，就有可能导致资金损失。当用户在目的链上完成他们的交易后，`router`在交易到期前要在原点链上提交相应的`fulfill`交易，以索取他们的资金。如果`router`未能做到这一点，原生链上的交易将自动退回给用户。

#### `router`拒绝服务

`router`可能通过锁定用户资金，导致用户在过期前无法使用该资金。

`connext`正在实施一个惩罚机制，惩罚没有完成所承诺的交易的`router`
