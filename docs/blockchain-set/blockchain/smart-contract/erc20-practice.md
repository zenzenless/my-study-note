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

TODO:
