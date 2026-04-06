---
tags:
  - web3
  - Eth
  - Sol
  - 链上转账
  - React
  - DApp
  - Transfer
---
这是一个典型且结构清晰的 **Ethereum + Solana 双链 Web3 基础转账 DApp**。它使用 React (基于 Vite 构建) 作为前端框架，无缝整合了 EVM (以太坊生态) 和 Solana 两个生态的核心 Web3 组件。
此项目不仅是一个实用的多链扫码/插件转账工具，同时也是一个极其标准的 Web3 DApp 架构参考示例。

---
## 🛠 技术栈概览

- **前端架构：** React 18, Vite
- **EVM (以太坊) 生态：** `Wagmi` (Hooks 管理), `Viem` (底层以太坊交互接口), `@tanstack/react-query` (状态缓存)
- **Solana 生态：** `@solana/wallet-adapter-react` (钱包适配器), `@solana/web3.js` (底层 RPC 交互)

---
## 🏗️ 1. 项目架构

整个项目架构采用高内聚、低耦合的设计，可分为三大层：

1. **表现层 (UI)**：由 `TransferPage.jsx` 和 `WalletConnector.jsx` 构成，负责渲染转账表单、钱包连接按钮组以及状态反馈界面。
2. **逻辑抽象层 (Hooks)**：`useTransfer.*` 和 `useTransferForm.js`。这一层将复杂的 Web3 交互（如构建交易 Payload、获取签名、验证 Base58/Hex 地址）封装起来，向 UI 层暴露出最原语化的 `send()`、`loading`、`isValid` 状态。
3. **基础设施层 (Providers & Config)**：由 `Web3Providers.jsx` 和 `wagmiConfig.js` 组成。负责给整个 React 组件树包裹生态 Context，注入并维护网络连接、RPC 节点和各路钱包的适配器。

---
## 📂 2. 核心文件指引

### 🔌 底层配置与 Provider (`src/providers` & `src/config`)

- **`Web3Providers.jsx`**: 全局 Provider 组合器。包含了 **ETH 层** (`QueryClientProvider`, `WagmiProvider`) 和 **SOL 层** (`ConnectionProvider`, `WalletProvider`, `WalletModalProvider`)。
- **`wagmiConfig.js`**: Wagmi 配置文件。定义支持的网络（Mainnet, Sepolia），连接器（MetaMask Injected, WalletConnect），及 RPC 传输层。
### 🎨 UI 组件 (`src/components`)
- **`TransferPage.jsx`**: 核心主页面。聚合了链切换、`WalletConnector` 挂载点以及包含完备防抖与反馈机制的转账表单。
- **`WalletConnector.jsx`**: 动态展示多链钱包连接。根据当前选中的链，分别渲染 Wagmi 提供的连接组件或 Solana 的 `WalletMultiButton`。
### ⚙️ 核心业务逻辑 (`src/hooks`)
- **`useTransferForm.js`**: 纯前端表单验证逻辑。对格式(如以太坊 0x 校验和、Base58 公钥)进行防错拦截。
- **`useTransfer.js`**: Facade 路由器 Hook。决定跨链策略向以太坊还是 Solana 转发。
- **`useEthTransfer.js`**: 以太坊转账细节逻辑。将输入转为 `Wei`，向以太坊网络发起交易并等待 `Receipt`。
- **`useSolTransfer.js`**: Solana 转账细节逻辑。转成 `Lamports`，实例化 `Transaction`，拉取最新的 Blockhash 并广播签名的包裹。

---
## 🔄 3. 详细实现流程与核心代码解析

当用户在界面上选中 `SOL` 或 `ETH` 链，填写收钱包地址和金额并点击 **“Send”** 的一瞬间，整个应用的执行流水线如下：
### 阶段 1：表单初始化与实时拦截验证 (Validation)
用户输入内容时，`useTransferForm` 会实时运行验证逻辑，确保格式不对或数额不合规的请求根本无法触达链上。

```javascript

// 核心代码片段: useTransferForm.js

const addressError = useMemo(() => {

if (!toAddress) return null

if (chain === 'ETH') {

return isAddress(toAddress) ? null : '以太坊地址格式无效'

}

if (chain === 'SOL') {

try {

new PublicKey(toAddress) // 尝试基于 Base58 解析公钥

return null

} catch {

return 'Solana 地址格式无效'

}

}

return null

}, [toAddress, chain])

```

### 阶段 2：提交表单并利用 Facade 分发请求 (Dispatch)

当用户点击发送按钮时，前端屏蔽浏览器默认行为，并通过入口钩子 `useTransfer.js` 动态分发任务。

```javascript

// 核心代码片段: useTransfer.js

export function useTransfer(chain) {

const eth = useEthTransfer()

const sol = useSolTransfer()

  

// 门面模式：动态返回底层真实的交易逻辑和状态

if (chain === 'SOL') {

return { send: sol.sendSol, loading: sol.loading, /* ... */ }

}

return { send: eth.sendEth, loading: eth.loading, /* ... */ }

}

```

### 阶段 3-A：底层交易生命周期 (基于 Solana 原生转账)

如果此时位于 Solana 环境，请求会流转到 `useSolTransfer.js`，进入复杂的区块构建和签名流程。

```javascript

// 核心代码片段: useSolTransfer.js

// 1. 精度换算：转化为 Lamports (1 SOL = 10^9 Lamports)

const lamports = Math.round(parsed * LAMPORTS_PER_SOL)

  

// 2. 调用 Solana 系统程序，构建转账操作的 Instruction 指令

const instruction = SystemProgram.transfer({

fromPubkey: publicKey,

toPubkey,

lamports,

})

  

// 3. 将指令封存在 Transaction 中

const transaction = new Transaction().add(instruction)

  

// 4. 获取最新的区块哈希，防止交易过期

const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash()

transaction.recentBlockhash = blockhash

transaction.feePayer = publicKey

  

// 5. 唤起钱包(如 Phantom)要求用户签名，并自动将签名后的交易广播去 RPC 节点

const signature = await sendTransaction(transaction, connection)

  

// 6. 阻断式等待区块链最终确认这笔打包交易

await connection.confirmTransaction({ signature, blockhash, lastValidBlockHeight })

```

### 阶段 3-B：底层交易生命周期 (基于以太坊 EVM 转账)

如果位于 ETH 环境，借助于 viem 强大的封装能力，代码将极其简洁。

```javascript

// 核心代码片段: useEthTransfer.js

// 1. 将常规浮点数转为 BigInt 的 Wei 值

const value = parseEther(String(amount))

  

// 2. 向前置钱包 (Metamask等) 发起支付授权及广播

const hash = await walletClient.sendTransaction({

to: toAddress,

value,

})

setTxHash(hash)

  

// 3. 异步等待区块确认回执

await publicClient.waitForTransactionReceipt({ hash })

```

### 阶段 4：结果反馈与清理 (UI Feedback)
交易确认后，底层 Hook 退出 `loading` 态并向外释放 `txHash`，触发主页面 React 重渲染。渲染时拦截状态，给用户呈现带有区块浏览器直达链接的绿色成功反馈画面。

```javascript

// 核心代码片段: TransferPage.jsx

{txHash && !loading && (

<div style={styles.successBox}>

<p style={styles.successTitle}>✅ 恭喜，交易已成！</p>

<a

href={EXPLORER_URL[chain](txHash)} // 动态读取预设好的区块浏览器 URL 格式

target="_blank"

style={styles.txLink}

>

点击通过区块链浏览器查阅详情 ↗

</a>

</div>

)}

```
