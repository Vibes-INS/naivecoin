# 第三章 交易

- [概览](#概览)
- [非对称加密和签名](#非对称加密和签名)
- [私钥和公钥](#私钥和公钥)
- [交易概览](#交易概览)
- [交易outputs](#交易outputs)
- [交易inputs](#交易inputs)
- [交易数据结构](#交易数据结构)
- [交易id](#交易id)
- [交易签名](#交易签名)
- [未消费的交易outputs](#未消费的交易outputs)
- [未消费交易outputs清单更新](#未消费交易outputs清单更新)
- [交易有效性验证](#交易有效性验证)
- [原始交易](#原始交易)
- [测试体验](#测试体验)
- [小结](#小结)

### 概览
本章我们将引入加密货币中的**交易**机制。有了交易这个机制之后，我们的区块链将会从一个只有基本功能的区块链华丽转身成一个加密货币系统。 最终我们就能通过指定目标用户的地址，和对应的用户进行加密货币交易。 当然，交易之前你必须能证明你是该货币的持有者。

为了达到这个目标，我们还需要了解不少的一些概念。 比如非对称加密，签名，交易inputs(可以理解成发起者)和交易outputs(可以理解成接收者)等。

### 非对称加密和签名

在非对称加密中，你会有一个公钥-私钥对，其中公钥是通过私钥演绎生成，但私钥不能通过公钥演绎出来。钥如其名，公钥可以安全的分享给其他人，但是私钥只能自己保存。

任何消息都能通过私钥进行加密而生成一个签名。任何拥有对应公钥的人，都能通过该公钥将信息进行解密，进而验证该签名的有效性。

![](https://lhartikk.github.io/assets/Digital_signatures.png)

这里我们将会用到一个叫做[elliptic](https://github.com/indutny/elliptic)的库来实现非对称加密，该库用的是椭圆曲线加密算法(ECDSA)。

总的来说，我们的加密货币系统中会用到两套不同的加密算法来实现不同的目的:

- SHA256: 在工作量证明中对区块进行哈希计算，保证数据的完整性和不可篡改性。
- 非对称加密: 用户身份验证。

### 私钥和公钥

一个有效的ECDSA中的私钥是一个32字节的字符串，示例如下：
```
19f128debc1b9122da0635954488b208b829879cf13b3d6cac5d1260c0fd967c
```

一个有效的公钥是由‘04’开头，紧接着64个字节的字符串，示例如下：
```
04bfcab8722991ae774db48f934ca79cfb7dd991229153b9f732ba5334aafcd8e7266e47076996b55a14bf9913ee3145ce0cfc1372ada8ada74bd287450313534a
```
公钥从私钥演绎生成。公钥将会被用作交易的接收者(即接收者地址)。

### 交易概览

在我们编写任何代码之前，我们先对交易的结构进行大致的了解。交易将会由两个部分组成：inputs和outputs。 outputs指定了加密货币交易的接收者(可能会有多个)，inputs则是用来证明用来交易的货币是确实存在且是被交易发送者所持有的。每个交易中的inputs下的项都会指向一个已经存在的「未消费output」，本章节后面可以看到系统会拿所指向的output的地址(即代表某用户的公钥)来对本交易的签名进行解密，如果能解开，则代表该output确实属于该用户。

![](https://lhartikk.github.io/assets/transactions.png)

### 交易outputs
交易output(TxOut)结构由地址和数量两个成员变量组成。 数量代表了交易的虚拟货币的数量。地址就是一个ECDSA的公钥，代表接收者。意味着只有拥有对应地址的私钥的用户才能访问对应的加密货币。

```typescript
class TxOut {
    public address: string;
    public amount: number;

    constructor(address: string, amount: number) {
        this.address = address;
        this.amount = amount;
    }
}
```

### 交易inputs
交易input(TxIn)结构提供了加密货币的来源信息。每一个txIn都会通过txOutId指向此前的一次交易(交易id)，本次交易的加密货币就是从此前的该次交易解锁出来的，解锁后的货币就能在output中被发送给接收者了。结构中的signature这个属性则是发送者用自己私钥进行加密的签名(签名数据为本次交易的id)，通过用指向的前一次交易中对应output中的地址(即公钥)对该签名进行验证，即能证明该发送者是该交易及所指向的前一次交易的拥有者。

``` typescript
class TxIn {
    public txOutId: string;
    public txOutIndex: number;
    public signature: string;
}
```

需要注意的是这里保存的只是通过私钥进行的签名，而不是私钥本身。在区块链的整个系统中，仅仅存在公钥和签名，而不会出现私钥。

总的来说，我们可以认为inputs解锁了对应的加密货币，而outputs重新锁定这些货币并让接收者成为新的持有者。

![](https://lhartikk.github.io/assets/transactions2.png)

### 交易数据结构

当定义好上面的交易inputs和outputs之后，交易本身的数据结构就变得异常简单了。

``` typescript
class Transaction {
    public id: string;
    public txIns: TxIn[];
    public txOuts: TxOut[];
}
```
### 交易id
交易id代表了一次交易的唯一性，是通过对交易数据结构中的内容做哈希计算出来的。这里要注意的是我们并没有包含发起者的签名，这个会在之后添加。

``` typescript
const getTransactionId = (transaction: Transaction): string => {
    const txInContent: string = transaction.txIns
        .map((txIn: TxIn) => txIn.txOutId + txIn.txOutIndex)
        .reduce((a, b) => a + b, '');

    const txOutContent: string = transaction.txOuts
        .map((txOut: TxOut) => txOut.address + txOut.amount)
        .reduce((a, b) => a + b, '');

    return CryptoJS.SHA256(txInContent + txOutContent).toString();
};
```

### 交易签名
通过签名来保证交易内容不被修改是非常重要的。因为交易都是被公开的，任何人都能够访问所有的交易，就算这些交易还没有来得及加入到区块链当中去。

当对交易进行签名时，事实上我们只会对交易id进行签名。也就是说，参考前面的交易id的生成，只要交易中任何的一项内容发生变化，交易id都会发生变化， 然后相应的对交易id的签名也就会改变，导致整个交易无效。

``` typescript
const signTxIn = (transaction: Transaction, txInIndex: number,
                  privateKey: string, aUnspentTxOuts: UnspentTxOut[]): string => {
    const txIn: TxIn = transaction.txIns[txInIndex];
    const dataToSign = transaction.id;
    const referencedUnspentTxOut: UnspentTxOut = findUnspentTxOut(txIn.txOutId, txIn.txOutIndex, aUnspentTxOuts);
    const referencedAddress = referencedUnspentTxOut.address;
    const key = ec.keyFromPrivate(privateKey, 'hex');
    const signature: string = toHexString(key.sign(dataToSign).toDER());
    return signature;
};
```
这里我们看下如果发生攻击时，交易签名是如何对我们的系统进行保护的：

- 攻击者节点收到一个交易广播：从AAA地址中发送10个币到BBB,交易id为0x555...
- 攻击者随后将接收者地址修改成CCC, 然后把这个交易继续发送到网络中。现在这个广播内容就会变成：从AAA地址中发送10个币到CCC
- 但是，因为交易数据中的接收者地址被修改了，对应的交易id就不再有效。新的交易id应该会改变，比如变成0x567...。 当其他用户收到该交易时，首先会验证交易id，立即就会发现这个数据被篡改了。
- 如果攻击者修改接收地址的同时，把交易id也修改了呢？因为AAA只是对交易原始id 0x555...进行签名，并没有用私钥对0x567...进行签名， 其他节点验证时即能识破
- 所以最终该篡改的交易都不会被其他节点接受，因为无论怎么修改，它都是无效的。

### 未消费的交易outputs
一笔交易中，发起者必须在input中指定还没有被消费的交易output。 你在区块链中拥有的加密货币， 指的其实就是在**未消费交易outputs**中，接受者地址为自己的公钥的一系列outputs。

当对交易进行有效性验证时，我们只需要关注**未消费交易outputs**这份清单。当然，这份独立的清单也可以从我们的区块链中演绎出来。这样子实现的话，当我们把交易纳入到区块链中时，我们将同时也会更新未消费交易outputs这份清单。

**未消费交易output**的数据结构大致如下所示:

``` typescript
class UnspentTxOut {
    public readonly txOutId: string;
    public readonly txOutIndex: number;
    public readonly address: string;
    public readonly amount: number;

    constructor(txOutId: string, txOutIndex: number, address: string, amount: number) {
        this.txOutId = txOutId;
        this.txOutIndex = txOutIndex;
        this.address = address;
        this.amount = amount;
    }
}
```

整一个清单其实就是一个数组:

``` typescript
let unspentTxOuts: UnspentTxOut[] = [];
```

### 未消费交易outputs清单更新
每当一个新的区块加入到区块链中，我们都必须对我们的未消费outputs进行更新。 因为新的区块将可能会消费掉「未消费outputs」中的一些outputs，并肯定会引入新的outputs。

为了对此进行处理，我们需要在新区块加入时，将区块的交易数据中的的未消费交易outputs给解析出来：

``` typescript
 const newUnspentTxOuts: UnspentTxOut[] = newTransactions
        .map((t) => {
            return t.txOuts.map((txOut, index) => new UnspentTxOut(t.id, index, txOut.address, txOut.amount));
        })
        .reduce((a, b) => a.concat(b), []);
```

同时，我们还要找出这个新增区块将会消耗掉哪些未交易outputs。我们通过检验交易数据的inputs下的项，即可以将这些数据找出来:

``` typescript
const consumedTxOuts: UnspentTxOut[] = newTransactions
        .map((t) => t.txIns)
        .reduce((a, b) => a.concat(b), [])
        .map((txIn) => new UnspentTxOut(txIn.txOutId, txIn.txOutIndex, '', 0));

```

最终我们通过删除已经消费的并且加上新的未消费的，从而产生了新的未消费交易outputs，具体代码如下：

``` typescript
const resultingUnspentTxOuts = aUnspentTxOuts
        .filter(((uTxO) => !findUnspentTxOut(uTxO.txOutId, uTxO.txOutIndex, consumedTxOuts)))
        .concat(newUnspentTxOuts);
```

以上代码片段都是在updateUnspentTxOuts这个方法中实现的。 需要注意的是，这个方法只有在区块含有的交易数据(以及这个块本身)都被验证没有问题的时候才会被调用。


### 交易有效性验证

现在我们可以制定一些规则来检查交易是否有效：

- 交易数据结构有效性验证
对数据结构的类型等进行验证:

``` typescript
const isValidTransactionStructure = (transaction: Transaction) => {
        if (typeof transaction.id !== 'string') {
            console.log('transactionId missing');
            return false;
        }
        ...
       //check also the other members of class
    }
```

- 交易id有效性验证

``` typescript
if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid tx id: ' + transaction.id);
        return false;
    }
```

- inputs有效性验证

交易数据结构中的inputs中的签名必须有效，且指向的交易来源outputs必须还没有被消费掉。

``` typescript
const validateTxIn = (txIn: TxIn, transaction: Transaction, aUnspentTxOuts: UnspentTxOut[]): boolean => {
    const referencedUTxOut: UnspentTxOut =
        aUnspentTxOuts.find((uTxO) => uTxO.txOutId === txIn.txOutId && uTxO.txOutId === txIn.txOutId);
    if (referencedUTxOut == null) {
        console.log('referenced txOut not found: ' + JSON.stringify(txIn));
        return false;
    }
    const address = referencedUTxOut.address;

    const key = ec.keyFromPublic(address, 'hex');
    return key.verify(transaction.id, txIn.signature);
};
```

- outputs有效性验证
交易数据结构中outputs所指定的交易总额之和，必须与inputs中指向的交易来源的outputs总额之和一致。比如，指向的交易来源output有50个币，然后你需要给别人发送30个币，最终outputs中将会有两条记录，一条是发送30个币给对方，另外一条是发送20个币给自己，总共加起来就是50个币(当然，最终在未消费交易outputs中，交易来源的这条output将会被删除掉，而新的两个outputs将会被加进去)。

``` typescript
const totalTxInValues: number = transaction.txIns
        .map((txIn) => getTxInAmount(txIn, aUnspentTxOuts))
        .reduce((a, b) => (a + b), 0);

    const totalTxOutValues: number = transaction.txOuts
        .map((txOut) => txOut.amount)
        .reduce((a, b) => (a + b), 0);

    if (totalTxOutValues !== totalTxInValues) {
        console.log('totalTxOutValues !== totalTxInValues in tx: ' + transaction.id);
        return false;
    }
```

### 原始交易

如前面谈及的，交易的inputs所指向的交易来源总是来自「未消费交易outputs」， 但是，最原始的一条交易记录的inputs将会没地方指向。因为这时候还根本没有任何未消费交易outputs。为了解决这个问题，我们需要引入一个特殊的交易类型：「原始交易」

原始交易的数据结构中只会有一个output，且input不会指向任何交易来源。也就是说这种交易只是为了增加新币进行流通用的，并不是一个用户和另外一个用户进行的交易。每一次挖矿成功，都会产生一个原始交易。

我们对原始交易中产生的货币量定义为50个币：

``` typescript
const COINBASE_AMOUNT: number = 50;
```

每个区块的第一个交易记录都是原始交易，且该第一个交易中的output中指向的接收者地址都是挖出该区块的矿工的公钥。所以，原始交易可以堪称是对挖矿的一种激励机制：一旦你挖出了一个区块，你就会获得50个币的激励。

同时，我们会将区块的高度信息(可以理解为区块的序号)加入到原始交易的input当中，这样做的目的是为了保证每笔原始交易id都是不一样的。因为交易id是通过对交易的内容做哈希算出来的，多条‘给地址0x5cc发放50个币‘记录将不至于会生成同一个交易id。

对原始交易的有效性验证将会和对普通交易的有效性验证有所不同:

``` typescript
const validateCoinbaseTx = (transaction: Transaction, blockIndex: number): boolean => {
    if (getTransactionId(transaction) !== transaction.id) {
        console.log('invalid coinbase tx id: ' + transaction.id);
        return false;
    }
    if (transaction.txIns.length !== 1) {
        console.log('one txIn must be specified in the coinbase transaction');
        return;
    }
    if (transaction.txIns[0].txOutIndex !== blockIndex) {
        console.log('the txIn index in coinbase tx must be the block height');
        return false;
    }
    if (transaction.txOuts.length !== 1) {
        console.log('invalid number of txOuts in coinbase transaction');
        return false;
    }
    if (transaction.txOuts[0].amount != COINBASE_AMOUNT) {
        console.log('invalid coinbase amount in coinbase transaction');
        return false;
    }
    return true;
};
```
### 测试体验

因为当前还没有引入钱包机制，手动测试将会非常困难，所以不建议现在进行测试体验。 在下一章节中实现了钱包之后，体验起来就会方便很多了。

### 小结

这个章节中我们将交易引入到我们的区块链中来。基本思路是很简单的：我们通过在交易数据结构中的inputs中指定「未消费outputs」来作为交易货币来源，并通过「未消费outputs」中的接收者地址来对本交易的签名进行验证，来证明该交易的发起者是该未消费outputs持有者，然后通过outputs中的接收者的地址来将该未消费outputs重新分配给指定的接收者，最终交易完成。

但是，到现在为止，去创建一笔交易还是相当的麻烦的。我们必须手动去创建交易的inputs和outputs，然后用我们的私钥去对交易进行加密签名。在我们下一章节中，我们将引入钱包机制，这些困局将会被一一击破。

本章节完整代码请查看[这里](https://github.com/zhubaitian/naivecoin/tree/chapter3)

[第四章](https://github.com/zhubaitian/naivecoin/blob/chapter4/README.md)





