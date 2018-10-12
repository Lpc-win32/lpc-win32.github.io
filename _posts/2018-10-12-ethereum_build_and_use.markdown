---
layout:     post
title:      "搭建最简单的以太坊环境 - 基本功能使用"
subtitle:   " \"初识go-ethereum,在深入学习源码之前,我们先学会怎么用\""
date:       2018-10-12 14:14
header-img: "img/post-bg-ethereum.jpg" 
author:     "pepperliu"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - ethereum
    - blockchain
---

> 在深入学习源码之前，我们先学会怎么用

下面我们使用geth命令，带领大家搭建一个基础联盟链环境

### 1. 创世区块

创建两个区块链目录user\_a与user\_b，并在路径下加入创世区块json文件，内容如下

```
{
    "config":   {
        "chainId":  15,
        "homesteadBlock":   0,
        "eip155Block":  0,
        "eip158Block":  0
    },
    "coinbase"  :   "0x0000000000000000000000000000000000000000",
    "difficulty"    :   "0x40000",
    "extraData" :   "",
    "gasLimit"  :   "0xffffffff",
    "nonce" :   "0x0000000000000042",
    "mixhash"   :   "0x0000000000000000000000000000000000000000000000000000000000000000",
    "parentHash"    :   "0x0000000000000000000000000000000000000000000000000000000000000000",
    "timestamp" :   "0x00",
    "alloc":    {   }
}
```

分别进入刚才创建的genesis\.json配置文件目录下，初始化创世区块。下面执行以下初始化操作用，涉及到操作参数为init

```
user_a$ ~/go/src/github.com/ethereum/go-ethereum/build/bin/geth --datadir ./data-init1/ init genesis.json
### 看到下面的输出就代表我们init成功了
INFO [10-12|10:51:43.849] Maximum peer count                       ETH=25 LES=0 total=25
INFO [10-12|10:51:43.857] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/geth/chaindata cache=16 handles=16
INFO [10-12|10:51:43.864] Writing custom genesis block 
INFO [10-12|10:51:43.865] Persisted trie from memory database      nodes=0 size=0.00B time=20.267µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [10-12|10:51:43.866] Successfully wrote genesis state         database=chaindata                                                                                         hash=a0e580…a5e82e
INFO [10-12|10:51:43.866] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/geth/lightchaindata cache=16 handles=16
INFO [10-12|10:51:43.868] Writing custom genesis block 
INFO [10-12|10:51:43.868] Persisted trie from memory database      nodes=0 size=0.00B time=2.983µs  gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [10-12|10:51:43.869] Successfully wrote genesis state         database=lightchaindata                                                                                         hash=a0e580…a5e82e

### 同样的方法，我们进入user_b目录init另一条链
user_b$ ~/go/src/github.com/ethereum/go-ethereum/build/bin/geth --datadir ./data-init2/ init genesis.json

INFO [10-12|10:52:51.380] Maximum peer count                       ETH=25 LES=0 total=25
INFO [10-12|10:52:51.388] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_b/data-init2/geth/chaindata cache=16 handles=16
INFO [10-12|10:52:51.391] Writing custom genesis block 
INFO [10-12|10:52:51.391] Persisted trie from memory database      nodes=0 size=0.00B time=14.416µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [10-12|10:52:51.392] Successfully wrote genesis state         database=chaindata                                                                                         hash=a0e580…a5e82e
INFO [10-12|10:52:51.392] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_b/data-init2/geth/lightchaindata cache=16 handles=16
INFO [10-12|10:52:51.393] Writing custom genesis block 
INFO [10-12|10:52:51.393] Persisted trie from memory database      nodes=0 size=0.00B time=3.049µs  gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [10-12|10:52:51.394] Successfully wrote genesis state         database=lightchaindata                                                                                         hash=a0e580…a5e82e
```

### 2. 启动控制台

