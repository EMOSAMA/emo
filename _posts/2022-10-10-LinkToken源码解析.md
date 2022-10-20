# 简介
link token 是继承于 ERC677 的一个代币合约。ERC677 和 ERC20 比较相似，只不过 ERC677 增加了 transferAndCall 方法，支持在代币转账的时候调用特定的函数。这个代币合约比较简单，大概看看代码就好。

# 代码分析
## ERC677
相比起 ERC20 ，ERC677 添加了一个新的 transferAndCall 函数，用于在转账的时候，不仅仅会发生代币转移，还会根据传入的 data 触发相应的逻辑。
```java
contract ERC677 is linkERC20 {
  function transferAndCall(address to, uint value, bytes data) returns (bool success);

  event Transfer(address indexed from, address indexed to, uint value, bytes data);
}
```

## ERC677Token
在这个合约当中主要单独实现了 ERC677 中单独提出的 transferAndCall 相关的功能。整体代码比较简单。
```java
pragma solidity ^0.4.11;


import "./interfaces/ERC677.sol";
import "./interfaces/ERC677Receiver.sol";


contract ERC677Token is ERC677 {

  /**
  * @dev transfer token to a contract address with additional data if the recipient is a contact.
  * @param _to The address to transfer to.
  * @param _value The amount to be transferred.
  * @param _data The extra data to be passed to the receiving contract.
  */
  function transferAndCall(address _to, uint _value, bytes _data)
    public
    returns (bool success)
  { // 实现 transferAndCall
    super.transfer(_to, _value); // 转账
    Transfer(msg.sender, _to, _value, _data); // 转账事件
    if (isContract(_to)) { // 判断地址上的账户是不是合约账户
      contractFallback(_to, _value, _data); // 调用 oracle 的方法 onTokenTransfer
    }
    return true;
  }


  // PRIVATE

  function contractFallback(address _to, uint _value, bytes _data)
    private
  {
    ERC677Receiver receiver = ERC677Receiver(_to); // oracle 都是实现了里面的 onTokenTransfer 方法的
    receiver.onTokenTransfer(msg.sender, _value, _data);
  }

  function isContract(address _addr)
    private
    returns (bool hasCode)
  {
    uint length;
    assembly { length := extcodesize(_addr) }
    return length > 0;
  }

}
```

## BasicToken
实现了基础的账户查看和转账相关的基础功能
```java
pragma solidity ^0.4.24;


import { ERC20Basic as linkERC20Basic } from "../interfaces/ERC20Basic.sol";
import { SafeMathChainlink as linkSafeMath } from "./SafeMathChainlink.sol";


/**
 * @title Basic token
 * @dev Basic version of StandardToken, with no allowances. 
 */
contract BasicToken is linkERC20Basic {
  using linkSafeMath for uint256;

  mapping(address => uint256) balances;

  /**
  * @dev transfer token for a specified address
  * @param _to The address to transfer to.
  * @param _value The amount to be transferred.
  */
  function transfer(address _to, uint256 _value) returns (bool) {
    balances[msg.sender] = balances[msg.sender].sub(_value);
    balances[_to] = balances[_to].add(_value);
    Transfer(msg.sender, _to, _value);
    return true;
  }

  /**
  * @dev Gets the balance of the specified address.
  * @param _owner The address to query the the balance of. 
  * @return An uint256 representing the amount owned by the passed address.
  */
  function balanceOf(address _owner) constant returns (uint256 balance) {
    return balances[_owner];
  }

}

```

