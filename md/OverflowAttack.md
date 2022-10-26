# 溢出攻击
---

    author：Thomas_Xu
在介绍溢出攻击前，让我们先来了解一下solidity中溢出和下溢。
* 溢出
    假设我们有一个 uint8, 只能存储8 bit数据。这意味着我们能存储的最大数字就是二进制 11111111 (或者说十进制的 2^8 - 1 = 255).<br/>
    来看看下面的代码。最后 number 将会是什么值？<br/>
    ```
    uint8 number = 255;
    number++; //number = 0
    ```
    在这个例子中，我们导致了溢出 — 虽然我们加了1， 但是number 出乎意料地等于 0了。<br/>
    ![](imageS/overflow1.png)
    下溢(underflow)也类似，如果你从一个等于 0 的 uint8 减去 1, 它将变成 255 (因为 uint 是无符号的，其不能等于负数)。
# 漏洞分析
上述就是在solidity中，数据溢出的原理，那么在智能合约中，由于合约代码考虑不规范，可能会导致合约数据溢出漏洞，下来举例一个在以太坊公链中有数据溢出BUG的合约代码：
```
pragma solidity ^0.4.18;

contract Hexagon {
/* Main information */
string public constant name = "Hexagon";
string public constant symbol = "HXG";
uint8 public constant decimals = 4;
uint8 public constant burnPerTransaction = 2;
uint256 public constant initialSupply = 420000000000000;
uint256 public currentSupply = initialSupply;

/* Create array with balances */
mapping (address => uint256) public balanceOf;
/* Create array with allowance */
mapping (address => mapping (address => uint256)) public allowance;

/* Constructor */
function Hexagon() public {
    /* Give creator all initial supply of tokens */
    balanceOf[msg.sender] = initialSupply;
}

/* PUBLIC */
/* Send tokens */
function transfer(address _to, uint256 _value) public returns (bool success) {
    _transfer(msg.sender, _to, _value);

    return true;
}

/* Return current supply */
function totalSupply() public constant returns (uint) {
    return currentSupply;
}

/* Burn tokens */
function burn(uint256 _value) public returns (bool success) {
    /* Check if the sender has enough */
    require(balanceOf[msg.sender] >= _value);
    /* Subtract from the sender */
    balanceOf[msg.sender] -= _value;
    /* Send to the black hole */
    balanceOf[0x0] += _value;
    /* Update current supply */
    currentSupply -= _value;
    /* Notify network */
    Burn(msg.sender, _value);

    return true;
}

/* Allow someone to spend on your behalf */
function approve(address _spender, uint256 _value) public returns (bool success) {
    /* Check if the sender has already  */
    require(_value == 0 || allowance[msg.sender][_spender] == 0);
    /* Add to allowance  */
    allowance[msg.sender][_spender] = _value;
    /* Notify network */
    Approval(msg.sender, _spender, _value);

    return true;
}

/* Transfer tokens from allowance */
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
    /* Prevent transfer of not allowed tokens */
    require(allowance[_from][msg.sender] >= _value);
    /* Remove tokens from allowance */
    allowance[_from][msg.sender] -= _value;

    _transfer(_from, _to, _value);

    return true;
}

/* INTERNAL */
function _transfer(address _from, address _to, uint _value) internal {
    /* Prevent transfer to 0x0 address. Use burn() instead  */
    require (_to != 0x0);
    /* Check if the sender has enough */
    //问题代码，数据溢出的攻击点
    require (balanceOf[_from] >= _value + burnPerTransaction);
    /* Check for overflows */
    require (balanceOf[_to] + _value > balanceOf[_to]);
    /* Subtract from the sender */
    balanceOf[_from] -= _value + burnPerTransaction;
    /* Add the same to the recipient */
    balanceOf[_to] += _value;
    /* Apply transaction fee */
    balanceOf[0x0] += burnPerTransaction;
    /* Update current supply */
    currentSupply -= burnPerTransaction;
    /* Notify network */
    Burn(_from, burnPerTransaction);
    /* Notify network */
    Transfer(_from, _to, _value);
}

/* Events */
event Transfer(address indexed from, address indexed to, uint256 value);
event Burn(address indexed from, uint256 value);
event Approval(address indexed _owner, address indexed _spender, uint256 _value);
}
```
在合约的转账代码 function _transfer(address _from, address _to, uint _value) internal 实现中，判断转账支付方账户是否具有足够的余额，有如下的判断语句：
```
    //问题代码，数据溢出的攻击点
    require (balanceOf[_from] >= _value + burnPerTransaction);
```
加入攻击者给`_value`一个很大的值，那么在加上`burnPerTransaction`后很可能会发生溢出，相加后的结果很小，导致`require`发生错误的判断。结果给接收方地址增加一笔非常大的TOKEN。下面将举个例子说明：
>假设合约中 burnPerTransaction = 0xf ，
所以当转账_value为0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0时，
_value + burnPerTransaction =0 ，即可成功攻击，为balanceOf[_to]增加大量代币。

# 漏洞避免
* 该段代码安全的写法应该是这样的：
    ```
    require (balanceOf[_from] >= _value );
    require (balanceOf[_from] >= burnPerTransaction);
    require (balanceOf[_from] >= _value + burnPerTransaction);
    ```

* **使用SafeMath**
    为了避免溢出和下溢的情况，OpenZeppelin 建立了一个叫做 SafeMath 的 库(library)，默认情况下可以防止这些问题。
    一个库 是 Solidity 中一种特殊的合约。其中一个有用的功能是给原始数据类型增加一些方法。<br/>
    比如，使用 SafeMath 库的时候，我们将使用 using SafeMath for uint256 这样的语法。 SafeMath 库有四个方法 — add， sub， mul， 以及 div。现在我们可以这样来让 uint256 调用这些方法：
    ```
    using SafeMath for uint256;
    
    uint256 a = 5;
    uint256 b = a.add(3); // 5 + 3 = 8
    uint256 c = a.mul(2); // 5 * 2 = 10
    ```
    我们注意到了一个不常见的语法`using···for···`这是因为SafeMath源码使用了library关键字，库允许我们使用 using 关键字，它可以自动把库的所有方法添加给一个数据类型。<br/>
    我们来看一下SafeMath的源码：
    ```
    library SafeMath {
    
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
        return 0;
        }
        uint256 c = a * b;
        assert(c / a == b);
        return c;
    }
    
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        // assert(b > 0); // Solidity automatically throws when dividing by 0
        uint256 c = a / b;
        // assert(a == b * c + a % b); // There is no case in which this doesn't hold
        return c;
    }
    
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        assert(b <= a);
        return a - b;
    }
    
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        assert(c >= a);
        return c;
    }
    }
    
    ```
    解释一下源码中的`assert`
    
    >assert 和 require 相似，若结果为否它就会抛出错误。 assert 和 require 区别在于，require 若失败则会返还给用户剩下的 gas， assert则不会。所以大部分情况下，你写代码的时候会比较喜欢 require，assert 只在代码可能出现严重错误的时候使用，比如 uint 溢出。