打开两个窗口启动两个节点，这里有一点需要注意的是，虽然是两个节点，但他们的启动都是geth，只不过datadir目录不同而已。

```
我们先启动第一个节点，命令如下
user_a$ ~/go/src/github.com/ethereum/go-ethereum/build/bin/geth --datadir ./data-init1/ --networkid 88 --nodiscover console

INFO [10-12|11:14:11.818] Maximum peer count                       ETH=25 LES=0 total=25
INFO [10-12|11:14:11.830] Starting peer-to-peer node               instance=Geth/v1.8.16-unstable-1a16cc71/darwin-amd64/go1.11
INFO [10-12|11:14:11.831] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/geth/chaindata cache=768 handles=128
INFO [10-12|11:14:11.850] Initialised chain configuration          config="{ChainID: 15 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: <nil> EIP155: 0 EIP158: 0 Byzantium: <nil> Constantinople: <nil> Engine: unknown}"
INFO [10-12|11:14:11.851] Disk storage enabled for ethash caches   dir=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/geth/ethash count=3
INFO [10-12|11:14:11.851] Disk storage enabled for ethash DAGs     dir=/Users/liupengcheng/.ethash                                                                    count=2
INFO [10-12|11:14:11.852] Initialising Ethereum protocol           versions="[63 62]" network=88
INFO [10-12|11:14:11.853] Loaded most recent local header          number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:14:11.854] Loaded most recent local full block      number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:14:11.854] Loaded most recent local fast block      number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:14:11.855] Regenerated local transaction journal    transactions=0 accounts=0
INFO [10-12|11:14:11.857] Starting P2P networking 
INFO [10-12|11:14:11.861] RLPx listener up                         self="enode://e9111e71e9ad4fdead695dc1bc23bebdf1aea492bd2eeb986e4d9e355df5838479a075a050381fd7bf8b540fee9eb4b8e32d12c229002cd417f889069a08e7fc@[::]:30303?discport=0"
INFO [10-12|11:14:11.867] IPC endpoint opened                      url=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/geth.ipc
Welcome to the Geth JavaScript console!

instance: Geth/v1.8.16-unstable-1a16cc71/darwin-amd64/go1.11
 modules: admin:1.0 debug:1.0 eth:1.0 ethash:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> 
```

- networkid指定网络id，确保不使用1-4，1-4为系统内置使用。我们随便给定一个88端口
- nodiscover此参数确保geth不去寻找peers节点，主要是为了控制联盟链接入的节点

如果日志中显示“Welcome to the Geth JavaScript console!”,则说明启动成功了

> 注意：这里我们在启动第一个节点时并没有指定port参数，因此此处采用了默认的port，也就是30303。另一个节点在启动时需要避免端口重复使用

下面在另一个shell中启动另一个节点，换一个端口使用30306

