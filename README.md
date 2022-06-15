# 僵尸工厂学习总结
## Lesson 1
* **无符号整数：`uint`**
    `uint` 无符号数据类型， 指其值不能是负数，对于有符号的整数存在名为 `int` 的数据类型。
    >Solidity中， uint 实际上是 uint256代名词， 一个256位的无符号整数。你也可以定义位数少的uints — uint8， uint16， uint32， 等…… 但一般来讲你愿意使用简单的 uint， 除非在某些特殊情况下。
* **Solidity命名习惯**
    >习惯上函数里的变量都是以(_)开头 (但不是硬性规定) 以区别全局变量。

    例如以下的代码：
    ```solidity
    function eatHamburgers(string _name, uint _amount) {

    }
    ```
* **创建新的结构体**
    1. 申明一个结构体：
        ```
        struct Person {
            uint age;
            string name;
        }

        Person[] public people;
        ```
    2. 创建新的 Person 结构，然后把它加入到名为 people 的数组中：
        ```
        // 创建一个新的Person:
        Person satoshi = Person(172, "Satoshi");

        // 将新创建的satoshi添加进people数组:
        people.push(satoshi);
        ```
        当然我们也可以采取更简洁的写法：
        ```
        people.push(Person(16, "Vitalik"));
        ```
* **私有 / 公共函数**
    Solidity 定义的函数的属性**默认**为公共。 这就意味着任何一方 (或其它合约) 都可以调用你合约里的函数<br/>
    显然，不是什么时候都需要这样，而且这样的合约易于受到攻击。 所以将自己的函数定义为私有是一个好的编程习惯，只有当你需要外部世界调用它时才将它设置为公共。

    <b>如何定义一个私有的函数呢？</b>
    ```
    uint[] numbers;

    function _addToArray(uint _number) private {
    numbers.push(_number);
    }
    ```
    在给函数添加上`private`修饰符后，意味着只有我们合约中的其它函数才能够调用这个函数，给 numbers 数组添加新成员。
* **返回值**
    要想函数返回一个数值，按如下定义：
    ```
    string greeting = "What's up bro?";

    function sayHello() public returns (string) {
    return greeting;
    }
    ```
* **函数修饰符**
    >Solidity 语言有两类和状态读写有关的函数类型，一类是 view 函数（也称为视图函数），另一类是 pure 函数（也称为纯函数）。他们的区别是 view 函数不修改状态，pure 函数即不修改状态也不读取状态。

    调用这两种函数时，均不消耗gas。因此我们称view和pure是节约gas的利器。
    * **view**
        >可以将函数声明为 view 函数类型，这种情况下函数保证不修改状态。

        constant 曾经是 view 的别名，但在0.5.0版本中删除了这一点。
    <br/>
    * **pure**
        >在保证不读取或修改状态的情况下，函数可以被声明为 pure 函数。特别是，在编译时只给出函数输入和msg.data ，但又不知道当前区块链状态的情况下，建议使用 pure 函数。这意味着对 immutable 变量的读取可以是非纯操作。

* **类型转换**
    有时你需要变换数据类型。例如:
    ```
    uint8 a = 5;
    uint b = 6;
    // 将会抛出错误，因为 a * b 返回 uint, 而不是 uint8:
    uint8 c = a * b;
    // 我们需要将 b 转换为 uint8:
    uint8 c = a * uint8(b);
    ```
    上面, a * b 返回类型是 uint, 但是当我们尝试用 uint8 类型接收时, 就会造成潜在的错误。如果把它的数据类型转换为 uint8, 就可以了，编译器也不会出错。

* **事件**
    事件 是合约和区块链通讯的一种机制。你的前端应用“监听”某些事件，并做出反应。
    ```
    // 这里建立事件
    event IntegersAdded(uint x, uint y, uint result);

    function add(uint _x, uint _y) public {
    uint result = _x + _y;
    //触发事件，通知app
    IntegersAdded(_x, _y, result);
    return result;
    }
    ```
    你的 dapp 前端可以监听到这个事件。JavaScript 实现如下:
    ```
    YourContract.IntegersAdded(function(error, result) {
    // dosome
    })
    ```

## Lesson 2
*  映射（Mapping）和地址（Address）
    我们通过给数据库中的僵尸指定“主人”， 来支持“多玩家”模式。

    如此一来，我们需要引入2个新的数据类型：mapping（映射） 和 address（地址）。
    * Address
    以太坊区块链由 _ account _ (账户)组成，你可以把它想象成银行账户。每个帐户都有一个“地址”，你可以把它想象成银行账号。这是账户唯一的标识符，它看起来长这样：
    `0x0cE446255506E92DF41614C46F1d6df9Cc969183`

    * Mapping
        `Mapping`是另一种在 Solidity 中存储有组织数据的方法。
        ```
        //对于金融应用程序，将用户的余额保存在一个 uint类型的变量中：
        mapping (address => uint) public accountBalance;
        //或者可以用来通过userId 存储/查找的用户名
        mapping (uint => string) userIdToName;
        ```
        映射本质上是存储和查找数据所用的键-值对。在第一个例子中，键是一个 address，值是一个 uint，在第二个例子中，键是一个uint，值是一个 string。
