# Cosmos

## 什么是cosmos网络？

* Cosmos并不是一条独立的区块链，而是一个网络，区块链的网络。
* Cosmos的目标不是建立自己的区块链，而是建立一个可互通的网络生态系统。为了实现互通的网络
* Cosmos提供开发者工具降低开发门槛，包括Tendermint共识引擎，Cosmos SDK 的模块化开发框架，加上IBC通信协议，实现区块链之间的信息和资产转移，打通不同区块链的孤岛，形成互联网。

### 回顾一下比特币、以太坊

**比特币(区块链1.0--去中心化的价值)：**

* 区块链上的第一个去中心化应用
* 采用POW机制
* 拥有51%算力才能改写历史
* 缺点：
  * 代码库非常单一，所有三层：网络、共识和应用程序都混合在一起。
  * 比特币脚本语言受到限制且不易于使用。
  * TPS低
  * 能源消耗巨大

**以太坊(区块链2.0--应用的爆发)：**

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

* 可在链上构建去中心化应用程序
* POW机制
* 无审查：任何人都能在以太坊上部署智能合约
*   缺点：

    * 可扩展性差，TPS低，约为每秒15笔交易，使用POW，能源消耗巨大，多个dApp竞争单个资源有限的区块链。
    * 智能合约局限于几种语言编写，开发灵活性较低。
    * 每个应用程序都局限在以太坊的底层环境上，如果应用程序出现错误，未经以太坊治理批准，将无能为力。





<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption><p>单一区块链架构</p></figcaption></figure>

这些限制并非特定于以太坊，而是所有试图创建适合所有用例的单一平台的区块链。这就是 Cosmos 发挥作用的地方。

**cosmos的愿景(区块链3.0--可扩展的模块化区块链网络)**

Cosmos 的愿景是让开发人员能够轻松地构建区块链并通过允许它们相互交易来打破区块链之间的障碍。最终目标是创建一个区块链互联网，一个能够以分散方式相互通信的区块链网络。借助 Cosmos，区块链可以维护主权，快速处理交易并与生态系统中的其他区块链进行通信，使其成为各种用例的最佳选择。

这些愿景是通过一组开源工具实现的，如：

* Tendermint
* Cosmos SDK
* IBC(跨链通讯协议)

旨在让人们快速构建自定义、安全、可扩展和可互操作的区块链应用程序。

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

## 共识

### Tendermint BFT

​ 一般构建区块链需要从头开始构建所有三层（网络、共识和应用程序）。

​ 以太坊通过提供虚拟机区块链简化了去中心化应用程序的开发，任何人都可以在其上以智能合约的形式部署自定义逻辑。但是，它并没有简化区块链本身的开发。就像比特币一样，`Go-Ethereum` 仍然是一个难以分叉和定制的单一技术堆栈。这就是 Jae Kwon 在 2014 年创建的 `Tendermint` 的用武之地。

`Tendermint BFT` 引擎通过称为应用程序区块链接口的套接字协议连接到应用程序（ABCI)，该协议可以包装在任何编程语言中，使开发人员可以选择适合他们需求的语言。



`Tendermint BFT` 是一种将区块链的网络和共识层打包到通用引擎中的解决方案，允许开发人员专注于应用程序开发，而不是复杂的底层协议。

#### 参与角色

* **验证者**：按照质押数量大小顺序，选出125位验证者(当前数量，未来可增长到150)。验证者通过广播加密签名、投票或者对下一个区块表决同意来参与共识协议。当网络上有占权重2/3的投票同意时，区块就能正确产生。（验证者的投票权重并不相同）
* **提议者**： 每一轮都由一个提议者负责出块。提议者是从验证者中选出的，每一轮选哪个验证者作为提议者，是按照验证者的质押权重比例确定性的选出，形成一个列表。

#### 共识算法

按照规则，验证者要按轮次（round）对每一个区块达成共识。每一轮都包含三个基本步骤：提议阶段（Propose）、预投票阶段（Prevote）、预提交阶段（Precommit），以及两个后续步骤：提交阶段（Commit）、新高度阶段（NewHeight）。

过程如下：

1. **提议阶段**：验证者中选一个作为提议者出块（已经预先安排好提议者顺序了）
2. **预投票阶段**： 每一位验证者对提议者提出的区块进行投票，当区块超过2/3的投票时，就进入提交阶段。
3. **提交阶段**： 投票通过的区块将被提交到链，使链高度增加。如果提交失败，要么返回预投票阶段，要么返回提议阶段。
4. **进入新高度**

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

