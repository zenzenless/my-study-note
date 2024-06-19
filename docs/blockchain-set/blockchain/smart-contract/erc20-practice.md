# ERC20合约实战



### 什么是ERC20?

ERC20是以太坊区块链上的一个技术标准[EIP20](https://eips.ethereum.org/EIPS/eip-20)，用于创建可替代的代币。这些代币可以代表资产、权利、所有权、访问权限、加密货币或任何其他非独特的事物，但可以进行转移。ERC20标准允许开发者创建能够与其他产品和服务一起使用的智能合约启用的代币。

**ERC20的主要特点**：

* **可替代性**：ERC20代币是可替代的，这意味着每个代币在类型和价值上都与其他代币完全相同。
* **标准化**：ERC20提供了一组标准的函数和事件，任何遵循这些标准的智能合约都被认为是ERC20兼容的。
* **互操作性**：由于遵循统一的标准，ERC20代币可以在以太坊生态系统中的不同应用程序和服务之间轻松交互。

**ERC20标准包括的功能**：

* 转移代币（`transfer`）
* 获取账户的代币余额（`balanceOf`）
* 获取代币的总供应量（`totalSupply`）
* 授权第三方账户可以花费的代币数量（`approve`）
* 查询授权给第三方账户的代币数量（`allowance`）

**事件**：

* 代币转移（`Transfer`事件）
* 授权（`Approval`事件）

这个标准自2015年提出以来，已经成为以太坊上最广泛使用的代币标准之一。许多知名的数字货币，如Maker (MKR)、Basic Attention Token (BAT)、Augur (REP)和OMG Network (OMG)，都使用ERC20标准

### ERC20 Basic合约接口

最早的 ERC20合约仅支持部分函数与公开属性，它受到社区改进提议EIP179所提出的标准启发，共支持3个函数和一个事件，它的代码如下。

```solidity
// ERC20Basic.sol
pragma solidity ^0.4.24;

/**
 * @title ERC20Basic
 * @dev Simpler version of ERC20 interface.
 * See https://github.com/ethereum/EIPs/issues/179
 */
contract ERC20Basic {
    // Total supply of token.
    function totalSupply() public view returns (uint256);
    // Balance of a holder _who
    function balanceOf(address _who) public view returns (uint256);
    // Transfer _value from msg.sender to receiver _to.
    function transfer(address _to, uint256 _value) public returns (bool);
    // Fired when a transfer is made
    event Transfer(
        address indexed from,
        address indexed to,
        uint256 value
    );
}

```



### 接口解释

* `totalSupply()` 用于查询总供应量，所以用 `public` `view` 修饰。
* `balanceOf(address _who)` 查询某个地址有多少token
* `transfer(address _to,uint256 _value)public returns (bool)` 用于token转账

### ERC20 合约接口

ERC20合约在ERC20 Basic合约上进行了部分扩容，增加了函数定义.转让Token的过程可以由“主动转账”变为“授权索取”，在便利性上而言,主动转账更为直接，但要求知道转让对象是谁；而被动索取更适用于家长-孩子关系中管理的零花钱模式，让家长能够定授权孩子动用一部分的资金，至于资金流向何方，是孩子的决定权。

```solidity
pragma solidity ^0.4.24;

import "./ERC20Basic.sol";
/**
 * @title ERC20 interface
 * @dev Enhanced interface with allowance functions.
 * See https://github.com/ethereum/EIPs/issues/20
 */
contract ERC20 is ERC20Basic { //继承了ERC20Basic合约
    // Check the allowed value that the _owner allows the _spender to take from his balance.
    function allowance(address _owner, address _spender) public view returns (uint256);

    // Transfer _value from the balance of holder _from to the receiver _to.
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool);

    // Approve _spender to take some _value from the balance of msg.sender.
    function approve(address _spender, uint256 _value) public returns (bool);

    // Fired when an approval is made.
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
}

```

* allowance()函数：查阅授权情况。这是个公开函数，任何人都可以查询任何其他人的授权情况，函数接受两个参数，参数第一个是授权人 \_owner，第二个是被授权人 \_spender。因为查询了区块链相关的存储区，所以用public和view来修饰该函数，函数返回值是授权token的数量，采用最宽位256bit正整数来表示。
* approve()函数：允许授权行为，持有者允许被授权人转走一定数量的Token资产。这是个公开可调用函数，但修改了区块链状态，故仅采用public进行修饰。函数接收两个参数，第一个是 \_spender被授权人，第二个是 \_value即授权的 Token数量。这个函数有一定的问题，在被授权人花掉token的时候若授权方调整了数额，则有一定概率会发生授权过多的现象。该函数执行前提是检查msg.sender是否有足够的额度可供授权。
* transferFrom()函数：被授权人划走一定量的Token去往他指定的地点。这个函数公开可调用，但会修改区块链状态。在划走之前一定要检查权限，是否该人被授权动用了这些额度的Token。函数共接收三个参数 \_from、\_to、 \_value。分别代表了转移支付方，转移受付方，以及转移Token的额度。

该合约还定义了Approval 事件，该事件与Transfer事件一样，一旦发生相应的行为就会被触发，Approval事件记录了授权事件的授权方、被授权方和授权的数额。