```
user_b$ ~/go/src/github.com/ethereum/go-ethereum/build/bin/geth --datadir ./data-init2/ --port 30306 --networkid 88 --nodiscover console

INFO [10-12|11:21:10.711] Maximum peer count                       ETH=25 LES=0 total=25
INFO [10-12|11:21:10.720] Starting peer-to-peer node               instance=Geth/v1.8.16-unstable-1a16cc71/darwin-amd64/go1.11
INFO [10-12|11:21:10.720] Allocated cache and file handles         database=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_b/data-init2/geth/chaindata cache=768 handles=128
INFO [10-12|11:21:10.734] Initialised chain configuration          config="{ChainID: 15 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: <nil> EIP155: 0 EIP158: 0 Byzantium: <nil> Constantinople: <nil> Engine: unknown}"
INFO [10-12|11:21:10.734] Disk storage enabled for ethash caches   dir=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_b/data-init2/geth/ethash count=3
INFO [10-12|11:21:10.734] Disk storage enabled for ethash DAGs     dir=/Users/liupengcheng/.ethash                                                                    count=2
INFO [10-12|11:21:10.734] Initialising Ethereum protocol           versions="[63 62]" network=88
INFO [10-12|11:21:10.735] Loaded most recent local header          number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:21:10.735] Loaded most recent local full block      number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:21:10.735] Loaded most recent local fast block      number=0 hash=a0e580…a5e82e td=262144 age=49y5mo3w
INFO [10-12|11:21:10.735] Regenerated local transaction journal    transactions=0 accounts=0
INFO [10-12|11:21:10.736] Starting P2P networking 
INFO [10-12|11:21:10.736] RLPx listener up                         self="enode://84218da5fc039dd28b9cef9cf2e7ea09a84bd8a395c259a06aba5226239c70a40b684fabd548f619980d29f4397ac669d88462a64eadadf086c4926ab0673797@[::]:30306?discport=0"
INFO [10-12|11:21:10.739] IPC endpoint opened                      url=/Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_b/data-init2/geth.ipc
Welcome to the Geth JavaScript console!

instance: Geth/v1.8.16-unstable-1a16cc71/darwin-amd64/go1.11
 modules: admin:1.0 debug:1.0 eth:1.0 ethash:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

> 
```

### 3. 添加coinbase账户

coinbase：接收挖矿挖来的ETH的地址

现在创建一个密码为123456账户，操作如下：

```
> personal.listAccounts
[]
> personal.newAccount("123456")
"0x85bd26dccaa0a567342eec3dbd69effec67cd2bf"
> personal.listAccounts
["0x85bd26dccaa0a567342eec3dbd69effec67cd2bf"]
```

同样我们在另一个节点也创建一个密码为123456的账户

```
> personal.listAccounts
[]
> personal.newAccount("123456")
"0x39bd93de97ab89e18ea0cd86939705577959fadb"
> personal.listAccounts
["0x39bd93de97ab89e18ea0cd86939705577959fadb"]
```

这样我们两个节点都有了自己的账户。它们的keystore路径下对应生成了加密的私钥文件

由于只有一个账户，因此该地址就是coinbase地址  
如果一个节点存在多个账户，coinbase地址为第一个账户的地址

我们也可以通过下面的命令查看coinbase账户

```
> eth.coinbase
"0x85bd26dccaa0a567342eec3dbd69effec67cd2bf"
```

如果想查看更多信息可以执行下面的命令，不仅打印了账户信息，还打印出了私钥存储的位置和账户状态等信息

```
> personal.listWallets
[{
    accounts: [{
        address: "0x85bd26dccaa0a567342eec3dbd69effec67cd2bf",
        url: "keystore:///Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/keystore/UTC--2018-10-12T03-26-58.569544000Z--85bd26dccaa0a567342eec3dbd69effec67cd2bf"
    }],
    status: "Locked",
    url: "keystore:///Users/liupengcheng/Desktop/go-ethereum-workspace/alliance_chain/user_a/data-init1/keystore/UTC--2018-10-12T03-26-58.569544000Z--85bd26dccaa0a567342eec3dbd69effec67cd2bf"
}]
```

### 4. 联盟链互通

下面我们把两个节点连起来。首先查看一下peers情况

```
> admin.peers
[]
```

发现节点并没有链接上其他任何节点，这是nodiscover选项发挥的效果，下面通过分享enode的方式来让两个节点建立连接

下面我们使用节点a主动连接节点b

首先我们在b节点执行下面的命令，查询b节点的enode值

```
### b node
> admin.nodeInfo.enode
"enode://84218da5fc039dd28b9cef9cf2e7ea09a84bd8a395c259a06aba5226239c70a40b684fabd548f619980d29f4397ac669d88462a64eadadf086c4926ab0673797@[::]:30306?discport=0"
```

下面我们在a节点连接b节点

```
### a node
> admin.addPeer("enode://84218da5fc039dd28b9cef9cf2e7ea09a84bd8a395c259a06aba5226239c70a40b684fabd548f619980d29f4397ac669d88462a64eadadf086c4926ab0673797@[::]:30306?discport=0")
true
```