#### 特点

* **Tendermint BFT 仅处理区块链的网络和共识**，这意味着它帮助节点传播交易，并且验证者同意一组交易以附加到区块链。应用层由开发者自行构建。
* **高性能**：Tendermint BFT 的出块时间约为 1 秒（目前是7秒），每秒处理多达数千笔交易。
* **即时确定性**：Tendermint 共识算法的一个属性是即时确定性。这意味着只要超过三分之二权重的投票确认，区块就确定了，并且永远不会发生分叉。用户可以确保他们的交易在创建区块后立即完成。
* **安全性**：Tendermint 共识不仅是容错的，而且是问责制的。如果区块链分叉，将对分叉的验证者进行惩罚。

### 区块链互联

TODO

#### IBC的工作原理

TODO

#### 连接异构链

TODO

### 模块化构建区块链

#### Cosmos SDK

Cosmos SDK 提供一套构建应用层的框架，就像是区块链界的 Ruby-on-Rails （Ruby-on-Rails 是一种让开发者轻松通过默认设置构建网页端应用的框架），Cosmos SDK 也为开发者提供了一种基于 Tendermint 内核构建安全的区块链应用的框架。

要记住，区块链就是一个在所有节点将状态做相同备份的状态机，而 Cosmos SDK 让你可以构建能在多个节点间进行复制的实际状态机。SDK 让你自定义应用的状态、事务类型，及状态-转变函数所需的功能和工具。

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

**Cosmos 应用如何运行（抽象视角)**

Cosmos SDK 提供一种「multistore」机制来定义及维护应用层状态机的状态。Multistore 将应用层的状态划分到不同组件，通过各自的「模块」进行管理。

Cosmos SDK 的强大之处就是它独特的模块化设计，每个模块定义及维护一个状态子集，这些状态子集构成完整的区块链。举例来说：

* **银行模块**：允许你在应用中持有代币，及进行代币转账
* **权限模块**：允许你创建及管理账户和签名
* **质押与罚没模块**：允许你通过编码构建 PoS 共识机制

每个模块都是个**小状态机**，可以相互聚合生成总体状态机。

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

应用开发者按照每个模块和修改状态的惯用逻辑来定义子集，除了 Cosmos SDK 提供的模块，开发者还能调用第三方模块。

这种用于构建区块链应用的插件模块非常强大，因为它赋予开发者使用 SDK 或外部模块的灵活性。

**应用层如何与共识层交互？**

发生在应用层的交易通过区块链应用交互界面（ABCI, Application Blockchain Interface）与 Tendermint 共识及网络层通信。

<figure><img src="../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

ABCI 是 Socket 通信协议，连接 Tendermint 核心（共识 + 网络）与应用。它可以兼容任何编程语言，也就是说使用 Cosmos SDK 构建的区块链应用理论上能以任何语言编写，而不仅仅是 Tendermint 底层共识和网路层所用的编程语言。

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

总而言之，Cosmos SDK 允许开发者基于 Tendermint 内核构建去中心化应用，这个应用理论上能用任何语言开发，并通过 ABCI 连接 Tendermint 共识引擎。

将应用层（Cosnos SDK、ABCI）与网路层、共识层（Tendermint 内核）剥离，能让开发者在开发各种类型应用的时候有更大的灵活性，再加上 Cosmos SDK 允许这些应用以任何语言开发（e.g. Golang），让区块链 App 开发过程更像平常的应用开发的样子。

***

#### Modules

**每个Cosmos链都是一个特制的区块链**。Cosmos SDK模块定义了每个链的独特属性。模块可以被认为是更大的状态机中的状态机。它们包含存储布局或状态和状态转换函数，也就是消息方法。

`Modules`定义了Cosmos SDK应用程序的大部分逻辑。

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

当一个事务从底层的`Tendermint`共识引擎中转时，`BaseApp`会分解事务中包含的消息。`BaseApp`将消息传送到适当的`Modules`进行处理。当适当的`Modules`消息处理程序收到消息时，就会进行解释和执行。

**Module 作用范围**

核心功能：

* 应用区块链接口（ABCI）的模板实现，与底层的`Tendermint`共识引擎进行通信。
* 一个通用的数据存储，用于持久保存模块状态，称为multistore。
* 一个服务器和接口，促进与节点的互动。

**module 的组件**

