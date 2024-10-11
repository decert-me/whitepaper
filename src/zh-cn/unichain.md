# Unichain 白皮书

2024 年 10 月

- Hayden Adams [hayden@uniswap.org](hayden@uniswap.org)
- Mark Toda [mark@uniswap.org](mark@uniswap.org)
- Alex Karys [alex.karys@uniswap.org](alex.karys@uniswap.org)
- Xin Wan [xin@uniswap.org](xin@uniswap.org)
- Daniel Gretzke [daniel.gretzke@uniswap.org](daniel.gretzke@uniswap.org)
- Eric Zhong [eric.zhong@uniswap.org](eric.zhong@uniswap.org)
- Zach Wong [zach.wong@uniswap.org](zach.wong@uniswap.org)
- Daniel Marzec [dan@flashbots.net](dan@flashbots.net)
- Robert Miller [robert@flashbots.net](robert@flashbots.net)
- Hasu [hasu@flashbots.net](hasu@flashbots.net)
- Karl Floersch [karl@oplabs.co](karl@oplabs.co)
- Dan Robinson [dan@paradigm.xyz](dan@paradigm.xyz)

## 摘要

Unichain 是一个乐观的 Rollup，旨在通过提供快速状态更新、为应用程序提供内化 MEV 的框架以及提供快速跨区块链结算的经济最终性系统来优化高效市场。

## 1. 介绍

以太坊的 Rollup 中心化路线图通过 Rollup 的普及成功地扩展了链上活动。然而，这种方法为 DeFi 生态系统引入了新的挑战，包括执行质量不佳、用户体验下降和流动性碎片化。

本文介绍了 Unichain，这是一个基于 OP Stack [16] 构建的乐观 Rollup，旨在通过两个关键创新来解决这些核心挑战：
- **可验证的区块构建**：与 Flashbots 合作构建的区块构建机制，最初设计为提供：

    - 通过将每个区块分成四个“Flashblocks”来实现 200-250ms 的有效区块时间。
    - 在每个 Flashblock 内透明地执行优先级排序，允许应用程序为其用户分配部分最大可提取价值（MEV）。
    - 对交易的无信任回滚保护。

- **Unichain 验证网络**：一个去中心化的 Unichain 节点运营商网络，旨在减少区块排序过程中的某些关键风险，实现跨链交易的更快经济最终性，并支持潜在的未来扩展。

Unichain 构建在 Superchain 上，这是一个基于 OP Stack 的可扩展和互联的 Rollup 网络，作为促进流动性无缝移动的基础环境。与基于意图的跨链桥 [21] 和来自 Unichain 验证网络的快速最终性一起，Superchain 的连接旨在为 Rollup 用户提供快速、廉价和广泛的流动性访问。Unichain 正在通过迭代的开源过程进行开发，其代码库可供其他 OP Stack Rollup 使用。本文档中描述的功能将在名为 Unichain Experimental 的公开测试网上进行全面测试，然后部署到 Unichain 主网上。

## 2. 先前工作和当前挑战

以太坊在构建一个强大、无许可和可信中立的网络方面取得了巨大进展。然而，随着区块链技术的采用增加，各种问题也随之出现。其中最紧迫的问题之一是在网络拥堵期间 gas 费用的不可预测性和高成本 [1, 2]，这是由于吞吐量的限制造成的。Rollup 被提议作为扩展底层网络整体处理能力的策略，并被社区建议作为主要策略 [6]。目前，最受欢迎的 Rollup 的底层技术由 OP Labs [16, 17] 和 Offchain Labs [12] 等开发。大多数这些 Rollup 需要信任排序器会遵守其声明的区块构建协议。

此外，允许用户从其他用户中提取最大可提取价值（“MEV” [8]）的区块生产环境限制了可以构建的市场和应用程序的效率。诸如私有内存池之类的解决方案被引入作为缓解 MEV 提取风险的措施，但它们引入了单点故障 [2, 9, 15, 18]，对此，可信执行环境（TEEs）被提议作为解决方案 [10]。

由 Uniswap Labs 开创的自动化做市商（AMMs）[3–5] 在以太坊等区块链上也取得了显著增长 [22]。虽然它们通过使流动性提供无许可化革新了数字资产交易，但其他挑战也随之出现。具体而言，现有区块链的限制（如长区块时间）增加了链上流动性提供者的不利选择风险 [13, 14]，而公共内存池和不受约束的交易排序可能导致三明治攻击 [19]。

为了解决当前的一系列挑战，Unichain 引入了两个主要功能：可验证的区块构建和 Unichain 验证网络。这些功能从上述研究和创新中汲取灵感。

## 3. 可验证的区块构建

区块构建在确定 MEV 泄漏和延迟特性方面起着至关重要的作用。Unichain 采用了一种新颖的区块构建协议，优化了用户体验和价值保留，同时保持中立性。这是通过与 Flashbots 合作开发的 Rollup-Boost [11] 实现的。

### 3.1 排序器构建者分离