看到终端打印了true，说明我们成功建立了ab节点之间的互通

```
> admin.peers

[{
    caps: ["eth/63"],
    id: "84218da5fc039dd28b9cef9cf2e7ea09a84bd8a395c259a06aba5226239c70a40b684fabd548f619980d29f4397ac669d88462a64eadadf086c4926ab0673797",
    name: "Geth/v1.8.16-unstable-1a16cc71/darwin-amd64/go1.11",
    network: {
      inbound: false,
      localAddress: "[::1]:59759",
      remoteAddress: "[::1]:30306",
      static: true,
      trusted: false
    },
    protocols: {
      eth: {
        difficulty: 262144,
        head: "0xa0e580c6769ac3dd80894b2a256164a76b796839d2eb7f799ef6b9850ea5e82e",
        version: 63
      }
    }
}]
```

b节点也可以看到类似的peers信息

### 5. 查询余额

执行查看余额命令：

```
> eth.getBalance(eth.coinbase)
0
```

发现两个节点的余额都是0，这是因为我们还没有挖矿。下面我们开始挖矿

### 6. 挖矿

我们在a节点挖矿，b节点先不管

```
> miner.start()

INFO [10-12|12:11:13.951] Updated mining threads                   threads=4
INFO [10-12|12:11:13.951] Transaction pool price threshold updated price=1000000000
null
> INFO [10-12|12:11:13.952] Commit new mining work                   number=1 sealhash=e7725f…f3ffef uncles=0 txs=0 gas=0 fees=0 elapsed=273.714µs
INFO [10-12|12:11:15.727] Successfully sealed new block            number=1 sealhash=e7725f…f3ffef hash=45e7e2…efc52e elapsed=1.774s
INFO [10-12|12:11:15.727] 🔨 mined potential block                  number=1 hash=45e7e2…efc52e
INFO [10-12|12:11:15.728] Commit new mining work                   number=2 sealhash=06866c…fefadc uncles=0 txs=0 gas=0 fees=0 elapsed=722.878µs
INFO [10-12|12:11:18.325] Generating DAG in progress               epoch=1 percentage=0 elapsed=3.204s
INFO [10-12|12:11:21.014] Successfully sealed new block            number=2 sealhash=06866c…fefadc hash=d35585…79c22c elapsed=5.285s
INFO [10-12|12:11:21.014] 🔨 mined potential block                  number=2 hash=d35585…79c22c
INFO [10-12|12:11:21.014] Commit new mining work                   number=3 sealhash=418b97…1745ff uncles=0 txs=0 gas=0 fees=0 elapsed=201.063µs
INFO [10-12|12:11:21.382] Generating DAG in progress               epoch=1 percentage=1 elapsed=6.261s
INFO [10-12|12:11:21.387] Successfully sealed new block            number=3 sealhash=418b97…1745ff hash=941ef6…802bf5 elapsed=372.347ms
INFO [10-12|12:11:21.387] 🔨 mined potential block                  number=3 hash=941ef6…802bf5
INFO [10-12|12:11:21.387] Commit new mining work                   number=4 sealhash=6b261d…fc0667 uncles=0 txs=0 gas=0 fees=0 elapsed=134.488µs
INFO [10-12|12:11:24.376] Generating DAG in progress               epoch=1 percentage=2 elapsed=9.255s
INFO [10-12|12:11:24.464] Successfully sealed new block            number=4 sealhash=6b261d…fc0667 hash=708ad9…e18469 elapsed=3.077s
INFO [10-12|12:11:24.464] 🔨 mined potential block                  number=4 hash=708ad9…e18469
INFO [10-12|12:11:24.465] Commit new mining work                   number=5 sealhash=b7349e…5e2ba1 uncles=0 txs=0 gas=0 fees=0 elapsed=296.366µs
INFO [10-12|12:11:27.447] Generating DAG in progress               epoch=1 percentage=3 elapsed=12.326s
INFO [10-12|12:11:30.240] Generating DAG in progress               epoch=1 percentage=4 elapsed=15.119s
INFO [10-12|12:11:31.842] Successfully sealed new block            number=5 sealhash=b7349e…5e2ba1 hash=0f8a23…e72331 elapsed=7.376s

> miner.stop()
```

