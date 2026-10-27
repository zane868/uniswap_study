---
eip: 20
title: 代币标准
author: Fabian Vogelsteller <fabian@ethereum.org>, Vitalik Buterin <vitalik.buterin@ethereum.org>
type: 标准跟踪
category: ERC
status: 最终
created: 2015-11-19
---

## 简单总结

代币的标准接口。


## 摘要

以下标准允许在智能合约中实现代币的标准 API。
该标准提供基本功能以转移代币，并允许代币被批准以便可以被其他链上第三方使用。


## 动机

标准接口允许以太坊上的任何代币被其他应用重用：从钱包到去中心化交易所。


## 规范

## 代币
### 方法

**注意**：
 - 以下规范使用 Solidity `0.4.17`（或更高版本）的语法
 - 调用者必须处理 `returns (bool success)` 中的 `false`。调用者不得假设 `false` 永远不会被返回！


#### name

返回代币的名称 - 例如 `"MyToken"`。

可选 - 此方法可用于提高可用性，但接口和其他合约不得期望这些值存在。


``` js
function name() public view returns (string)
```


#### symbol

返回代币的符号。例如 "HIX"。

可选 - 此方法可用于提高可用性，但接口和其他合约不得期望这些值存在。

``` js
function symbol() public view returns (string)
```



#### decimals

返回代币使用的小数位数 - 例如 `8`，意味着将代币数量除以 `100000000` 以获得其用户表示。

可选 - 此方法可用于提高可用性，但接口和其他合约不得期望这些值存在。

``` js
function decimals() public view returns (uint8)
```


#### totalSupply

返回代币的总供应量。

``` js
function totalSupply() public view returns (uint256)
```



#### balanceOf

返回另一个地址为 `_owner` 的账户余额。

``` js
function balanceOf(address _owner) public view returns (uint256 balance)
```



#### transfer

将 `_value` 数量的代币转移到地址 `_to`，并必须触发 `Transfer` 事件。
如果消息调用者的账户余额不足以支出，则该函数应抛出异常。

*注意* 0 值的转移必须被视为正常转移并触发 `Transfer` 事件。

``` js
function transfer(address _to, uint256 _value) public returns (bool success)
```



#### transferFrom

将 `_value` 数量的代币从地址 `_from` 转移到地址 `_to`，并必须触发 `Transfer` 事件。

`transferFrom` 方法用于提取工作流，允许合约代表您转移代币。
这可以用于例如允许合约代表您转移代币和/或收取子货币的费用。
除非 `_from` 账户通过某种机制故意授权消息的发送者，否则该函数应抛出异常。

*注意* 0 值的转移必须被视为正常转移并触发 `Transfer` 事件。

``` js
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
```



#### approve

允许 `_spender` 从您的账户多次提取，最多到 `_value` 数量。如果再次调用此函数，则会用 `_value` 覆盖当前的授权。

**注意**：为了防止像 [这里描述的攻击向量](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/) 和 [这里讨论的](https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729)，客户端应确保以这样的方式创建用户界面，即在将授权设置为另一个值之前，先将其设置为 `0`。
尽管合约本身不应强制执行此操作，以允许与之前部署的合约向后兼容。

``` js
function approve(address _spender, uint256 _value) public returns (bool success)
```


#### allowance

返回 `_spender` 仍然被允许从 `_owner` 提取的金额。

``` js
function allowance(address _owner, address _spender) public view returns (uint256 remaining)
```



### 事件


#### Transfer

在代币转移时必须触发，包括零值转移。

创建新代币的代币合约应在代币创建时触发 `Transfer` 事件，`_from` 地址设置为 `0x0`。

``` js
event Transfer(address indexed _from, address indexed _to, uint256 _value)
```



#### Approval

在任何成功调用 `approve(address _spender, uint256 _value)` 时必须触发。

``` js
event Approval(address indexed _owner, address indexed _spender, uint256 _value)
```



## 实现

以太坊网络上已经部署了许多符合 ERC20 的代币。
不同的实现由不同的团队编写，具有不同的权衡：从节省 gas 到提高安全性。

#### 示例实现可在以下位置找到
- [OpenZeppelin 实现](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)


## 历史

与此标准相关的历史链接：

- Vitalik Buterin 的原始提案：https://github.com/ethereum/wiki/wiki/Standardized_Contract_APIs/499c882f3ec123537fc2fccd57eaa29e6032fe4a
- Reddit 讨论：https://www.reddit.com/r/ethereum/comments/3n8fkn/lets_talk_about_the_coin_standard/
- 原始问题 #20：https://github.com/ethereum/EIPs/issues/20



## 版权
版权及相关权利通过 [CC0](../LICENSE.md) 放弃。