<br/>
* Msg.sender
    在 Solidity 中，有一些全局变量可以被所有函数调用。 其中一个就是 msg.sender，它指的是当前调用者（或智能合约）的 address。
    >在 Solidity 中，功能执行始终需要从外部调用者开始。 一个合约只会在区块链上什么也不做，除非有人调用其中的函数。所以 msg.sender总是存在的。
<br/>
*  Require
    `require()`语句用于判断某个条件是否满足，若不满足括号里的条件，函数将直接返回，类似于`break`
    在调用一个函数之前，用 require 验证前置条件是非常有必要的。
<br/>
* 继承
    有个让 Solidity 的代码易于管理的功能，就是合约 inheritance (继承)：
    ```
    contract Doge {
    function catchphrase() public returns (string) {
        return "So Wow CryptoDoge";
    }
    }

    contract BabyDoge is Doge {
    function anotherCatchphrase() public returns (string) {
        return "Such Moon BabyDoge";
    }
    }
    ```
    由于 `BabyDoge` 是从 `Doge` 那里继承过来的。 这意味着当你编译和部署了 `BabyDoge`，它将可以访问 `catchphrase()` 和 `anotherCatchphrase()`和其他我们在 `Doge` 中定义的其他公共函数。<br/>
    这可以用于逻辑继承（比如表达子类的时候，Cat 是一种 Animal）。 但也可以简单地将类似的逻辑组合到不同的合约中以组织代码。
<br/>
* Storage与Memory
    在 Solidity 中，有两个地方可以存储变量 —— `storage` 或 `memory`。<br/>
    Storage 变量是指永久存储在区块链中的变量。 Memory 变量则是临时的，当外部函数对某合约调用完成时，内存型变量即被移除。 你可以把它想象成存储在你电脑的硬盘或是RAM中数据的关系。
    >大多数时候你都用不到这些关键字，默认情况下 Solidity 会自动处理它们。 也有一些情况下，你需要手动声明存储类型，主要用于处理函数内的 _ 结构体 _ 和 _ 数组 _ 时：
    ```
    contract SandwichFactory {
    struct Sandwich {
        string name;
        string status;
    }

    Sandwich[] sandwiches;

    function eatSandwich(uint _index) public {
        // Sandwich mySandwich = sandwiches[_index];

        // ^ 看上去很直接，不过 Solidity 将会给出警告
        // 告诉你应该明确在这里定义 `storage` 或者 `memory`。

        // 所以你应该明确定义 `storage`:
        Sandwich storage mySandwich = sandwiches[_index];
        // ...这样 `mySandwich` 是指向 `sandwiches[_index]`的指针
        // 在存储里，另外...
        mySandwich.status = "Eaten!";
        // ...这将永久把 `sandwiches[_index]` 变为区块链上的存储

        // 如果你只想要一个副本，可以使用`memory`:
        Sandwich memory anotherSandwich = sandwiches[_index + 1];
        // ...这样 `anotherSandwich` 就仅仅是一个内存里的副本了
        // 另外
        anotherSandwich.status = "Eaten!";
        // ...将仅仅修改临时变量，对 `sandwiches[_index + 1]` 没有任何影响
        // 不过你可以这样做:
        sandwiches[_index + 1] = anotherSandwich;
        // ...如果你想把副本的改动保存回区块链存储
    }
    }
    ```
* internal 和 external
    除 `public` 和 `private` 属性之外，Solidity 还使用了另外两个描述函数可见性的修饰词：`internal`（内部） 和 `external`（外部）。
    * internal
        internal 和 private 类似，不过， 如果某个合约继承自其父合约，这个合约即可以访问父合约中定义的“内部”函数。
    * external
        external 与public 类似，只不过这些函数只能在合约之外调用 - 它们不能被合约内的其他函数调用。
<br/>