Unichain 通过与 Flashbots 合作构建的可验证区块构建者将区块构建的角色与排序器分离。区块构建操作在可信执行环境（TEE）中执行，允许外部用户验证是否符合声明的排序规则。相对于它们所替代的服务器，TEEs 提供了增强的信任和安全保证。

初始 TEE 构建者将在 Intel 的 TDX 硬件上运行一个开源构建者代码库，该硬件通过其计算完整性属性 [7] 提供私有数据访问和可验证执行。执行证明将公开发布，允许用户验证区块是否根据声明的策略在 TEE 内构建。

TEE 区块构建是 Rollup 的一个强大原语，不仅可以缓解任意区块排序的风险，还提供了一个框架来进行透明的增量改进。

### 3.2 Flashblocks

Flashblocks 是由 TEE 区块构建者发出的区块预确认。较短的区块时间降低了流动性提供者的不利选择成本 [13, 14]，减少了用户的延迟，并促进了更高效的链上市场。当交易流向 TEE 构建者时，它会逐步承诺 Flashblocks，这些是将包含在最终提议区块中的有序交易集。然后，排序器将这些 Flashblocks 作为待定区块广播，提供给用户、应用程序和集成商的体验是区块时间比默认快多倍。在大多数当前的 Rollup 架构中，由于序列化和状态根生成，区块提案面临高固定延迟，使得亚秒级区块时间不可行。Flashblocks 在短时间范围内绕过了这种开销，实现了低延迟的区块链交互。

TEE 强制执行每个 Flashblock 的优先级排序，并支持一种 Flashblock 捆绑类型，使用户能够针对特定的 Flashblocks 进行包含。这两个功能的结合通过应用程序（例如通过 MEV 税 [20]）为用户分配 MEV 提供了便利。

### 3.3 无信任回滚保护

可验证的构建者实现了无信任回滚保护，降低了用户为失败交易付费的风险。TEE 在构建区块时模拟交易，并被编程为检测和删除任何回滚交易。

回滚保护减少了用户的摩擦，提高了 AMMs 和基于意图的系统的效率，因为参与者可以对其交易有更大的信心。

### 3.4 未来工作

可验证的区块构建者是 Unichain 上许多未来改进的基础，例如：
- 加密内存池：用户可以加密交易，增强其交易前隐私。
- 预定的交易：TEE 可以被编程为允许用户或智能合约提交自动交易以执行计划或重复操作。
- TEE 协处理器：TEE 可以允许智能合约请求私有、可验证的计算。

## 4. UNICHAIN 验证网络

Unichain 通过引入 Unichain 验证网络 (UVN) 来解决与单一排序器架构相关的风险，这是一个去中心化的节点运营商网络，独立验证最新的区块链状态。虽然 Rollup 受益于基础区块链的强大安全性，但排序器的行为可能会影响区块链的活跃性、MEV 动态和最终性。UVN 是一个可扩展的平台，最初专注于验证区块以加快最终性。

在单一排序器 Rollup 中，特别出现了两个主要风险，影响跨链结算的速度：
- **区块对等风险**：排序器在同一高度提出多个冲突区块的可能性，导致最终哪个区块会被确认的不确定性。
- **无效区块风险**：排序器发布无效区块的风险，当故障证明被提交时导致链回滚，进一步延迟结算。

这些风险导致区块链最终性等待时间更长，阻碍了网络间流动性的无缝流动。UVN 通过在区块被提议时让验证者证明规范链，提供更快的经济最终性来解决这些挑战。

### 4.1 抵押验证者

为了成为 UVN 的验证者，节点运营商必须在以太坊主网上抵押 UNI。抵押通过 Unichain 上的智能合约进行跟踪，该合约通过原生桥接接收抵押和取消抵押操作的通知。Unichain 区块被分割成固定长度的纪元。在每个纪元开始时，当前的抵押余额会被快照，区块链费用会被收集，并计算每个抵押代币的奖励值。参与者还可以抵押并为验证者投票，增加验证者的抵押权重。有限数量的验证者将根据最高的 UNI 抵押权重被视为活跃集，并有资格发布证明并获得纪元的指定补偿。

活跃验证者需要在线运行一个经过仪器化的 Reth Unichain 节点，执行提议区块的验证。验证者签署区块哈希并在每个纪元将其发布到 Unichain 上的 UVN 服务智能合约，作为对网络有效性的公开证明。服务智能合约在证明发布时验证这些证明，并根据验证者的抵押权重立即补偿验证者。未能为纪元发布有效证明的验证者将无法获得其分配，该补偿将滚入下一个纪元。

每个证明中执行的具体检查是可扩展的。首先，UVN 验证者将执行简单的区块证明，以增加对规范链状态的信心。

## 5. 潜在的未来工作

Unichain 验证者网络旨在成为一个平台，可以对排序过程进行各种检查。系统的一些未来扩展可能包括：
- **可信中立性**：验证者可以监控 Rollup 的内存池，确保交易被及时包含。
- **限制发布**：BatchPoster 合约可以要求在包含区块之前达到一定的证明权重，从而有效地限制排序器发布不遵循特定规则的区块的能力。

## 6. 结论