现在我们来看看钱包里的钱

```
> eth.getBalance(eth.coinbase)
30000000000000000000
```

### 7. 交易转账

现在我们从节点a的coinbase账户转账一笔交易给节点b的coinbase账户。在转账交易前我们需要unlock两个节点上的coinbase账户，解锁的密码就是之前创建account的密码123456

```
> personal.unlockAccount(eth.coinbase)
Unlock account 0x85bd26dccaa0a567342eec3dbd69effec67cd2bf
Passphrase: 
true
```

在节点a上执行转账操作

```
> eth.sendTransaction({from: eth.coinbase, to: '0x39bd93de97ab89e18ea0cd86939705577959fadb', value: 1000000})
INFO [10-12|13:16:13.833] Setting new local account                address=0x85BD26DccAA0a567342EEC3dBd69Effec67cD2bF
INFO [10-12|13:16:13.834] Submitted transaction                    fullhash=0x6350ee52e4b54727ab8992acf4cf3b2647a041de5b50abc5f237368f72916831 recipient=0x39bd93dE97aB89E18Ea0cD86939705577959fAdb
"0x6350ee52e4b54727ab8992acf4cf3b2647a041de5b50abc5f237368f72916831"
```

现在我们来看看b节点上是否收到转账交易

```
> eth.getBalance(eth.coinbase)
0
```

我们发现b的coinbase账户余额依然是0，为什么呢？  
因为我们虽然发起了交易，但并没有旷工挖矿打包交易。再次执行miner.start()。再次查询b节点coinbase账户余额

```
> eth.getBalance(eth.coinbase)
1000000
```

通过我们上述的操作，我们已经完成了一个拥有两个节点的联盟链

### 8. 解锁

```
> personal.unlockAccount(eth.accounts[0])
```

### 9. 交易

```
> eth.sendTransaction({from: eth.coinbase, to: '0x39bd93de97ab89e18ea0cd86939705577959fadb', value: 1000000})
```

### 10. 写一个简单的智能合约