* **与其他合约交互**
    1. 定义接口
        假设在区块链上有这么一个合约：
        ```
        contract LuckyNumber {
        mapping(address => uint) numbers;

        function setNum(uint _num) public {
            numbers[msg.sender] = _num;
        }

        function getNum(address _myAddress) public view returns (uint) {
            return numbers[_myAddress];
        }
        }
        ```
        这是个很简单的合约，您可以用它存储自己的幸运号码，并将其与您的以太坊地址关联。 这样其他人就可以通过您的地址查找您的幸运号码了。<br/>
        现在假设我们有一个外部合约，使用 getNum 函数可读取其中的数据。
        >首先，我们定义 LuckyNumber 合约的 interface ：
        ```
        contract NumberInterface {
        function getNum(address _myAddress) public view returns (uint);
        }
        ```
        请注意，这个过程虽然看起来像在定义一个合约，但其实内里不同.<br/>
        首先，我们只声明了要与之交互的函数 —— 在本例中为 getNum —— 在其中我们没有使用到任何其他的函数或状态变量。
        其次，我们并没有使用大括号（{ 和 }）定义函数体，我们单单用分号（;）结束了函数声明。这使它看起来像一个合约框架。<br/>
        编译器就是靠这些特征认出它是一个接口的。
    2. 实现接口
        我们可以在合约中这样使用：
        ```
        contract MyContract {
        address NumberInterfaceAddress = 0xab38...;
        // ^ 这是FavoriteNumber合约在以太坊上的地址
        NumberInterface numberContract = NumberInterface(NumberInterfaceAddress);
        // 现在变量 `numberContract` 指向另一个合约对象

        function someFunction() public {
            // 现在我们可以调用在那个合约中声明的 `getNum`函数:
            uint num = numberContract.getNum(msg.sender);
            // ...在这儿使用 `num`变量做些什么
        }
        }
        ```
        通过这种方式，只要将您合约的可见性设置为public(公共)或external(外部)，它们就可以与以太坊区块链上的任何其他合约进行交互。
* 处理多返回值
    ```
    function multipleReturns() internal returns(uint a, uint b, uint c) {
    return (1, 2, 3);
    }

    function processMultipleReturns() external {
    uint a;
    uint b;
    uint c;
    // 这样来做批量赋值:
    (a, b, c) = multipleReturns();
    }

    // 或者如果我们只想返回其中一个变量:
    function getLastReturnValue() external {
    uint c;
    // 可以对其他字段留空:
    (,,c) = multipleReturns();
    }
    ```
## Lesson 3
* **Ownable Contracts**
    指定合约的“所有权” - 就是说，给它指定一个主人（没错，就是您），只有主人对它享有特权。
    * OpenZeppelin库的Ownable 合约
        下面是一个 Ownable 合约的例子： 来自 _ OpenZeppelin _ Solidity 库的 Ownable 合约。 
        ```
        /**
        * @title Ownable
        * @dev The Ownable contract has an owner address, and provides basic authorization control
        * functions, this simplifies the implementation of "user permissions".
        */
        contract Ownable {
        address public owner;
        event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

        /**
        * @dev The Ownable constructor sets the original `owner` of the contract to the sender
        * account.
        */
        function Ownable() public {
            owner = msg.sender;
        }

        /**
        * @dev Throws if called by any account other than the owner.
        */
        modifier onlyOwner() {
            require(msg.sender == owner);
            _;
        }

        /**
        * @dev Allows the current owner to transfer control of the contract to a newOwner.
        * @param newOwner The address to transfer ownership to.
        */
        function transferOwnership(address newOwner) public onlyOwner {
            require(newOwner != address(0));
            OwnershipTransferred(owner, newOwner);
            owner = newOwner;
        }
        }
        ```
        函数修饰符：modifier onlyOwner()。 修饰符跟函数很类似，不过是用来修饰其他已有函数用的， 在其他语句执行前，为它检查下先验条件。 在这个例子中，我们就可以写个修饰符 onlyOwner 检查下调用者，确保只有合约的主人才能运行本函数。我们下一章中会详细讲述修饰符，以及那个奇怪的_;。
        >所以Ownable 合约基本都会这么干：
        1.合约创建，构造函数先行，将其 owner 设置为msg.sender（其部署者）
        2.为它加上一个修饰符 onlyOwner，它会限制陌生人的访问，将访问某些函数的权限锁定在 owner 上。
        3.允许将合约所有权转让给他人。
<br/>

* **onlyOwner 函数修饰符**
    我们仔细读读 `onlyOwner`:
    ```
    modifier onlyOwner() {
    require(msg.sender == owner);
    _;
    }
    ```
    `onlyOwner` 函数修饰符是这么用的：
    ```
    contract MyContract is Ownable {
    event LaughManiacally(string laughter);

    //注意！ `onlyOwner`上场 :
    function likeABoss() external onlyOwner {
        LaughManiacally("Muahahahaha");
    }
    }
    ```
    注意 likeABoss 函数上的 onlyOwner 修饰符。 当你调用 likeABoss 时，首先执行 onlyOwner 中的代码， 执行到 onlyOwner 中的 _; 语句时，程序再返回并执行 likeABoss 中的代码。

    可见，尽管函数修饰符也可以应用到各种场合，但最常见的还是放在函数执行之前添加快速的 require检查