在`x/moduleName`文件夹中定义一个模块是一个最佳做法。例如，名为`Checkers`的模块将放在`x/checkers`中。如果你去看`Cosmos SDK`的基础代码，你可以看到它也定义了它的模块在一个`x/文件夹中`。

模块实现了几个元素:

* `Interfaces` (接口): 促进了模块之间的通信，并将多个模块组成连贯的应用程序。
* `Protobuf`是一个处理消息的`Msg`服务和一个处理查询的`gRPC`查询服务。
* `Keeper`是一个控制器，它定义了状态并提出了更新和检查状态的方法。

> #### Interfaces# 接口
>
> A module must implement **three application module interfaces** to be integrated with the rest of the application:
>
> * **`AppModuleBasic`:** implements non-dependent elements of the module.
> * **`AppModule`:** interdependent, specialized elements of the module that are unique to the application.
> * **`AppModuleGenesis`:** interdependent, genesis/initialization elements of the module that establish the initial state of the blockchain at inception.
>
> You define `AppModule` and `AppModuleBasic`, and their functions in your module's `x/moduleName/module.go` file.

**Keeper**:

其他`modules`可能需要访问`store`，但其他模块也有可能是恶意的。出于这个原因，开发人员需要考虑谁可以访问他们的`module store`。为了防止一个`module`在运行时随机访问另一个`module`，一个需要访问另一个`module`的`module`需要在构建时声明其使用另一个`module`的意图。在这一点上，这样的模块被授予一个运行时密钥，让它访问另一个`module`。只有持有这个密钥的`module`才能访问存储空间。这就是所谓的对象能力`module`的一部分。

#### Protobuf

Protobuf是一种数据序列化的方法。开发人员用它来描述消息格式。在Cosmos应用程序中存在大量的内部通信，而Protobuf是如何进行通信的核心。

gRPC

#### Multistore and Keepers

​ `Keepers` 负责管理对`module`定义的状态的访问。访问状态是通过 `keeper` 完成的。因此， `keeper` 是确保不变量应用始终安全的理想场所。

在一个特定应用的区块链上的Cosmos SDK应用通常由几个模块组成，分别处理不同的问题。每个模块都有一个状态，是整个应用状态的一个子集。

Cosmos SDK采用了一种基于对象能力的方法，以帮助开发者更好地**保护他们的应用程序**免受不必要的`module`间互动。`keeper`是这种方法的核心。

`Keeper`是Cosmos SDK的一个抽象，它管理对一个`module`所定义的状态子集的访问。所有对`module`数据的访问必须通过`module`的`keeper`。

(个人感觉有点类似`solidty`中的`modifier`)

#### BaseApp

BaseApp 是 Cosmos SDK 应用程序的样板实现。这种抽象实现了每个 Cosmos 应用程序所需的功能，从 Tendermint 应用程序区块链接口 (ABCI) 的实现开始。

**实现**：

* BaseApp 包括 ABCI 的实现，ABCI 本身包括交付交易的 DeliverTx 等方法。 BaseApp 包括一个服务路由器实现，将不同类型的交易发送给不同的处理器。
* BaseApp 还提供状态机实现，状态机的实现是应用程序级别的问题，因为 `Tendermint` 共识与应用程序无关。
* BaseApp 在应用程序、区块链和状态机之间提供安全接口，同时尽可能少地定义状态机。