[remix-ethereum-ide，用来写合约并得到ABI接口和合约二进制代码](http://remix.ethereum.org/)

合约代码如下：

```
contract Sample {
uint public value;

    function Sample(uint v) {
        value = v;
    }
    
    function set(uint v){
        value=v;
    }
    
    function get() constant returns (uint) {
        return value;
    }
}
```

abi如下：

```
[
{
    "constant": true,
        "inputs":   [],
        "name": "value",
        "outputs":  [
        {
            "name": "",
            "type": "uint256"
        }
        ],
        "payable":  false,
        "type": "function",
        "stateMutability":  "view"
},
{
    "constant": false,
    "inputs":   [
    {
        "name": "v",
        "type": "uint256"
    }
    ],
    "name": "set",
    "outputs":  [],
    "payable":  false,
    "type": "function",
    "stateMutability":  "nonpayable"
        ,
    {
        "constant": true,
        "inputs":   [],
        "name": "get",
        "outputs":  [
        {
            "name": "",
            "type": "uint256"
        }
        ],
        "payable":  false,
        "type": "function",
        "stateMutability":  "view"
    },
    {
        "inputs":   [
        {
            "name": "v",
            "type": "uint256"
        }
        ],
        "payable":  false,
        "type": "constructor",
        "stateMutability":  "nonpayable"
    }
}
]
```

对这段数据进行json数据压缩

```
[{"constant":true,"inputs":[],"name":"value","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function","stateMutability":"view"},{"constant":false,"inputs":[{"name":"v","type":"uint256"}],"name":"set","outputs":[],"payable":false,"type":"function","stateMutability":"nonpayable"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function","stateMutability":"view"},{"inputs":[{"name":"v","type":"uint256"}],"payable":false,"type":"constructor","stateMutability":"nonpayable"}]
```

在第一个节点输入：

```
> abi=[{"constant":true,"inputs":[],"name":"value","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function","stateMutability":"view"},{"constant":false,"inputs":[{"name":"v","type":"uint256"}],"name":"set","outputs":[],"payable":false,"type":"function","stateMutability":"nonpayable"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"","type":"uint256"}],"payable":false,"type":"function","stateMutability":"view"},{"inputs":[{"name":"v","type":"uint256"}],"payable":false,"type":"constructor","stateMutability":"nonpayable"}]
[{
    constant: true,
    inputs: [],
    name: "value",
    outputs: [{
        name: "",
        type: "uint256"
    }],
    payable: false,
    stateMutability: "view",
    type: "function"
}, {
    constant: false,
    inputs: [{
        name: "v",
        type: "uint256"
    }],
    name: "set",
    outputs: [],
    payable: false,
    stateMutability: "nonpayable",
    type: "function"
}, {
    constant: true,
    inputs: [],
    name: "get",
    outputs: [{
        name: "",
        type: "uint256"
    }],
    payable: false,
    stateMutability: "view",
    type: "function"
}, {
    inputs: [{
        name: "v",
        type: "uint256"
    }],
    payable: false,
    stateMutability: "nonpayable",
    type: "constructor"
}]
```

然后再输入：

```
> sample = eth.contract(abi)
{
  abi: [{
      constant: true,
      inputs: [],
      name: "value",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      constant: false,
      inputs: [{...}],
      name: "set",
      outputs: [],
      payable: false,
      stateMutability: "nonpayable",
      type: "function"
  }, {
      constant: true,
      inputs: [],
      name: "get",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      inputs: [{...}],
      payable: false,
      stateMutability: "nonpayable",
      type: "constructor"
  }],
  eth: {
    accounts: ["0x85bd26dccaa0a567342eec3dbd69effec67cd2bf"],
    blockNumber: 8,
    coinbase: "0x85bd26dccaa0a567342eec3dbd69effec67cd2bf",
    compile: {
      lll: function(),
      serpent: function(),
      solidity: function()
    },
    defaultAccount: undefined,
    defaultBlock: "latest",
    gasPrice: 1000000000,
    hashrate: 0,
    mining: false,
    pendingTransactions: [],
    protocolVersion: "0x3f",
    syncing: false,
    call: function(),
    contract: function(abi),
    estimateGas: function(),
    filter: function(options, callback, filterCreationErrorCallback),
    getAccounts: function(callback),
    getBalance: function(),
    getBlock: function(),
    getBlockNumber: function(callback),
    getBlockTransactionCount: function(),
    getBlockUncleCount: function(),
    getCode: function(),
    getCoinbase: function(callback),
    getCompilers: function(),
    getGasPrice: function(callback),
    getHashrate: function(callback),
    getMining: function(callback),
    getPendingTransactions: function(callback),
    getProtocolVersion: function(callback),
    getRawTransaction: function(),
    getRawTransactionFromBlock: function(),
    getStorageAt: function(),
    getSyncing: function(callback),
    getTransaction: function(),
    getTransactionCount: function(),
    getTransactionFromBlock: function(),
    getTransactionReceipt: function(),
    getUncle: function(),
    getWork: function(),
    iban: function(iban),
    icapNamereg: function(),
    isSyncing: function(callback),
    namereg: function(),
    resend: function(),
    sendIBANTransaction: function(),
    sendRawTransaction: function(),
    sendTransaction: function(),
    sign: function(),
    signTransaction: function(),
    submitTransaction: function(),
    submitWork: function()
  },
  at: function(address, callback),
  getData: function(),
  new: function()
}
```

然后输入合约二进制代码：

```
> SampleHEX="0x6060604052341561000c57fe5b60405160208061013a833981016040528080519060200190919050505b806000819055505b505b60f9806100416000396000f30060606040526000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff1680633fa4f24514604e57806360fe47b11460715780636d4ce63c14608e575bfe5b3415605557fe5b605b60b1565b6040518082815260200191505060405180910390f35b3415607857fe5b608c600480803590602001909190505060b7565b005b3415609557fe5b609b60c2565b6040518082815260200191505060405180910390f35b60005481565b806000819055505b50565b600060005490505b905600a165627a7a723058205628425e21eefb62faf95e98fbde766001427f1fc9a6ad9856c3ed2ae336a5430029"
```

把合约代码部署上链条

```
> thesample=sample.new(1, {from:eth.coinbase, data:SampleHEX, gas:3000000})

{
  abi: [{
      constant: true,
      inputs: [],
      name: "value",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      constant: false,
      inputs: [{...}],
      name: "set",
      outputs: [],
      payable: false,
      stateMutability: "nonpayable",
      type: "function"
  }, {
      constant: true,
      inputs: [],
      name: "get",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      inputs: [{...}],
      payable: false,
      stateMutability: "nonpayable",
      type: "constructor"
  }],
  address: undefined,
  transactionHash: "0xa04b0c5ec43c4467daef3735f043fa0c58ca32879a11c8debdd4c1c788abf5f1"
}
```

挖一会儿矿，看看交易细节

```
> samplerecept=eth.getTransactionReceipt("0xa04b0c5ec43c4467daef3735f043fa0c58ca32879a11c8debdd4c1c788abf5f1")
{
  blockHash: "0xf59d918cfe2d518a52aa71fefd24c2df0af8843b13d56028616bc3df5a3725b8",
  blockNumber: 9,
  contractAddress: "0xfd0a74a25a25eb8a804a81b53d1b7d016f7b03e6",
  cumulativeGasUsed: 141836,
  from: "0x85bd26dccaa0a567342eec3dbd69effec67cd2bf",
  gasUsed: 141836,
  logs: [],
  logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  root: "0xdc948a431de412e8f843f743accdbc2a36a9dfaf2df5e6d2b0c44af346913ddc",
  to: null,
  transactionHash: "0xa04b0c5ec43c4467daef3735f043fa0c58ca32879a11c8debdd4c1c788abf5f1",
  transactionIndex: 0
}
```

看看合约命名：

```
> samplecontract=sample.at("0xfd0a74a25a25eb8a804a81b53d1b7d016f7b03e6")

{
  abi: [{
      constant: true,
      inputs: [],
      name: "value",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      constant: false,
      inputs: [{...}],
      name: "set",
      outputs: [],
      payable: false,
      stateMutability: "nonpayable",
      type: "function"
  }, {
      constant: true,
      inputs: [],
      name: "get",
      outputs: [{...}],
      payable: false,
      stateMutability: "view",
      type: "function"
  }, {
      inputs: [{...}],
      payable: false,
      stateMutability: "nonpayable",
      type: "constructor"
  }],
  address: "0xfd0a74a25a25eb8a804a81b53d1b7d016f7b03e6",
  transactionHash: null,
  allEvents: function(),
  get: function(),
  set: function(),
  value: function()
}
```

调用合约接口

```
> samplecontract.get.call()
1
> samplecontract.set.sendTransaction(10, {from:eth.coinbase, gas:3000000})
"0xf3158f4e7a5b847a171d4d84e7275d1327e7e4449037616b857d2402dc02a1cb"
```

执行挖矿，打包这组交易。再来看看我们是否set成功

```
> samplecontract.get.call()
10
```