<br/>

* **省 gas 的招数：结构封装 （Struct packing）**
    通常情况下我们不会考虑使用 uint 变种，因为无论如何定义 uint的大小，Solidity 为它保留256位的存储空间。例如，使用 uint8 而不是uint（uint256）不会为你节省任何 gas.  <br/>
    除非，把 uint 绑定到 struct 里面。
    如果一个 struct 中有多个 uint，则尽可能使用较小的 uint, Solidity 会将这些 uint 打包在一起，从而占用较少的存储空间。例如：
    ```
    struct NormalStruct {
    uint a;
    uint b;
    uint c;
    }

    struct MiniMe {
    uint32 a;
    uint32 b;
    uint c;
    }

    // 因为使用了结构打包，`mini` 比 `normal` 占用的空间更少
    NormalStruct normal = NormalStruct(10, 20, 30);
    MiniMe mini = MiniMe(10, 20, 30); 
    ```
<br/>

* **时间单位**
    Solidity 使用自己的本地时间单位。
    变量 now 将返回当前的unix时间戳（自1970年1月1日以来经过的秒数）。
    >Solidity 还包含秒(seconds)，分钟(minutes)，小时(hours)，天(days)，周(weeks) 和 年(years) 等时间单位。它们都会转换成对应的秒数放入 uint 中。所以 1分钟 就是 60，1小时是 3600（60秒×60分钟），1天是86400（24小时×60分钟×60秒），以此类推。

    下面是一些使用时间单位的实用案例：
    ```
    uint lastUpdated;

    // 将‘上次更新时间’ 设置为 ‘现在’
    function updateTimestamp() public {
    lastUpdated = now;
    }

    // 如果到上次`updateTimestamp` 超过5分钟，返回 'true'
    // 不到5分钟返回 'false'
    function fiveMinutesHavePassed() public view returns (bool) {
    return (now >= (lastUpdated + 5 minutes));
    }
    ```
<br/>

* **带参数的函数修饰符**
    之前我们已经读过一个简单的函数修饰符了：onlyOwner。函数修饰符也可以带参数。例如：
    ```
    // 存储用户年龄的映射
    mapping (uint => uint) public age;

    // 限定用户年龄的修饰符
    modifier olderThan(uint _age, uint _userId) {
    require(age[_userId] >= _age);
    _;
    }

    // 必须年满16周岁才允许开车 (至少在美国是这样的).
    // 我们可以用如下参数调用`olderThan` 修饰符:
    function driveCar(uint _userId) public olderThan(16, _userId) {
    // 其余的程序逻辑
    }
    ```
    这表明我们其实可以自己创建一个属于自己合约的修饰符函数，进而实现方法调用的控制。
<br/>

## Lesson 4
* **可支付（payable）修饰符**
    `payable` 方法是让 Solidity 和以太坊变得如此酷的一部分 —— 它们是一种可以接收以太的特殊函数。
    一个栗子：
    ```
    contract OnlineStore {
    function buySomething() external payable {
        // 检查以确定0.001以太发送出去来运行函数:
        require(msg.value == 0.001 ether);
        // 如果为真，一些用来向函数调用者发送数字内容的逻辑
        transferThing(msg.sender);
    }
    }
    ```
    在这里，msg.value 是一种可以查看向合约发送了多少以太的方法，另外 ether 是一个內建单元。
<br/>

* **随机数**
    Solidity 中最好的随机数生成器是 `keccak256` 哈希函数.
    我们可以这样来生成一些随机数
    ```
    // 生成一个0到100的随机数:
    uint randNonce = 0;
    uint random = uint(keccak256(now, msg.sender, randNonce)) % 100;
    randNonce++;
    uint random2 = uint(keccak256(now, msg.sender, randNonce)) % 100;
    ```
    这个方法首先拿到 now 的时间戳、 msg.sender、 以及一个自增数 nonce （一个仅会被使用一次的数，这样我们就不会对相同的输入值调用一次以上哈希函数了）。<br/>
    然后利用 keccak 把输入的值转变为一个哈希值, 再将哈希值转换为 uint, 然后利用 % 100 来取最后两位, 就生成了一个0到100之间随机数了。
    >这个方法其实并不安全，很容易受到不诚实的节点攻击，但同时，攻击的代价也是很大的。
    关于安全的随机数生成可以查看这个[StackOverFlow上的讨论](https://ethereum.stackexchange.com/questions/191/how-can-i-securely-generate-a-random-number-in-my-smart-contract)