```go
// BaseApp reflects the ABCI application implementation.
type BaseApp struct { // nolint: maligned
	// initialized on creation
	logger            log.Logger
	name              string               // application name from abci.Info
	db                dbm.DB               // common DB backend
	cms               sdk.CommitMultiStore // Main (uncached) state
	storeLoader       StoreLoader          // function to handle store loading, may be overridden with SetStoreLoader()
	router            sdk.Router           // handle any kind of message
	queryRouter       sdk.QueryRouter      // router for redirecting query calls
	grpcQueryRouter   *GRPCQueryRouter     // router for redirecting gRPC query calls
	msgServiceRouter  *MsgServiceRouter    // router for redirecting Msg service messages
	interfaceRegistry types.InterfaceRegistry
	txDecoder         sdk.TxDecoder // unmarshal []byte into sdk.Tx

	anteHandler    sdk.AnteHandler  // ante handler for fee and auth
	initChainer    sdk.InitChainer  // initialize state with validators and state blob
	beginBlocker   sdk.BeginBlocker // logic to run before any txs
	endBlocker     sdk.EndBlocker   // logic to run after all txs, and to determine valset changes
	addrPeerFilter sdk.PeerFilter   // filter peers by address and port
	idPeerFilter   sdk.PeerFilter   // filter peers by node ID
	fauxMerkleMode bool             // if true, IAVL MountStores uses MountStoresDB for simulation speed.

	// manages snapshots, i.e. dumps of app state at certain intervals
	snapshotManager    *snapshots.Manager
	snapshotInterval   uint64 // block interval between state sync snapshots
	snapshotKeepRecent uint32 // recent state sync snapshots to keep

	// volatile states:
	//
	// checkState is set on InitChain and reset on Commit
	// deliverState is set on InitChain and BeginBlock and set to nil on Commit
	checkState   *state // for CheckTx
	deliverState *state // for DeliverTx

	// an inter-block write-through cache provided to the context during deliverState
	interBlockCache sdk.MultiStorePersistentCache

	// absent validators from begin block
	voteInfos []abci.VoteInfo

	// paramStore is used to query for ABCI consensus parameters from an
	// application parameter store.
	paramStore ParamStore

	// The minimum gas prices a validator is willing to accept for processing a
	// transaction. This is mainly used for DoS and spam prevention.
	minGasPrices sdk.DecCoins

	// initialHeight is the initial height at which we start the baseapp
	initialHeight int64

	// flag for sealing options and parameters to a BaseApp
	sealed bool

	// block height at which to halt the chain and gracefully shutdown
	haltHeight uint64

	// minimum block time (in Unix seconds) at which to halt the chain and gracefully shutdown
	haltTime uint64

	// minRetainBlocks defines the minimum block height offset from the current
	// block being committed, such that all blocks past this offset are pruned
	// from Tendermint. It is used as part of the process of determining the
	// ResponseCommit.RetainHeight value during ABCI Commit. A value of 0 indicates
	// that no blocks should be pruned.
	//
	// Note: Tendermint block pruning is dependant on this parameter in conunction
	// with the unbonding (safety threshold) period, state pruning and state sync
	// snapshot parameters to determine the correct minimum value of
	// ResponseCommit.RetainHeight.
	minRetainBlocks uint64

	// application's version string
	appVersion string

	// recovery handler for app.runTx method
	runTxRecoveryMiddleware recoveryMiddleware

	// trace set will return full stack traces for errors in ABCI Log field
	trace bool

	// indexEvents defines the set of events in the form {eventType}.{attributeKey},
	// which informs Tendermint what to index. If empty, all events will be indexed.
	indexEvents map[string]struct{}
}
```

#### Queries

查询是应用程序的最终用户通过接口发出并由全节点处理的对信息的请求。可用信息包括：

* 有关网络的信息
* 有关应用程序本身的信息
* 有关应用程序状态的信息

#### Events

事件是包含有关应用程序执行信息的对象。区块浏览器和钱包等服务提供商使用事件来跟踪各种消息和索引交易的执行情况。

#### Context

事务在上下文中执行。上下文包括有关应用程序、块和事务的当前状态的信息。

上下文表示为携带有关应用程序当前状态的信息的数据结构，旨在从一个函数传递到另一个函数。上下文提供对作为整个状态的安全分支的分支存储的访问，以及有用的对象，以及 gasMeter、块高度和共识参数等信息。

#### Migrations

**提议**

治理使用经过投票、通过或拒绝的提案。升级提案采用接受或拒绝通过治理准备和提交的计划的形式。提案可以在执行前与取消提案一起撤回。

区块链在已采用计划的区块高度处暂停。这将启动升级过程。升级过程本身可能包括切换到下载和安装相对较小的新二进制文件，或者它可能包括广泛的数据重组过程。在任何一种情况下，验证器都会停止处理块，直到它完成该过程。当处理程序对升级的完整性程度感到满意时，验证器会恢复处理块。从用户的角度来看，这显示为暂停，并在新版本中恢复。

#### Inter-Blockchain Communication (IBC)

> Cosmos 中的跨链通信可实现并行性和可扩展性以及交易最终性。这种交易终结性解决了困扰其他平台的众所周知的问题：交易成本、网络容量和交易确认终结性。

区块链间通信协议 (IBC) 允许将数据从一个区块链发送到另一个区块链。大多数 Cosmos 应用程序在他们自己的专用区块链上执行，运行他们自己的验证器集。这些是使用 Cosmos SDK 构建的特定于应用程序的区块链。

一条链上的应用程序可能需要与另一条区块链上的应用程序通信。例如，应用程序可以接受来自另一个区块链的代币作为一种支付方式。此级别的互操作性需要一种交换有关状态或另一个区块链上的交易的数据的方法。