## StandardToken
这里面主要实现了 allowance 相关的功能。允许代币委托。
```java
pragma solidity ^0.4.11;


import { BasicToken as linkBasicToken } from "./BasicToken.sol";
import { ERC20 as linkERC20 } from "../interfaces/ERC20.sol";


/**
 * @title Standard ERC20 token
 *
 * @dev Implementation of the basic standard token.
 * @dev https://github.com/ethereum/EIPs/issues/20
 * @dev Based on code by FirstBlood: https://github.com/Firstbloodio/token/blob/master/smart_contract/FirstBloodToken.sol
 */
contract StandardToken is linkERC20, linkBasicToken {

  mapping (address => mapping (address => uint256)) allowed;


  /**
   * @dev Transfer tokens from one address to another
   * @param _from address The address which you want to send tokens from
   * @param _to address The address which you want to transfer to
   * @param _value uint256 the amount of tokens to be transferred
   */
  function transferFrom(address _from, address _to, uint256 _value) returns (bool) {
    var _allowance = allowed[_from][msg.sender];

    // Check is not needed because sub(_allowance, _value) will already throw if this condition is not met
    // require (_value <= _allowance);

    balances[_from] = balances[_from].sub(_value);
    balances[_to] = balances[_to].add(_value);
    allowed[_from][msg.sender] = _allowance.sub(_value);
    Transfer(_from, _to, _value);
    return true;
  }

  /**
   * @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
   * @param _spender The address which will spend the funds.
   * @param _value The amount of tokens to be spent.
   */
  function approve(address _spender, uint256 _value) returns (bool) {
    allowed[msg.sender][_spender] = _value;
    Approval(msg.sender, _spender, _value);
    return true;
  }

  /**
   * @dev Function to check the amount of tokens that an owner allowed to a spender.
   * @param _owner address The address which owns the funds.
   * @param _spender address The address which will spend the funds.
   * @return A uint256 specifying the amount of tokens still available for the spender.
   */
  function allowance(address _owner, address _spender) constant returns (uint256 remaining) {
    return allowed[_owner][_spender];
  }
  
    /*
   * approve should be called when allowed[_spender] == 0. To increment
   * allowed value is better to use this function to avoid 2 calls (and wait until 
   * the first transaction is mined)
   * From MonolithDAO Token.sol
   */
  function increaseApproval (address _spender, uint _addedValue) 
    returns (bool success) {
    allowed[msg.sender][_spender] = allowed[msg.sender][_spender].add(_addedValue);
    Approval(msg.sender, _spender, allowed[msg.sender][_spender]);
    return true;
  }

  function decreaseApproval (address _spender, uint _subtractedValue) 
    returns (bool success) {
    uint oldValue = allowed[msg.sender][_spender];
    if (_subtractedValue > oldValue) {
      allowed[msg.sender][_spender] = 0;
    } else {
      allowed[msg.sender][_spender] = oldValue.sub(_subtractedValue);
    }
    Approval(msg.sender, _spender, allowed[msg.sender][_spender]);
    return true;
  }

}
```

## LinkToken
这里是 LinkToken 的合约，定义了 LINK 代币的基本信息，并且给涉及转账和委托的方法添加了 validRecipient 修饰器约束。这个修饰器主要是限制转账或者委托给 **无效地址或者自己本身** 。
```java
pragma solidity ^0.4.11;


import "./ERC677Token.sol";
import { StandardToken as linkStandardToken } from "./vendor/StandardToken.sol";


contract LinkToken is linkStandardToken, ERC677Token {

  // 发行一个代币需要的基本常量信息
  uint public constant totalSupply = 10**27;
  string public constant name = "ChainLink Token";
  uint8 public constant decimals = 18;
  string public constant symbol = "LINK";

  function LinkToken()
    public
  {
    // 初始化账户
    balances[msg.sender] = totalSupply;
  }

  /**
  * @dev transfer token to a specified address with additional data if the recipient is a contract.
  * @param _to The address to transfer to.
  * @param _value The amount to be transferred.
  * @param _data The extra data to be passed to the receiving contract.
  */
  function transferAndCall(address _to, uint _value, bytes _data)
    public
    validRecipient(_to)
    returns (bool success)
  {
    return super.transferAndCall(_to, _value, _data);
  }

  // 给下面涉及转账和委托的函数添加了 validRecipient 修饰器约束。主要是限制转账或者委托的接收方必须是有效地址，而不不能自己转账给自己或者自己委托给自己

  /**
  * @dev transfer token to a specified address.
  * @param _to The address to transfer to.
  * @param _value The amount to be transferred.
  */
  function transfer(address _to, uint _value)
    public
    validRecipient(_to)
    returns (bool success)
  {
    return super.transfer(_to, _value);
  }

  /**
   * @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
   * @param _spender The address which will spend the funds.
   * @param _value The amount of tokens to be spent.
   */
  function approve(address _spender, uint256 _value)
    public
    validRecipient(_spender)
    returns (bool)
  {
    return super.approve(_spender,  _value);
  }

  /**
   * @dev Transfer tokens from one address to another
   * @param _from address The address which you want to send tokens from
   * @param _to address The address which you want to transfer to
   * @param _value uint256 the amount of tokens to be transferred
   */
  function transferFrom(address _from, address _to, uint256 _value)
    public
    validRecipient(_to)
    returns (bool)
  {
    return super.transferFrom(_from, _to, _value);
  }


  // MODIFIERS

  modifier validRecipient(address _recipient) {
    require(_recipient != address(0) && _recipient != address(this));
    _;
  }

}

```