Unichain 解决了以太坊 Rollup 中心扩展策略中一些最紧迫的交易挑战，特别是流动性碎片化和低效的跨链交互。通过引入像 Flashblocks、Unichain 验证网络这样的创新技术，并与 Superchain 集成，Unichain 旨在成为 DeFi 流动性的家园和跨 Rollup 访问 DeFi 的最佳场所。

## 参考文献

- [1] Austin Adams. 2024. Layer 2 be or Layer not 2 be: Scaling on Uniswap v3. arXiv preprint arXiv:2403.09494 (2024).
- [2] Austin Adams, Benjamin Y Chan, Sarit Markovich, and Xin Wan. 2023. Don’t Let MEV Slip: The Costs of Swapping on the Uniswap Protocol. arXiv preprint arXiv:2309.13648 (2023).
- [3] Hayden Adams. 2018. Uniswap v1 Core. 2023 年 6 月 12 日检索自 https://hackmd.io/@HaydenAdams/HJ9jLsfTz
- [4] Hayden Adams, Noah Zinsmeister, 和 Dan Robinson. 2020. Uniswap v2 Core. 2023 年 6 月 12 日检索自 https://uniswap.org/whitepaper.pdf
- [5] Hayden Adams, Noah Zinsmeister, Moody Salem, River Keefer, 和 Dan Robinson. 2021. Uniswap v3 Core. 2023 年 6 月 12 日检索自 https://uniswap.org/whitepaper-v3.pdf
- [6] Vitalik Buterin. 2020. A rollup-centric ethereum roadmap. https://ethereummagicians.org/t/a-rollup-centric-ethereum-roadmap/4698
- [7] Victor Costan 和 Srinivas Devadas. 2016. Intel SGX Explained. Cryptology ePrint Archive, Paper 2016/086. https://eprint.iacr.org/2016/086
- [8] Philip Daian, Steven Goldfeder, Tyler Kell, Yunqi Li, Xueyuan Zhao, Iddo Bentov, Lorenz Breidenbach, 和 Ari Juels. 2020. Flash boys 2.0: Frontrunning in decentralized exchanges, miner extractable value, and consensus instability. 在 2020 IEEE Symposium on Security and Privacy (SP). IEEE, 910–927.
- [9] Flashbots. [n. d.]. MEV Protection Overview. https://docs.flashbots.net/flashbotsprotect/overview
- [10] Flashbots. 2022. The Future of MEV is SUAVE. https://writings.flashbots.net/thefuture-of-mev-is-suave
- [11] Flashbots. 2024. Introducing Rollup Boost. https://writings.flashbots.net/introducing-rollup-boost
- [12] Harry Kalodner, Steven Goldfeder, Xiaoqi Chen, S Matthew Weinberg, 和 Edward W Felten. 2018. Arbitrum: Scalable, private smart contracts. 在 27th USENIX Security Symposium (USENIX Security 18). 1353–1370.
- [13] Jason Milionis, Ciamac C Moallemi, and Tim Roughgarden. 2023. Automated market making and arbitrage profits in the presence of fees. arXiv preprint arXiv:2305.14604 (2023).
- [14] Jason Milionis, Ciamac C Moallemi, Tim Roughgarden, and Anthony Lee Zhang.2022. Automated market making and loss-versus-rebalancing. arXiv preprint arXiv:2208.06046 (2022).
- [15] Robert Miller. 2023. MEV-Share: programmably private orderflow to share MEV with users. https://collective.flashbots.net/t/mev-share-programmably-privateorderflow-to-share-mev-with-users/1264
- [16] Optimism. [n. d.]. Optimism Docs. https://docs.optimism.io/
- [17] Optimism. 2019. Introducing the OVM. https://medium.com/plasma-group/introducing-the-ovm-db253287af50
- [18] COW Protocol. [n. d.]. MEV Blocker. https://cow.fi/mev-blocker
- [19] Dan Robinson and Georgios Konstantopoulos. 2020. Ethereum is a Dark Forest.https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest
- [20] Dan Robinson and Dave White. 2024. Priority Is All You Need. https://www.paradigm.xyz/2024/06/priority-is-all-you-need
- [21] Mark Toda, Matt Rice, and Nick Pai. 2024. ERC-7683: Cross Chain Intents: An interface for cross-chain trade execution systems. https://eips.ethereum.org/EIPS/eip-7683
- [22] RT Watson. 2024. Uniswap hits a historic $2 trillion in trading volume. https://www.theblock.co/post/286679/uniswap-hits-a-historic-2-trillionin-trading-volume

## 免责声明

本文仅供一般信息用途。它不构成投资建议或购买或出售任何投资的推荐或招揽，也不应在评估任何投资决策的优劣时使用。它不应被用于会计、法律或税务建议或投资建议。本文反映了作者的当前观点，并非代表 Uniswap Labs、Paradigm、Flashbots、OP Labs 或其附属机构的意见，也不一定反映 Uniswap Labs、Paradigm、Flashbots、OP Labs 或其附属机构的意见。本文中反映的观点可能会在不更新的情况下发生变化。

---  
官方白皮书: [https://docs.unichain.org/whitepaper.pdf](https://docs.unichain.org/whitepaper.pdf)