IBC 为所有 Cosmos SDK 应用程序提供了一个通用的协议和框架，用于实现标准化的跨链通信。

模块可以选择他们希望通过哪些通道进行通信。 IBC 期望模块实现在通道握手期间调用的回调。

四步握手是什么样子的？以下列方式进行四步握手：

1. 链 A 发送一个 ChanOpenInit 消息来表示与链 B 的通道初始化尝试。
2. 链 B 发送 ChanOpenTry 消息尝试打开链 A 上的通道。
3. 链 A 发送 ChanOpenAck 消息，将链 A 的通道结束状态标记为打开。
4. 链 B 发送 ChanOpenConfirm 消息，将链 B 的通道结束状态标记为打开。
5. 通道两边都是开放的。

正如端口带有动态功能一样，通道初始化返回模块必须声明的动态功能，以便模块可以传入验证通道操作的功能，例如发送数据包。通道功能在握手的第一部分被传递到回调中。

**Receipts and timeouts**

IBC 在分布式网络上工作。 IBC 可能需要依靠可能有故障的中继器在分类帐之间中继消息。跨链通信还需要处理数据包没有按时或根本没有发送到目的地的情况。

当达到超时时，可以将数据包证明超时提交给原始链，然后原始链可以执行特定于应用程序的逻辑，通过回滚数据包发送更改来使数据包超时，将任何锁定的资金退还给发送者。

通道中单个数据包的超时将关闭 ORDERED 通道中的通道。可以在 UNORDERED 通道中以任何顺序接收数据包。 IBC 为它在 UNORDERED 通道中接收到的每个序列写入一个数据包收据。

#### Bridges 桥

**The Gravity Bridge 重力桥**

重力桥 是 `Althea` 目前正在构建的一个正在进行的项目，其目标是促进 `ERC-20` 代币与基于 `Cosmos` 的区块链之间的转移。 `Gravity Bridge` 允许将 `Cosmos` 的力量与来自以太坊的资产价值相结合的新颖应用程序。开发人员可以使用 `Cosmos` 链计算在以太坊上太贵的智能合约的计算。开发人员可以接受以太坊 `ERC-20` 代币作为支付方式，或者围绕以太坊代币构建整个 Cosmos 应用程序。

**工作原理**

重力桥由几个部分组成：

* **Gravity.sol** 以太坊区块链上的以太坊智能合约
* **Cosmos Gravity module** 设计用于在 Cosmos Hub 上运行的 Cosmos 模块。
* **Orchestrator** 在 Cosmos 验证器上运行的程序，监控以太坊链并将以太坊上发生的事件作为消息提交给 Cosmos。
* **Relayers** 中继器。一个节点网络，它们争夺代表 Cosmos 验证者发送交易赚取费用的机会。

通过将代币发送到 Gravity.sol 智能合约，代币被锁定在以太坊一侧。这会发出一个事件，运行协调器的验证器可以观察到该事件。当一定数量的验证者同意代币已被锁定在以太坊上，包括必要的确认块时，就会选择一个中继器向 Cosmos Gravity 模块发送指令，该模块会发布新的代币。这是非稀释性的——它不会增加流通量，因为相同数量的代币被锁定在以太坊一侧。

为了将代币从 Cosmos Hub 转移到以太坊区块链，Cosmos 网络上的代币将被销毁，并从 Gravity.sol 智能合约中释放相同数量的代币（它们之前被存入）。

Cosmos Gravity Bridge 旨在在 Cosmos Hub 上运行。它专注于最大限度的设计简单性和效率。该桥可以将源自以太坊的 ERC-20 资产转移到基于 Cosmos 的链并返回到以太坊。交易是分批的，转账已扣除。这可以大规模提高效率并降低每次转移的交易成本。

***

### cosmos到底是什么？

1. Cosmos 通过 Tendermint BFT 和 Cosmos SDK 的模块化使区块链功能强大且易于开发。
2. Cosmos 使区块链能够通过 IBC 和 Peg-Zones 相互转移价值，同时让它们保留自己的主权。
3. Cosmos 允许区块链应用程序通过水平和垂直可扩展性解决方案扩展到数百万用户。

最重要的是，**Cosmos 不是一个产品，而是一个建立在一组模块化、适应性强和可互换工具之上的生态系统**。

### 愿景

### 主要流程

### 项目组成

### 综合评述
