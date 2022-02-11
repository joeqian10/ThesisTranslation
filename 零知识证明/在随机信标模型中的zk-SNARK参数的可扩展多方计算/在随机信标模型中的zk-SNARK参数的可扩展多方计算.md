# 在随机信标模型中的 zk-SNARK 参数的可扩展多方计算



我们将展示一个为 zk-SNARK 参数生成而构建的系统。zk-SNARKs 是用于任意计算的紧凑、高效且可公开验证的零知识证明。它们已成为可验证计算、隐私保护协议和区块链中很有价值的工具。当前实用的方案需要为每一个陈述在一个一次性设置中构造公共参考字符串 (CRS)。在此过程中的作恶将会导致伪造的证明，对加密货币类的应用可能导致数十亿美元的资产被盗。Ben-Sasson、Chiesa、Green、Tromer 和 Virza [9] 设计了一个多方协议来安全地计算一个这样的 CRS，该协议的一种适配版本被用于 Zcash 加密货币中构造 CRS [16]。这些协议的可信度受阻于需要一个“预先承诺阶段”，在这个阶段中需要强制提前选择极少数参与者，并要求他们在整个协议期间保护好他们秘密的随机性。我们主要的贡献是一种更具可扩展性的多方计算 (MPC) 协议，在随机信标模型 (*random beacon model*) 中保证安全，并且移除了预先承诺阶段。我们展示了即使一个作恶者对信标有有限的影响，安全性也是有保证的。接下来，我们应用我们的主要成果得到一个两阶段的协议来计算 Groth 的 zk-SNARK [27] 中的 CRS 的扩展版本。我们展示了当使用这个 CRS 时，在通用群模型中保持了知识的完备性。最后，我们实现和评估了我们的系统。

## 1	介绍

零知识简短非交互式知识论据 (zk-SNARKs) [12, 15, 24, 27-29, 32, 33, 35, 36] 在学术界和实际应用中已经有了越来越多的使用，从公共可验证计算到已经部署的匿名支付系统如 Zerocash [11] 和 Zcash [3] 及智能合约系统如 Ethereum。

尽管 zk-SNARKs 很强大，但在它们的广泛使用上仍存在挑战。最重要地，这些方案在公共参考字符串 (CRS) 模型中是安全的，它假设了一种对构造和验证证明所使用的参数的可信设置。生成这个 CRS 是一个主要的挑战，参数的损坏或颠覆意味着证明系统不再完备，例如，证明可以被伪造。在学术论文中经常假设这种可信设置参与方的存在；但实际上这些参与方难以找到，更难以让一个庞大而多样化的群体达成共识，并且面对现实世界部署中出现的有形的金钱收益可能不值得信赖。

当前部署的系统采用的方式是通过一个为了计算一个 CRS 的任务而从零开始构造的多方计算协议 [9, 16] 来生成 CRS。这些协议保证了当至少有一个参与者是诚实的时的完备性，例如证明是无法被伪造的；也保证了即使没有参与者是诚实的时的零知识性 [22]。然而，这些协议不能根本性地在参与者的数量上扩展，甚至可能在一些设置中对一个或两个参与者就会导致扩展的代价太高。这不是一个工程和优化的问题。根本上，这是一个密码学问题：因为 对付自适应性攻击者所需要的限制，当前 MPC 方案中的参与者必须提前承诺他们在参数中的部分并且在协议的整个过程中维持这些参数的可用性和安全性，即使是在他们大部分个人的计算已经结束以后。如果单个参与者中止，那么整个协议必须重启，所以必须很小心地排除掉会打断参数生成过程的攻击者。

这些密码学上的限制导致的结果就是参与者必须被提前仔细地选择，参与者的数量也极度有限，而且他们必须在协议整个过程中保持在线。这不仅扩大了攻击者的攻击面，也产生了实际问题因为参与者需要在协议过程中保管硬件。即使只有六个参与者，这个过程也花了2天 [37]。

即使 MPC 可以用一个单独的受信方来从“信任我”中移除对 zk-SNARKs 的设置，考虑到高昂的赌注，可以说它也不会走的足够远：例如，破坏 Zcash 中的 zk-SNARK CRS 将允许一个攻击者伪造等值于数百万美元的数字货币。在这种情况下，假设 100 或 1000 人中的一个是诚实的，比假设 6 或 10 人中的一个更有说服力。即使协议中包含数千个参与者是可能的，他们需要在协议开始前就被选好的事实既是一个后勤挑战，也是一个信任问题：谁来选择他们和谁来决定选择的数量已经足够？为了让 zk-SNARKs 在许多最引人注目的应用程序中使用，我们需要一个可以在现实世界中实际运行的协议，该协议可以扩展到数百或数千名参与者，并且不需要预先组织或选择。本文提出了一个高效的、实现了这些的协议。

**zk-SNARKs 的诉求**	对于一个正确的计算，zk-SNARKs 给出了它的可公开验证的恒定大小的零知识证明。证明的尺寸非常小 (取决于具体实现，即使对于很大的程序也在 160 [27] 到 288 [36] 字节之间) 并且只需花费不到 10 毫秒来验证。相反的，最好的不需要受信设置的方式的证明大小在数十到数百 KB [4] 甚至数个 MB [10]，并且验证时间的数量级在数百毫秒到数秒之间 [4, 10]。这使得 zk-SNARKs 在需要多次快速验证计算且空间非常宝贵的环境中成为独特的强大工具。

zk-SNARKs 有着很广泛的应用，从传统的密码学应用包括可验证的外包计算 [36] 和密码学基石的构造 [26]，到区块链、加密货币的应用以及所谓的“智能合约”。在任何这些应用的设置中通过一个颠覆的初始化设置过程来伪造证明都是有问题的。但正如我们稍后会看到的，颠覆的风险对于区块链应用来说尤其高。

**区块链的 zk-SNARKs**	zk-SNARKs 兑现了 “证明一次，随处 (几乎) 免费验证” 的承诺。这对于把它们使用在区块链和相关技术中给予了大量的关注因为维护区块链的数千个节点中的每一个都必须接收、验证和永久
并公开存储每笔交易，从而引发严重的可扩展性和隐私问题。zk-SNARKs 不仅可以剧烈改善区块链本身的效率和成本 [14, 19]，也可以被用于在区块链上构建复杂系统 [20, 31] 和用于解决大量基于区块链的应用的主要隐私和保密问题 [3, 20, 31]。

**面向高价值应用的群体扩展的参数生成**	如果至少有一个参与者是诚实的，那么 MPC 协议将产生一个诚实的 CRS。为了保证这个 CRS 是诚实的，我们想要包含尽可能多的参与者。这种扩展的需求导致证明的伪造和对破坏 CRS 产生过程的刺激。在一个质押了数百万或数十亿美元的系统中，假设大多数的参与者都是对抗性的，相信一小部分人中的一个并不能让人满意。

区块链为大量甚至比数字货币还有趣的应用提供了潜力，从分布式文件存储到身份和匿名凭证。许多这样的应用是投机的，不仅是在它们的成功还未得到证明这个层面上，在这里更相关的是，人们会认为这些应用将非常有利可图。在区块链中伪造证明当然是一种投机行为。但仅仅是得到数十亿美元的潜在可能，就既会激发对 CRS 生成过程的攻击，也会激发对参与者可信度的怀疑。

更进一步，价值数十亿美元的系统当前正在使用 zk-SNARKs。为它的隐私交易而使用 zk-SNARKs 的 Zcash 有价值近 10 亿美元的数字货币可能因为伪造的证明而被盗。市值大约 400 亿美元的 Ethereum 刚刚在智能合约中添加了对 zk-SNARKs 的期待已久的支持 [38]。而且，Ethereum 当前扩容的提案采用简短证明，一次失败将无疑会损失巨大。

**随机信标**	我们的协议使用一个随机信标。即使我们没有在本文中描述它精确构造的细节，我们也不能简单地假设它已经存在了，否则我们没有比直接假设一个 CRS 好多少。随机信标以固定的时间间隔产生公开可用且可验证的随机值。而且，我们的协议甚至允许作恶者篡改信标的少数比特位。信标本身可以被看作是一个 ”延迟了的“ 在一些高熵的和公开可用的数据上进行评估的哈希函数 [18] (例如，SHA256 的 $2^{40}$ 次循环 [1])。可能的数据源包括但不限于：某日股票市场的收盘价、一组选定的国家彩票的输出、或者一条或多条区块链某一高度的区块值。例如，Bitcoin 第 50000 个区块的哈希值 (撰写本文时，这将发生在 22 天后的将来)。

**随机信标 vs. 随机预言机**	我们强调清楚，随机信标和更广为人知的随机预言机的区别在于*它们的值直到某个特定的时隙才可用*。这意味着我们可以假设一个给定的随机信标值和一个作恶者在之前时隙中输出的值是*相互独立的*。(或者在作恶者对信标有影响的情况下，信标值具有取决于作恶者先前输出值的大量熵。)

这完全不同于一个随机预言机的值，后者取决于作恶者信息的熵可能为零 (例如，作恶者仅仅是简单地查询和输出预言机的值)。

### 1.1	我们的结果

本文中，我们设计、实现和评估了一个可扩展的开发参与的多方计算协议来生成 zk-SNARK 参数。我们旨在通过提供一种新的 zk-SNARKs 方案和适合现实世界使用的为了生成 CRS 的 MPC 系统来使得 zk-SNARKs 适合大规模使用。我们有三项贡献：

**参与者可交换的 MPC**	我们最主要的贡献是一种新的多方计算协议，一种*参与者可交换的 MPC* (px-MPC) 和一个高效的并已经实现的用来生成 CRS 的 px-MPC 协议。

px-MPC 是由一个参与方需要发送的消息的序列描述的；然而很重要的是，对这些消息发送方的身份没有限制。特别地，即使我们将讨论在每个阶段中所有参与者都参与一个循环过程的多阶段协议，也无需假设相同的参与者参与了不同的阶段。由于消息之间没有私有状态，参与者在发出每条消息后都有可能被换出或移除。

参与者可交换性避免了参与者预先选择的问题、对选择可靠的不会中途中止的参与者的需求、和让参与者在额外的时间保管敏感硬件的需要。唯一的要求就是在每个阶段中至少有一个参与者诚实地履行协议并且不和其他参与者串通。因此，该协议可以实际上扩展到无上限个参与者并在协议执行期间动态完成这些。例如，该协议是在线的和公开的。

该方法的关键就是使用随机信标来支持安全证明，从而减少对协议的限制。我们将证明即使一个作恶者对信标有有限的影响，协议也是安全的。

**具有高效且摊销的 px-MPC CRS 生成过程的 zk-SNARKs**	想要在实际中实现这个方案，我们必须选择一个特定的 zk-SNARK 并提供一个协议来生成它的 CRS。Groth 的 zk-SNARK 是目前最先进的协议，仅仅使用 3 个群元素来证明和 3 个配对来验证。我们证明了带有扩展了的允许一个两阶段的 px-MPC 协议的CRS 的 Groth 的 zk-SNARK 的安全性。更重要的是，第一个阶段对于陈述是不可知的，因此对所有很大 (但有界) 规模的陈述可以只执行一次。第二个阶段和特定陈述相关，但开销低的多，并且只需要每个参与者很小的工作量。这允许大部分设置的成本分摊到多个电路上。

**MMORPG，一个为了 zk-SNARK 参数生成而构建的系统和 BLS12-381，一条安全的用于 zk-SNARKs 的新曲线**	作为最后一个贡献，我们推出 MMORPG，一个为我们修改后的 Groth 的 zk-SNARK 版本构建的大规模多方开放可重用参数生成系统 (massively multi-party open reusable parameter generation)。我们评估了它的性能，表明了对于一个有 $2^{21}$ 个乘法门规模的电路，参与者在第一阶段中需要接收 1.2GB 的文件、执行一个在台式机上持续大约 13 分钟的计算、并产生一个 600MB 的文件。第二个阶段和特定陈述相关，但开销低的多。这允许大部分设置的成本分摊到多个电路上。

为了实现我们的协议，我们必须选择一条椭圆曲线来使用。现存的 zk-SNARK 的实现，比如在 Zcash 和 Ethereum 中使用的，采用一条支持配对的椭圆曲线，它被设计成对最初针对 128 位安全级别的 zk-SNARKs [13] 是高效的。然而，最近对于 Number Field Sieve 算法的优化已经使其安全性降级，所以我们采用了一条新的支持配对的椭圆曲线，称为 $BLS\mathcal{12}$-$\mathcal{381}$，它以 128 位安全性为目标，对性能的影响最小。我们提供了这条新椭圆曲线的一个稳定实现，使用 Rust 语言，有着有竞争力的性能、定义明确的序列化和跨平台支持。

### 1.2	大纲

本文的结构如下。我们将在第 2 章中给出我们方法的概述。在第 3 章中我们给出密码学预备知识、表示法、和支持引理。在第 4 章中阐述 MPC 协议的细节。在第 5 章阐述安全证明的细节。在第 6 章中我们使用 Groth 的 zk-SNARK 来实例化我们的协议。最后，在第 9 章中评估我们的实现。



## 2	我们方法的概述

我们的目标是在 个参与者和一个未受信的协调员之间构造一个实用的协议使得：

- 给出一个 zk-SNARK 的 CRS，满足如果至少 个参与者中的一个是诚实的，那么证明就无法被伪造
- 参与者的数量 没有限制
- 不需要预先选择参与者
- 不需要参与者预先承诺他们的随机数，因此也不需要在协议过程中保证它们的安全

达成这些目标的关键是移除在之前协议中使用的预先承诺阶段 [9, 16]。为此，我们围绕使用*随机信标*，一种到一个固定时间才可用的公共随机性的来源而设计了我们的协议。为了阐明我们的方法，现在我们展示怎样为一个玩具 CRS 构造一个协议和遇到的挑战的细节、为什么先前的工作中需要一个预先承诺、以及我们是如何移除它的。

**一个玩具 CRS**	为了简化展示，我们考虑一个只包含元素 $s\cdot g_1$ 和 $\alpha P(s)\cdot g_1$ 的 CRS，其中 $g_1$ 是一个阶为 $p$ 的群 $\mathbb{G}_1$ 的生成器；$s$ 和 $\alpha$ 是 $\mathbb{F}^*_p$ 中均匀的元素；以及 $P$ 是 $\mathbb{F}_p$ 上的一阶多项式 $P(x):=3x+5$。为了阐明主要思想，我们只分析两个参与方的情况，其中一方，Alice，是诚实的，另一方，Bob，是恶意的。在协议结束的时候，Alice 和 Bob 都不应该知道 $s$ 或 $\alpha$ 。

**一个 2 阶段的协议**	该协议由 2 个循环阶段组成。在阶段 1 中，每个参与者与另一方交流来计算 $s\cdot g_1$。在阶段 1 和 2 之间，未受信的协调员计算 $P(s)\cdot g_1$。最后，在阶段 2 中，一个 (潜在不同的) 参与者的集合计算 $\alpha P(s)\cdot g_1$。在每个阶段中，参与者发送一个消息，也接收一个消息。

**阶段 1**	阶段 1 中，Alice 和 Bob 需要对一个双方都不知道的均匀的 来计算 。

一个自然协议如下进行：Alice 选择一个均匀的 $s_1\in\mathbb{F}^*_p$，并发送 $M=s_1\cdot g_1$ 给 Bob。现在，Bob 需要选择一个均匀的 $s_2\in\mathbb{F}^*_p$ 来乘以 $M$。协议的输出被定义为 $s_2\cdot M=s_1s_2\cdot g_1$。

问题在于因为 Bob 是恶意的，他可以根据 Alice 的消息适应性地选择一个值 $s_2\in\mathbb{F}^*_p$，来操纵最后的输出值 $s_1s_2\cdot g_1$。基于此原因，在 [9, 16] 中加入了预先承诺阶段，Alice 和 Bob 对他们的值 $s_1,s_2$ 预先给出承诺。在下一个阶段中，Alice 和 Bob 将按照自然协议继续，但是加入了一个他们是使用承诺了的值 $s_1,s_2$ 的证明 (证明不会暴露 $s_1,s_2$ 的具体值)。这能阻止 Bob 适应性地选择 $s_2$。然而，预先承诺阶段有着之前提到过的缺点：

1. 最明显的，它在协议中多加了一个阶段
2. 参与者需要预先选择
3. 参与者需要预先选择他们的秘密元素并保护它们一段时间 (至少到广播他们的消息前，在所有随后的阶段中)

本文主要的意见是假设一个所有参与者都无法控制的公共随机性来源，例如，一个*随机信标*，我们就可以排除掉预先承诺阶段并仍可以阻止自适应性攻击。而且，即使当攻击者对随机信标有一些影响，我们仍然可以做到这些。

有了随机信标，我们协议的一个简化版本，仍然有第一个参与者是诚实的，第二个是恶意的，将按如下方式进行：

1. Alice 选择一个随机的 $s_1\in\mathbb{F}^*_p$ 并广播 $M=s_1\cdot g_1$。
2. Bob (用某种方式) 选择一个 $s_2\in\mathbb{F}^*_p$ 并广播 $M'=s_1s_2\cdot g_1$。
3. 协调员调用随机信标得到一个均匀的 $s_3\in\mathbb{F}^*_p$，协议的输出被定义为 $s_3\cdot M'=s_1s_2s_3\cdot g_1$。

注意到对于均匀的 $s\in\mathbb{F}^*_p$ 协议输出是 $s\cdot g_1$ 而不论 Bob 对 $s_2$ 的选择。你可能会问，为什么不跳过两个参与者而直接使用随机信标的输出 $s_3\in\mathbb{F}^*_p$ 来输出 $s_3\cdot g_1$？原因是没有任何参与者，或者更一般的，没有任何排除了至少一个参与者的串通参与者群体可以知道 $s$。这意味着我们不能使用*公开的*随机信标来选择秘密的 $s$，我们只能用它来随机化 $s$ 的选择。你可能也会问我们为什么不需要相信协调员。答案是简单的，给定公开的随机信标值 $s$，协调员以确定性和可验证的方式行事，可以通过简单地重复计算来检查。

**阶段 2**	注意，阶段 1 后，$\mathcal{R}$ 是公开值 $s\cdot g_1,g_1$ 的一个线性组合
$$
P(s)\cdot g_1=3\cdot(s\cdot g_1)+5\cdot g_1
$$
因此，协调员可以有效地计算 $P(s)\cdot g_1$。

阶段 2 可以和阶段 1 相似地进行：Alice 选择一个随机的 $\alpha_1\in\mathbb{F}^*_p$ 并广播 $M=\alpha_1P(s)\cdot g_1$。Bob 和协调员像他们在阶段 1 中做的那样操作。我们最终有了一个值 $\alpha P(s)\cdot g_1$；其中 $\alpha_2$ 是由 Bob 选择的，$\alpha_3$ 是由随机信标选择的。

最后，我们强调在证明中假设随机信标有低的协熵是足够的；因此该协议在攻击者对信标有有限影响的情况下也是可行的。

详细信息参考定理 5.1。



## 3	预备知识

### 3.1	表示法

我们将在阶为质数 $p$ 的双线性群 $\mathbb{G}_1,\mathbb{G}_2$ 和 $\mathbb{G}_T$ 上开展工作，对应的生成器分别是 $g_1,g_2$ 和 $g_T$。这些群配备有一个非退化的双线性映射 $e:\mathbb{G}_1\times\mathbb{G}_2\to\mathbb{G}_T$，即 $e(g_1,g_2)=g_T$。我们把 $\mathbb{G}_1$ 和 $\mathbb{G}_2$ 写作加法性的，把 $\mathbb{G}_T$ 写作乘法性的。对于 $a\in\mathbb{F}_p$，我们记作 $[a]_1:=a\cdot g_1,[a]_2:=a\cdot g_2$。我们使用这样的表示法 $\mathrm{G}:=\mathbb{G}_1\times\mathbb{G}_2$ 和 $\mathrm{g}:=(g_1,g_2)$。给定一个元素 $h\in\mathrm{G}$，我们把 $h$ 的 $\mathbb{G}_1(\mathbb{G}_2)$元素表示为 $h_1(h_2)$。我们把 $\mathbb{G}_1,\mathbb{G}_2$ 的非零元素表示为 $\mathbb{G}^*_1,\mathbb{G}^*_2$，且有 $\mathrm{G}^*:=\mathbb{G}^*_1\times\mathbb{G}^*_2$。

我们假设我们有一个生成器 $\mathcal{G}$，和统一选择的生成器 $g_1\in\mathbb{G}_1,g_2\in\mathbb{G}_2$ 一起，接收一个参数 $\lambda$ 并至少在包含 $\lambda$ 的超多项式时间内返回上述的三个阶为质数 $p$ 的群。我们假设 $\mathbb{G}_1,\mathbb{G}_2$ 中的群操作和映射 $e$ 能在 $\mathrm{poly}(\lambda)$ 时间内计算。当我们说一个事件有概率 $\gamma$，意味着除了任何其他在事件描述中指出的概率，它在 $\mathcal{G}$ 的随机性上有概率 $\gamma$。

当我们说一个参与方 $\mathcal{A}$ 是*高效的*，我们想说它是一个不均匀的电路序列，其索引是 $\lambda$，大小为 $\mathrm{poly}(\lambda)$。当我们说 $\mathcal{A}$ 是一个高效的预言机电路，我们想说它在上述定义上是高效的，并且在它执行期间可能会向一个预言机 $\mathcal{R}$ 作出 $\mathrm{poly}(\lambda)$ 次查询，而且它的输入是任意长度的字符串，输出是 $\mathbb{G}^*_2$ 中的元素。

我们假设在协议中这样的参与方 $\mathcal{A}$ 都可以使用相同预言机 ，它们的输出是 $\mathbb{G}^*_2$ 的均匀独立元素。

对于 $a\in\mathbb{F}_p$ 和 $C\in\mathrm{G}$，我们把 $C$ 和 $a$ 的坐标标量乘法表示为 $a\cdot C$；即 $a\cdot C:=(a\cdot C_1,a\cdot C_2) \in\mathrm{G}$。我们也允许有相同长度的向量在坐标上的操作。例如，对于 $a\in\mathbb{F}^t_p$ 和 $\mathrm{x}\in\mathbb{G}^t_1$，$a\cdot\mathrm{x}:=(a_1\cdot\mathrm{x_1},...,a_t\cdot\mathrm{x_t})$。

我们把 $\mathsf{acc}$ 和 $\mathsf{rej}$ 分别当作真和假。因此当我们对于一个函数  $f(x)$  和输入 $x$ 说“检查 $f(x)$”，我们想说的是检查 $f(x)=\mathsf{acc}$。

我们使用缩写 e.w.p. 来表示 “除非有概率 (except with probability)”；例如，e.w.p. $\gamma$ 表示 “最少有 $1-\gamma$ 的概率”。

我们假设在一个同步的设置中，我们有正整数个时间 “槽”；我们认定在槽 $J$ 中，参与方知道在槽 $1,...,J-1$ 中的消息内容和消息发出者。

### 3.2	随机信标

我们假设有一个任由我们处置的可以输出 $\mathbb{F}^*_p$ 中元素的“随机信标” $\mathsf{RB}$。我们把 $\mathsf{RB}$ 定义为一个以一个时隙 $J$ 和一个正整数 $k$ 为输入，以 $k$ 个元素 $a_1,...,a_k\in\mathbb{F}^*_p$ 为输出的函数。假设 $\mathsf{RB}$ 仅被定义为值 $J$ 的子集作为其第一个输入，这将是很方便的。如果对于定义了 $\mathsf{RB}$ 的任何正整数 $J$ 和 $k$：对任何在时隙 $J$ 前由 $\mathcal{A}$ 生成的随机变量 $X$ - 例如，使用在 $J'<J$ 时对 $\mathsf{RB}(J',k')$ 的调用、和在 $\mathcal{A}$ 是一个预言机电路时对预言机 $\mathcal{R}$ 的调用、和遵循一个 $\mathcal{A}$ 被设计参与到其中的协议的诚实参与者的消息 $H$；$\mathsf{RB}(J,k)$ 的分布在 $(\mathbb{F}^*_p)^k$ 上是均匀的并且独立于 $(\mathsf{rand}_\mathcal{A},X)$，其中 $\mathsf{rand}_\mathcal{A}$ 是 $\mathcal{A}$ 的随机性，我们说 $\mathsf{RB}$ 对 $\mathcal{A}$ 是*抵抗的* (resistant)。

我们现在将这个定义推广到模型中的作恶者对信标的值有有限影响的情况。如果对于定义了 $\mathsf{RB}$ 的任何正整数 $J$ 和 $k$：如上所述，对任何在时隙 $J$ 由 $\mathcal{A}$ 生成的随机变量 $X$，以任何固定 $(\mathsf{rand}_\mathcal{A},X)$ 的条件下 $\mathsf{RB}(J,k)$ 的分布有最多为 $u$ 的共最小熵 (co-min-entropy) (例如，最小熵最小为 $k\cdot\mathrm{log}|\mathbb{F}^*_p|-u$)，我们说 $\mathsf{RB}$ 对 $\mathcal{A}$ 是* (resistant)。

我们的协议始终具有循环的性质，其中参与者 $P_i$ 在每一个阶段中紧随参与者 $P_{i-1}$ 之后发送一个单独的信息，在每一个阶段的最后在参与者 $P_N$ 的消息发送后的时隙中 $\mathsf{RB}$ 将会被调用。因此，我们隐含地假设协议定义了参与者  $P_i$  发送在 $\ell$ 阶段消息的时隙是 $J=(\ell-1)\cdot(N+1)+i$。在此环境中，当且仅当 $J$ 是 $N+1$ 的倍数时，能比较方便地假设 $\mathsf{RB}(J,k)$ 是被定义的。

### 3.3	输入域

我们在所有方法描述中都隐含地假设，如果输入不在规定范围内，则方法输出 $\mathsf{rej}$。这意味着在一个该协议的实现中，例如一个期望输入在 $\mathbb{G}^*_2$ 中的方法会检查收到的输入的确是在这个范围内的，否则就会输出 $\mathsf{rej}$。

### 3.4	可交换玩家的协议和自适应的对手

我们假设在协议的每个阶段都有 $N$ 个参与者 ${P_1,...,P_N}$。即使我们对每个阶段使用该表述法，但我们没有假定 $P_i$ 在每个阶段是相同的参与者，我们也没有假定在协议中参与者的身份或者行为在他们发送消息的时隙前就已经被决定了。实际上，很可能参与者 $P_i$ 只是简单地停止往记录副本中添加任何内容。

当我们讨论当 $1\le K\le N$ 时，一个对手 $\mathcal{A}$ 在该协议中控制 $K$ 个参与者，意味着 $\mathcal{A}$ 能够在每个阶段中自适应地选择 $K$ 个参与者的一个不同子集去控制。也就是说，如果他们在阶段 $\ell$ 中还没有选择 $K$ 个参与者，那么在时隙 $(\ell-1)\cdot(N+1)+i$ 中他们能选择是否要在阶段 $\ell$ 中去控制参与者 $P_i$。

我们把到第 $i$ 个参与者在阶段 $\ell$ 中发送他的消息为止的记录副本表示为 $\mathsf{transcript}_{\ell,i}$。

### 3.5	初步声明

以下声明不难证明。

**声明 3.1.** 令 $A,B$ 为两个随机变量，使得对于任何 $A$ 的固定值 $a$，$B|A=a$ 具有最多是 $u$ 的共最小熵。令 $P$ 是一个有着输出范围为 $\{\mathsf{acc,rej}\}$ 的判别。$A$ 在 $B$ 的范围内均匀分布，令 $B'$ 是一个独立于 $A$ 的随机变量。那么有
$$
\mathrm{Pr}(P(A,B')=\mathsf{acc})\ge2^{-u}\cdot \mathrm{Pr}(P(A,B)=\mathsf{acc})
$$

### 3.6	辅助方法

我们使用配对 $e$ 来定义一些方法用来检查元素之间是否存在特定比率。以下定义和声明是从 [16] 中发布的。

**声明3.2.** 给定 $A,B\in\mathbb{G}^*_1$ 和 $C,D\in\mathbb{G}^*_2$，当且仅当存在 $s\in\mathbb{F}^*_p$ 使得 $B=s\cdot A$ 和 $D=s\cdot C$ 时有 $\mathsf{SameRatio}((A,B),(C,D))=\mathsf{acc}$。

___

**算法1** 判断是否存在 $x\in\mathbb{F}^*_p$ 使得 $B=x\cdot A$ 和 $D=x\cdot C$。

---

**要求：** $A,B\in\mathbb{G}_1$ 和 $C,D\in\mathbb{G}_2$ 且 $A,B,C,D$ 都不是单位元。
$$
\begin{flalign}
&1:\; \mathbf{function}\; \mathrm{SameRatio}((A, B),(C, D)) &\\
&2:\; \qquad \mathbf{if}\; e(A,D)=e(B,C)\; \mathbf{then} &\\
&3:\; \qquad\qquad \mathbf{return}\; \mathsf{acc} &\\
&4:\; \qquad \mathbf{else} &\\
&5:\; \qquad\qquad \mathbf{return}\; \mathsf{rej} &\\
&6:\; \qquad \mathbf{end\;if} &\\
&7:\; \mathbf{end\;function} &\\
\end{flalign}
$$

---

**算法2** 判断 $A$ 和 $B$ 之间的比率是否是在 $C$ 中编码的 $s\in\mathbb{F}^*_p$。

***

**要求：** $A,B\in\mathbb{G}^2_1$ 或 $A,B\in\mathbf{G}^2_1$。$C\in\mathbb{G}^*_2$ 或 $C\in(\mathbb{G}^*_2)^2$​。
$$
\begin{flalign}
&\ \ 1:\; \mathbf{function}\; \mathrm{Consistent}(A,B,C) &\\
&\ \ 2:\; \qquad \mathbf{if}\; C\in(\mathbb{G}^*_2)^2\; \mathbf{then} &\\
&\ \ 3:\; \qquad\qquad r\gets\mathsf{SameRatio}((A_1,B_1),(C_1,C_2)) &\\
&\ \ 4:\; \qquad \mathbf{else} &\\
&\ \ 5:\; \qquad\qquad r\gets\mathsf{SameRatio}((A_1,B_1),(g_2,C)) &\\
&\ \ 6:\; \qquad \mathbf{end\;if} &\\
&\ \ 7:\; \qquad \mathbf{if}\; A,B\in\mathbb{G}_1\; \mathbf{then} &\\
&\ \ 8:\; \qquad\qquad \mathbf{return}\; r &\\
&\ \ 9:\; \qquad \mathbf{else} &\\
&10:\; \qquad\qquad \mathbf{return}\; r\; \mathbf{AND}\; \mathsf{SameRatio}((A_1,B_1),(A_2,B_2)) &\\
&11:\; \qquad \mathbf{end\;if} &\\
&12:\; \mathbf{end\;function} &\\
\end{flalign}
$$

****

我们后面将使用推荐的表述法 $\mathsf{consistent}(A-B;C)$ 来表示上述的带有输入 $A,B,C$ 的方法。当 $c\in\mathbf{G}$ 时我们进一步重载 $\mathsf{consistent}(a-b;c)$ 来表达 $\mathsf{consistent}(a-b;c_2)$ 。

### 3.7	知识的证明

我们将使用一种根据指数知识假设的离散对数知识证明方案。

**定义 3.3** (指数知识假设 (KEA))。对任何高效的 $\mathcal{A}$，存在一个高效的确定性的 $\mathtt{X}$ 使得下述情况成立。考虑下面这个实验。$\mathcal{A}$ 被给定一个任意的 “辅助信息字符串” $z$ ，和一个独立于 $z$ 的均匀选择的 $r\in\mathbb{G}^*_2$。然后 $\mathcal{A}$ 生成 $x\in\mathbb{G}^*_1$ 和 $y\in\mathbb{G}^*_2$。$\mathtt{X}$ 被给定相同的输入 $r$ 和 $z$ 以及  $\mathcal{A}$  的内部随机性，输出 $\alpha\in\mathbb{F}^*_p$。下面两者的概率

1.  $\mathcal{A}$ “成功了”，例如，$\mathsf{SameRatio}((g_1,x),(r,y))$，
2.  $\mathtt{X}$ “失败了”，例如， $x\ne[\alpha]_1$，

都是 $\mathrm{negl}(\lambda)$。

**备注 3.4.** 注意，该假设是标准指数知识假设，除了它把元素划分到 $\mathbb{G}_1$ 和 $\mathbb{G}_2$：假设 $r=[\gamma]_2$ 和 $x=[\alpha]_1$。那么 $\mathsf{SameRatio}((g_1,x),(r,y))$ 意味着 $y=[\alpha\cdot\gamma]_2$。因此 $(x,y)$ 是一个具有 “比例” $\gamma$ 的配对，是从给定的同样具有比例 $\gamma$ 的配对 $(g_1,r)$ 中产生的；而且指数知识假设阐明了要生成这样的一个配对，我们必须知道原始配对的比例，也就是 $\alpha$。

注意，指数知识假设通常是指用乘法表示的群，因此这里 “系数知识假设 (KCA)” 可能是一个更好的名称。

***

**算法3** 构造一个 $\alpha$ 的知识证明  判断是否存在 $x\in\mathbb{F}^*_p$ 使得 $B=x\cdot A$ 和 $D=x\cdot C$。

---

**要求：** $\alpha\in\mathbb{F}^*_p$
$$
\begin{flalign}
&1:\; \mathbf{function}\; \mathrm{POK}(\alpha, \text{string}\; v)) &\\
&2:\; \qquad r\gets \mathcal{R}([\alpha]_1,v)\in\mathbb{G}^*_2 &\\
&5:\; \qquad \mathbf{return}\; ([\alpha]_1,\alpha\cdot r) &\\
&7:\; \mathbf{end\;function} &\\
\end{flalign}
$$

---

___

**算法4** 验证一个 $\alpha$ 的知识证明

___

**要求：** $a\in\mathbb{G}^*_1,b\in\mathbb{G}^*_2$
$$
\begin{flalign}
&1:\; \mathbf{function}\; \mathrm{CheckPOK}(a, \text{string}\; v,b)) &\\
&2:\; \qquad r\gets \mathcal{R}(a,v)\in\mathbb{G}^*_2 &\\
&5:\; \qquad \mathbf{return}\; \mathsf{SameRatio}((g_1,a),(r,b)) &\\
&7:\; \mathbf{end\;function} &\\
\end{flalign}
$$

___

**声明 3.5.** 在指数知识假设下，对任何高效的预言机电路 $\mathcal{A}$，存在一个高效的 $\mathtt{X}$ 使得下述情况成立。给定任何没有通过查询预言机 $\mathcal{R}$ 而生成的字符串 $z$，和随机的预言机响应 $r_1,...,r_\ell$，$\mathcal{A}$ 生成 $a\in\mathbb{G}_1,y\in\mathbb{G}_2$ 和一个字符串 $v$；$\mathtt{X}$ 被给定相同的输入 $r$ 和 $z$ 以及  $\mathcal{A}$  的内部随机性，输出 $\alpha\in\mathbb{F}^*_p$。下面两者的概率

1.  $\mathcal{A}$ “成功了”，例如，$\mathsf{CheckPOK}(a,v,y)=\mathsf{acc}$，
2.  $\mathtt{X}$ “失败了”，例如， $a\ne[\alpha]_1$，

都是 $\mathrm{negl}(\lambda)$。

*证明：*固定 $\mathcal{A}$ 和 $z$ 使得对于给定的 $z$ 和访问预言机 $\mathcal{R}$ 的权限，$\mathcal{A}$ 生成了一个配对 $a\in\mathbb{G}_1,y\in\mathbb{G}_2$ 和一个字符串 $z$。令 $\ell=\mathrm{poly}(\lambda)$ 是 $\mathcal{A}$ 对预言机 $\mathcal{R}$ 的调用次数。我们可以把 $\mathcal{A}$ 当作一个 $z$、$\mathcal{R}$ 的响应序列 $\mathbf{r}=r_1,...,r_\ell$ 和其内部随机性 $\mathsf{rand}_\mathcal{A}$ 的确定性函数。对于 $i\in[\ell]$，我们构造 $\mathcal{A}_i$ 使得对于给定的 $z$ 和 $r\in\mathbb{G}_2$ 将达成下述操作。对 $(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$ 调用 $\mathcal{A}$，其中 $r_j$ 是对 $j\ne i$ 作均匀选择的，且 $r_i=r$；$\mathsf{rand}_\mathcal{A}$ 也是均匀选择的。令 $(a,v,y):=\mathcal{A}(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$，令 $q_1,...,q_\ell$ 是查询 $\mathcal{R}$ 的序列。令 $D_i$ 是 $(\mathbf{r},\mathsf{rand}_\mathcal{A})$ 的集合使得 $q_i=(a,v)$ 且 $i$ 是第一个这样的索引。如果 $(\mathbf{r},\mathsf{rand}_\mathcal{A})\notin D_i$，$\mathcal{A}_i$ 中止。否则，$\mathcal{A}_i$ 输出 $(a,y)$。通过指数知识假设，存在一个高效的 $\mathtt{X}_i$ 使得下面两者在均匀的 $\mathbf{r},\mathsf{rand}_\mathcal{A}$ 上的概率

1.  $\mathsf{SameRatio}((g_1,a),(r,y))$，
2.  $\mathtt{X}_i$ 对于给定的 $z,\mathbf{r},\mathsf{rand}_\mathcal{A}$ 没有输出 $\alpha$ 使得 $a=[\alpha]_1$，

都是 $\mathrm{negl}(\lambda)$。我们可以把 $\mathcal{A}_i$ 当作一个确定性函数 $\mathcal{A}_i(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$，它把 $r_i$ 当作它的输入 $r$，并把 $r_1,...,r_{i-1},r_{i+1},...,r_\ell$ 作为它的随机性来回应在 $j\ne i$ 时对 $\mathcal{R}$ 的调用。我们可以采用同样的方法把 $\mathtt{X}_i$ 当作一个函数 $\mathtt{X}_i(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$。

现在我们如下构造一个高效的 $\mathtt{X}$。$\mathcal{A}(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$ 和它的输出 $(a,v,y)$ 作出的对 $\mathcal{R}$ 的查询序列 $q_1,...,q_\ell$ 是由 $\mathtt{X}$ 决定的。假设对于某些 $i\in[\ell]$ 有 $(\mathbf{r},\mathsf{rand}_\mathcal{A})\in D_i$，那么 $\mathtt{X}$ 返回 $\mathtt{X}_i(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$；否则 $\mathtt{X}$ 中止。现在假设 $(\mathbf{r},\mathsf{rand}_\mathcal{A})\in D_i$ 且 “$\mathcal{A}$ 打败了 $\mathtt{X}$”。也就是

1.  $\mathsf{CheckPOK}(a,v,y)=\mathsf{acc}$，
2.  $\mathtt{X}(z,v,\mathbf{r},\mathsf{rand}_\mathcal{A})=\alpha$ 其中 $a\ne[\alpha]_1$。

我们有 $\mathcal{R}(a,z)=r_i$ 和 $\mathtt{X}_i(z,\mathbf{r},\mathsf{rand}_\mathcal{A})=\mathtt{X}(z,\mathbf{r},\mathsf{rand}_\mathcal{A})$。因此

1.  $\mathsf{SameRatio}((g_1,a),(r_i,y))$，
2.  $\mathtt{X}(z,v,\mathbf{r},\mathsf{rand}_\mathcal{A})=\alpha$ 其中 $a\ne[\alpha]_1$。

但是这只会在 $(\mathbf{r},\mathsf{rand}_\mathcal{A})$ 的 $\mathrm{negl}(\lambda)$ 分数下发生。并且，如果对于任何 $i\in\ell$ 都有  $(\mathbf{r},\mathsf{rand}_\mathcal{A})\notin D_i$，那么 $\mathcal{R}(a,v)$ 的值还是未知数并也是均匀分布的，所以 $\mathsf{CheckPOK}(a,v,y)$ 的概率是 $\mathrm{negl}(\lambda)$。

一个捆绑在 $i\in\ell$ 上的并集现在给出了声明。



## 4	参数生成的多方计算

现在开始描述我们的协议。

### 4.1	电路结构

我们假设有一个在 $\mathbb{F}_p$ 上定义的算术电路 $\mathbf{C}$，可能它看起来是一个特例，但是它能帮助我们简化 [9] 中的协议设计，并且也满足一个在第 6 章中描述的用来计算 [27] 中的扩展 CRS 的电路要求。

该电路包含交替的乘法/除法层 $C_1,...,C_d$ 和线性组合层 $L_1,...,L_d$。我们把 $d$ 称为电路的*深度*。(在常规意义上一层的深度可以大于1。) 电路的输入 $\mathrm{x}$ 根据各个层被分为了*不相交集* $\mathrm{x}^1,...,\mathrm{x}^d$。具体地，我们把 $\mathrm{x}^\ell$ 记作乘法/除法层 $C_\ell$ 的输入，有时我们也用符号 $x\in C_\ell$ 来表示 $x\in \mathrm{x}^\ell$。我们把 $\mathrm{x}$ 和 $\mathrm{x}^\ell$ 认为是*枚举集*，并把它们当作函数的输入向量。一个乘法/除法层 $C$ 满足以下这些：

1. $C$ 中所有门的输出都是电路的输出。
2. $C=C_l$ 对每一个输入 $x\in \mathrm{x}^\ell$ 都有一个*输入门*。当另一个门希望使用这些输入中的一个，它会从对应的输入门引一条连接线。特别地，每个输入都是电路输出的一部分。
3. $C$ 中除了输入门的所有门都是双扇入 (fan-in two) 的乘法和除法门。这些门的左侧输入是 $C$ 或者上一层中门的输出；右侧输入是 $C$ 中的一个输入门。在确定是除法门的情况下，右侧输入永远都是分母。

一个线性组合层 $L$ 包含不固定扇入的线性组合门，它们的输入是 $L$ 或者上一层中门的输出。

### 4.2	协议协调员

除了参与者的消息以外，协议描述中也包含要由 *协议协调员* 发送的消息。这些消息是协议描述和到那时为止的记录副本的一个确定性函数。实际上，有一个计算力强大的参与方来填补这个角色是有帮助的。然而，这个参与方并不需要被信任，并且之后任何人都可以验证记录副本中的协议协调员的消息是正确的。特别地，协议验证者除了要做之前明确描述的那些步骤，还要独立地计算协议协调员的消息和检查它们是正确的。

### 4.3	多方计算 (MPC)

协议的目标是对统一选择的 $\mathrm{x}\in (\mathbb{F}^*_p)^t$ 计算 $\mathbf{C}(\mathbf{x})\cdot\mathbf{g}$ ，其中 $t$ 是 $\mathbf{C}$ 的输入个数。进一步来说，我们有 $\mathbf{x}=\mathbf{x}_1\cdots\mathbf{x}_N\cdot\mathbf{x}'$ (回顾一下这个乘积是以坐标定义的)，其中 $\mathrm{x}_i\in (\mathbb{F}^*_p)^t$ 是 $P_i$ 的输入，$\mathbf{x}'$是一个随机信标输出。

把 $\mathbf{C}$ 的各层表示为 $C_1,L_1,...,C_d,L_d$。该协议由对应于各个层的 $d$ 个阶段组成。

### 4.4	阶段的结构

我们固定一个层 $\ell\in [1..d]$ 并表示为 $C=C_\ell,L=L_\ell$。我们假设对于上一层 $C_1,L_1,...,C_\ell-1,L_\ell-1$ 的所有门 $\mathrm{g}$，我们已经计算出了一个输出值 $[\mathrm{g}]\in \mathbf{G}$。

注意，每个门 $\mathrm{g}\in C$ 的输出是 $C$ 的输入中的一个洛朗单项式 (Laurent monomial，比如两个单项式的比值)  ，可能被乘上了上一层某个门 $\mathsf{g'}$ 的输出。这个单项式表示为 $M_\mathsf{g}$，上一层的输出表示为 $\mathsf{g_src}$；如果不存在这样的输出，则使 $\mathsf{g_src}:=\mathbf{g}$。

1. 对 $j\in [N]$，参与者 $j$ 完成以下这些：

   1. 对每个在 $C$ 中使用的输入 $x$，输出 $[x_j]_1$ 和 $y_{x,j}:=\mathsf{POK}(x_j,v)$，其中 $v=\mathsf{transcript}_{\ell,j-1}$ 是当前参与者开始之前的协议记录副本。

   2. 对每个门 $\mathsf{g}\in C$：

      - 如果 $j=1$，输出 $[\mathsf{g}]^\mathbf{1}:=M_\mathsf{g}(\mathbf{x}^\ell_1)\cdot \mathsf{g_src}$。

      - 否则，当 $j>1$ 时，输出 $[\mathsf{g}]^\mathbf{j}:=M_\mathsf{g}(\mathbf{x}^\ell_j)\cdot [\mathsf{g}]^\mathbf{j-1}$。

2. 令 $J-1$ 作为 $P_N$ 在此阶段中应该广播消息的时隙。 协议协调员对每个 $\mathsf{g}\in C$ 计算并输出 $\mathbf{x}'^\ell:=\mathsf{RB}(J,t_\ell)$ 和 $[\mathsf{g}]:=M_\mathsf{g}(\mathbf{x}'^\ell)\cdot [\mathsf{g}]^\mathbf{N}$。

3. 最后，在同一个时隙中，协议协调员在线性组合层 $L=L_\ell$ 计算并输出所有门 $\mathsf{g}$ 的值 $[\mathsf{g}]$。

#### 验证：

对每个 $j\in [N]$，协议验证者完成以下这些：

1. 对每个输入 $x\in C$，令 $r_{x,j}:=\mathcal{R}([x_j]_1,\mathrm{transcript}_{l,j-1})$，检查 $\mathsf{CheckPOK}([x_j]_1,\mathsf{transcript}_{\ell,j-1},y_{x,j})$ 和 $\mathsf{consistent}([x]^\mathbf{j-1}-[x]^\mathbf{j};(r_{x,j},y_{x,j}))$ 。
2. 令 $\mathsf{g_L}$ 和 $\mathsf{g_R}$ 作为门 $\mathsf{g}$ 的输入。
3. 如果 $\mathsf{g_L}\in C$，那么
   - 如果 $\mathsf{g}$ 是一个乘法门，检查 $\mathsf{consistent}([\mathsf{g_L}]^\mathbf{j}-[\mathsf{g}]^\mathbf{j};[\mathsf{g_R}]^\mathbf{j})$。
   - 如果 $\mathsf{g}$ 是一个除法门，检查 $\mathsf{consistent}([\mathsf{g}]^\mathbf{j}-[\mathsf{g_L}]^\mathbf{j};[\mathsf{g_R}]^\mathbf{j})$。
4. 如果 $\mathsf{g_L}$ 是从上一层接入，那么
   - 如果 $\mathsf{g}$ 是一个乘法门，检查 $\mathsf{consistent}([\mathsf{g_L}]-[\mathsf{g}]^\mathbf{j};[\mathsf{g_R}]^\mathbf{j})$。
   - 如果 $\mathsf{g}$ 是一个除法门，检查 $\mathsf{consistent}([\mathsf{g_L}]^\mathbf{j}-[\mathsf{g}];[\mathsf{g_R}]^\mathbf{j})$。



## 5	安全证明

我们把电路 $\mathbf{C}$ 在均匀选择的输入下编码后的输出表示为一个随机变量 $\mathbf{C}_S$。也就是对于 $\mathrm{s}\in (\mathbb{F}^*_p)^t$，$\mathbf{C}_S:=[\mathbf{C}(s)]$。

令 $\mathcal{A}$ 是 3.4 节中描述的能够在每个阶段中控制 $N-1$ 个参与者的一个子集的一个作恶者。我们用 $\mathbf{C}_\mathcal{A}$ 来表示每个阶段中 $\mathcal{A}$ 和一个诚实的参与者同时参与到这个协议中后所产生的电路输出。我们认为 $\mathcal{A}$ 在协议结束之后输出一个字符串 $\mathsf{z}$。 $\mathbf{C}_\mathcal{A}$ 和 $\mathsf{z}$ 是由一个由 $\mathcal{A}$ 的随机性 $\mathsf{rand}_\mathcal{A}$、诚实参与者的输入-包含 $\mathbb{F}^*_p$ 中均匀分布的独立元素和随机预言机 $\mathcal{R}$ 的输出 -是 $\mathbb{G}_2$ 中均匀分布的元素；和随机信标的输出 $\mathsf{rand}_\mathsf{beacon}$ (是 上的元素，可能有一些有限的影响) 组成的函数决定的随机变量。

对于一个输出范围为 $\{\mathsf{acc,rej}\}$ 的判别 $P$，我们定义
$$
\mathrm{adv}_{\mathcal{A},P}:=\mathrm{Pr}(P(\mathbf{C}_\mathcal{A},\mathsf{z})=\mathsf{acc})
$$
注意，$\mathrm{adv}_{\mathcal{A},P}$ 取决于 $\mathsf{RB}$ 和 $\mathcal{A}$ 对 $\mathsf{RB}$ 的影响有多大。我们认为 $\mathsf{RB}$ 是固定的，所以没有把它作为一个额外的参数。

**定理 5.1.**  固定任何高效的预言机电路 $\mathcal{A}$ 和 $u>0$。固定参与者的数量 $N$ 并有 $N(\lambda)=\mathrm{poly}(\lambda)$。存在一个高效的 $\mathcal{B}$ 使得如果 $\mathsf{RB}$ 对 $\mathcal{A}$ 是 u-co-resistant 的，那么对每一个判别 $P$ 有
$$
\mathrm{Pr}(P(\mathbf{C}_S,\mathcal{B}(\mathbf{C}_S))=\mathsf{acc})\ge2^{-ud}\cdot \mathrm{adv}_{\mathcal{A},P}-\mathrm{negl}(\lambda)
$$
假设 是一个判别，它同时担任一个带有一些固定公共输入的 zk-SNARK 验证者角色，它把第一个输入作为 zk-SNARK 的参数，把第二个输入作为证明；取一个常数 $d$ 和 $u=O(\mathrm{log}\;\lambda)$。以上定理表明如果 $\mathcal{A}$ 对于独立生成的参数有不可忽略的概率不能构造正确的证明，那么它对它所参与的协议中生成的参数就不能这样做。

*证明：* 用 $H$ 来表示每个阶段中诚实参与者的输入集合。用 $\mathsf{rand}_\mathsf{beacon}$ 来表示在每个阶段结束的时候随机信标给协议协调员的响应。用 $\mathsf{rand}_\mathsf{oracle}$ 来表示随机预言机对诚实参与者 (当对于 $x\in H$ 计算  $\mathsf{POK}(x,z)$ 时) 和对 $\mathcal{A}$ 的查询的响应。电路输出 $\mathbf{C}_\mathcal{A}$ 和字符串 $\mathsf{z}$。$\mathcal{A}$ 将在协议能被看作一个关于 $\mathsf{x}=(\mathsf{rand}_\mathcal{A},H,\mathsf{rand}_\mathsf{oracle},\mathsf{rand}_\mathsf{beacon})$ 的函数的时候才输出。称这个函数为 $F$；例如 $F(\mathsf{x})=(\mathbf{C}_\mathcal{A}(\mathsf{x}),\mathsf{z(x)})$。令这样的 $\mathsf{x}$ 的集合为 $\mathcal{X}$。我们对 $\mathsf{RB}$ 有 $d$ 次调用 - 根据字符串 $\mathsf{rand}_\mathsf{beacon}=\mathsf{rand}_\mathsf{beacon\ 1},...,\mathsf{rand}_\mathsf{beacon\ d}$，每个阶段结束时一次。因为 $\mathsf{RB}$ 对 $\mathcal{A}$ 是 u-co-resistant 的，我们知道在该协议中，在任何固定 $\mathsf{rand}_\mathcal{A},H,\mathsf{rand}_\mathsf{oracle},\mathsf{rand}_\mathsf{beacon\ 1},...,\mathsf{rand}_\mathsf{beacon\ \ell-1}$ 的情况下 $\mathsf{rand}_\mathsf{beacon\ \ell}$有最多是 $u$ 的共最小熵。 特别地，对于一个在 $(\mathsf{rand}_\mathcal{A},H,\mathsf{rand}_\mathsf{oracle})$ 的可能值上均匀分布的 $A$，和在任何固定 $A$ 的情况下有最多是 $ud$ 的共最小熵的、用来描述 $\mathsf{rand_beacon}$ 的值的随机变量 $B$，有
$$
\mathrm{adv}_{\mathcal{A},P}=\mathrm{Pr}(P(\mathbf{C}_\mathcal{A}(A,B),\mathsf{z}(A,B))=\mathsf{acc})
$$
现在，根据声明 3.1，它服从下式
$$
\mathrm{Pr}_{\mathsf{x}\gets\mathcal{X}}(P(\mathbf{C}_\mathcal{A}(\mathsf{x}),\mathsf{z(x)})=\mathsf{acc})\ge2^{-ud}\cdot \mathrm{adv}_{\mathcal{A},P}
$$
其中 $\mathsf{x}\gets\mathcal{X}$ 是指一个 $\mathsf{x}$ 的均匀选择。

给定 $\mathcal{A}$，我们用以下的属性构造 $\mathcal{B}$。$\mathcal{B}$ 接收随机变量 $\mathbf{C}_S$ 的输出值 $[\mathbf{C}(s)]$。给定 $[\mathbf{C}(s)]$，它产生输出 $\mathsf{z}(\mathsf{x})$，使得对于 $\mathsf{x}$ 有

1.  $\mathsf{x}$ 在 $\mathcal{X}$ 中 (在 $s\in (\mathbb{F}^*_p)^t$ 的随机性和 $\mathcal{B}$ 的随机性上) 是均匀的
2.  $\mathcal{B}$ 没有用 $\mathbf{C}_{\mathcal{A}}(\mathsf{x})=[\mathbf{C}(s)]$来产生一个输出 $\mathsf{z}(\mathsf{x})$ 的 $\mathsf{x}$ 值有密度 $\mathrm{negl}(\lambda)$。

这符合下式
$$
\mathrm{Pr}(P(\mathbf{C}_S,\mathcal{B}(\mathbf{C}_S))=\mathsf{acc})\ge \\
\mathrm{Pr}_{\mathsf{x}\gets\mathcal{X}}(P(\mathbf{C}_\mathcal{A}(\mathsf{x}),\mathsf{z(x)})=\mathsf{acc})-\mathrm{negl}(\lambda)\ge \\
2^{-ud}\cdot \mathrm{adv}_{\mathcal{A},P}-\mathrm{negl}(\lambda)
$$
我们继续来描述 $\mathcal{B}$ 并展示它的输出正如所声明的那样。

我们有 $[\mathbf{C}(s)]=\{[\mathrm{g}(s)]\}_{\mathrm{g\in M_C}}$，其中 $\mathrm{g\in M_C}$ 是 $\mathrm{C}$ 中所有乘法/除法层的所有门的集合。$\mathcal{B}$ 和 $\mathcal{A}$ 像下面这样执行这个协议。我们认为 $\mathcal{B}$ 内部有一个查询 $\mathcal{R}$ 的预言机电路 $\mathcal{B}^*$ 。当 $\mathcal{B}^*$ 对 $\mathcal{R}$ 发出一个新的查询，$\mathcal{B}$ 的响应在 $\mathbb{G}^*_2$ 中均匀分布，否则它的响应和上次响应一致。如果 $\mathcal{B}^*$ 在下面的描述中中止了，那么 $\mathcal{B}$ 对一些任意的字符串 $\mathsf{x}'$ 输出 $\mathsf{z}(\mathsf{x'})$。

$\mathcal{B}^*$ 反过来如下操作 $\mathcal{A}$。

1. $\mathcal{B}^*$ 对 $\mathcal{R}$ 的响应初始化一个包含 “异常” 的空白表格 $T$。

2. 无论何时 $\mathcal{A}$ 对 $\mathcal{R}$ 发起一个查询 $q$，$\mathcal{B}^*$ 检查响应 $\mathcal{R}(q)$ 是否在表格 $T$ 中；如果在的话，它会根据表格内容回应，如果不在的话，则根据 $\mathcal{R}$ 来回应。它对查询 $\mathsf{RB}$ 的回应如下。

3. 对于每个 $\ell\in[1..d]$，它仿真第 $\ell$ 个阶段如下。

   1. 令 $j$ 是阶段 $\ell$ 中诚实参与者的索引。令 $C:=C_{\ell}$。前面提到过 $\mathrm{x}^\ell$ 表示 $C$ 的输入。$\mathcal{B}^*$ 首先通过在先前阶段的记录副本上调用 $\mathcal{A}$ 来执行到参与者 $P_{j-1}$的阶段。

      对于每个 $1\le j'<j$ 使得 $P'_j$ 中止或者输出一个让协议协调员拒绝的无效消息，$\mathcal{B}$ 设置 $\mathrm{x}^\ell_{j'}=(1,...,1)\in(\mathbb{F}^*_p)^{t_\ell}$。否则，对于每个 $x\in\mathrm{x}^\ell_{j'}$，$P_{j'}$ 输出用 $\mathsf{POK}(x,\mathsf{transcript}_{\ell,j'-1})$ 生成的 $[x]_1$ 和 $y\in\mathbb{G}_2$。当把 $\mathcal{A}$ 作为 $\mathcal{B}^*$ 的一个变体时，它使用相同的随机字符串并且和 $\mathcal{B}^*$ 运行一致但是在运行到该点时就停止并输出 $[x]_1,\mathsf{transcript}_{\ell,j-1},y$；和当 $z=[\mathbf{C}(s)]$ 时，令 $\mathtt{X}$ 是从声明 3.5 中得到的提取者。$\mathcal{B}^*$ 计算 $x^*=\mathtt{X}(z,\mathbf{r},\mathsf{rand}_\mathcal{B^*})$，其中 $\mathbf{r}$ 是直到输出 $[x]_1,y$ 时从 $\mathcal{R}$ 到 $\mathcal{B}^*$ 的响应序列。如果 $\mathtt{X}$ 的输出 $x^*$ 不等于 $x$，$\mathcal{B}^*$ 将中止。(这能通过检验 $[x^*]_1=[x]_1$ 来验证)

   2. 如果 $\mathcal{B}^*$ 没有中止，它得到了 $\mathrm{x}^\ell_1,...,\mathrm{x}^\ell_{j-1}$。$\mathcal{B}^*$ 现在选择均匀的 $b\in(\mathbb{F}^*_p)^{t_\ell}$​，并定义
      $$
      \mathrm{x}^\ell_{j}:=\frac{bs^\ell}{\mathrm{x}^\ell_{1}\cdots\mathrm{x}^\ell_{j-1}}
      $$
      因为 $\mathcal{B}^*$ 不知道 $s$，所以它无法计算 $\mathrm{x}^\ell_{j}$。然而，它有 $[s^\ell]$，作为 $[\mathbf{C}(s)]$ 的一部分，其中 $s^\ell$ 是 $s$ 对 $C$ 的输入 $\mathrm{x}^\ell$ 的限制。因此它能计算
      $$
      [\mathrm{x}^\ell_{j}]:=\frac{b\cdot[s^\ell]}{\mathrm{x}^\ell_{1}\cdots\mathrm{x}^\ell_{j-1}}
      $$
      注意，有 $\mathrm{x}^\ell_{1}\cdots\mathrm{x}^\ell_j=bs^\ell$。所以对于每个 $\mathrm{g}\in C$，$\mathcal{B}^*$ 能够计算和广播 $[\mathrm{g}]^\mathbf{j}=M_\mathrm{g}({\mathrm{x}^\ell_{1}\cdots\mathrm{x}^\ell_{j}})\cdot\mathrm{g}_\mathsf{src}=M_\mathrm{g}(b)M_\mathrm{g}(s^\ell)\cdot\mathrm{g}_\mathsf{src}=M_\mathrm{g}(b)\cdot[\mathrm{g}(s)]$。其中 $[\mathrm{g}(s)]$ 是作为 $[\mathbf{C}(s)]$ 的一部分给出的。因此，能够用 4.4 节步骤 1.2 中 $\mathrm{x}^\ell_j$ 的值正确地扮演 $P_j$ 的角色和生成一个有效的消息。

   3. 剩下的就是像 4.4 节步骤 1 中说的对于 $x\in\mathrm{x}^\ell_{j}$ 生成 $\mathsf{POK}([x]_1,\mathsf{transcript}_{\ell,j-1})$。如果 $\mathcal{R}([x]_1,\mathsf{transcript}_{\ell,j-1})$ 已经被 $\mathcal{A}$ 查询了，它将中止。否则，$\mathcal{B}^*$ 选择一个随机的 $r\in\mathbb{F}^*_p$ 并增加查询 $(([x]_1,\mathsf{transcript}_{\ell,j-1}),[r]_2)$ 到异常表格 $T$ 中。它输出 $y:=r\cdot[x]_2$。注意，如果我们有 $\mathcal{R}([x]_1,\mathsf{transcript}_{\ell,j-1})=[r]_2$，那么我们将有 $\mathsf{CheckPOK}([x]_1,\mathsf{transcript}_{\ell,j-1},y)$；所以给定 $H$ 和 $\mathsf{rand}_\mathsf{oracle}$，从 $\mathcal{A}$ 的观点来看这是一个正确的消息。

   4. 现在，$\mathcal{B}^*$ 利用 $\mathcal{A}$ 来完成阶段 $\ell$ 中 $P_{j+1},...,P_N$ 的部分。同样地，对任何 $j+1\le j'\le N$ 使得 $ P_{j'}$ 没有输出一条有效消息时，$\mathrm{x}^\ell_{j'}$ 被设置为向量 $(1,...,1)$。

   5. 跟之前类似，对于任何的 $j+1\le j'\le N$ 使得 $P_{j'}$ 没有广播一条有效消息时，对任何 $x\in\mathrm{x}^\ell_{j'}$，$P_{j'}$ 输出用 $\mathsf{POK}(x,\mathsf{transcript}_{\ell,j'-1})$ 生成的 $[x]_1$ 和 $y\in\mathbb{G}_2$。当把 $\mathcal{A}$ 作为 $\mathcal{B}^*$ 的一个变体时，它使用相同的随机字符串并且和 $\mathcal{B}^*$ 运行一致但是在运行到该点时就停止并输出 $[x]_1,\mathsf{transcript}_{\ell,j-1},y$；和当 $z=[\mathbf{C}(s)]$ 时，令 $\mathtt{X}$ 是从声明 3.5 中得到的提取者。$\mathcal{B}^*$ 计算 $x^*=\mathtt{X}(z,\mathbf{r},\mathsf{rand}_\mathcal{B^*})$，其中 $\mathbf{r}$ 是直到输出 $[x]_1,y$ 时从 $\mathcal{R}$ 到 $\mathcal{B}^*$ 的响应序列。如果 $\mathtt{X}$ 的输出 $x^*$ 不等于 $x$，$\mathcal{B}^*$ 将中止。

   6. 如果 $\mathcal{B}^*$ 没有中止，它就得到了 $\mathrm{x}^\ell_{j+1},...,\mathrm{x}^\ell_{N}$。它定义 $\mathrm{x'}^\ell:=\frac{1}{b\cdot\mathrm{x}^\ell_{j+1}\cdots\mathrm{x}^\ell_{N}}$；并输出 $\mathrm{x'}^\ell$ 作为信标的输出 $\mathsf{RB}(J,t_\ell)$。注意，如果我们到了这里还没有中止，我们有 $\mathrm{x}^\ell_{1}\cdots\mathrm{x}^\ell_N\cdot\mathrm{x'}^\ell=s^\ell$。

4. 最终 $\mathcal{B}^*$ 在协议的最后输出 $\mathcal{A}$ 的输出 $\mathsf{z}$。$\mathbf{z}$

我们接着证明第一个属性 - 我们需要展示在协议中使用的元素 $(\mathsf{rand}_\mathcal{A},H,\mathsf{rand}_\mathsf{oracle},\mathsf{rand}_\mathsf{beacon})$ 是均匀的和相互独立的。

- $\mathsf{rand}_\mathcal{A}$ - $\mathcal{B}^*$ 使用它的随机硬币的一个均匀选择来运行 $\mathcal{A}$，所以 $\mathsf{rand}_\mathcal{A}$ 是均匀分布的。
- $\mathsf{rand}_\mathsf{oracle}$ - $\mathcal{B}$ 均匀地选择 $\mathcal{R}$ 的输出并且独立于其他任何事件。$\mathsf{rand}_\mathsf{oracle}$ 的其他元素是在步骤 3.3 中选择的元素 $[r]_2$，它们在 $\mathbb{G}^*_2$ 上是均匀的并且独立于其他变量。
- $H$ - 每个层 $C_\ell$ 的诚实输入 $\mathrm{x}^\ell_j$ 被选作 $\frac{b\cdot s^\ell}{a}$，其中 $a$ 是在同一层中诚实参与者之前被 $\mathcal{A}$ 控制的参与者输入的积。$b$ 和 $s^\ell$ 都在 $(\mathbb{F}^*_p)^{t_\ell}$ 上是均匀的；且和彼此、$a$ 还有其他层的相同变量独立。因此 $H$ 是均匀的且和之前的变量独立。
- $\mathsf{rand}_\mathsf{beacon}$ - 层 $C=C_\ell$ 中 $\mathsf{rand}_\mathsf{beacon}$ 的部分有着 $\frac{1}{a\cdot b}$ 的形式，其中 $a$ 包含了在诚实参与者之后被 $\mathcal{A}$ 控制的参与者的输入。$b$ 出现的仅有的另一个地方是在 $\mathrm{x}^\ell_j$ 里。但是即使固定住 $\mathrm{x}^\ell_j$ 使没有了 $b$，因此 $\mathsf{rand}_\mathsf{beacon}$ 这部分在层 $\ell$ 中是均匀的。

为了证明第二个属性，我们注意到协议如前描述的输出值 $\mathsf{x}$ 将不会是 $[\mathbf{C}(s)]$，而是那些在步骤 3.1、3.5 或 3.3 中引起中止的。根据声明 3.5，在步骤 3.1 和 3.5 中的中止是由一个 $\mathsf{x}\in\mathcal{X}$ 的 $\mathrm{negl}(\lambda)$ 分数引发的；3.3 中的中止只在 $\mathcal{A}$ 选择在一个大小至少为 $|\mathbb{G}^*_2|$ 的域中后来均匀选择的一个输出中预先查询 $\mathcal{R}$ 时引发，因此只由一个 $\mathsf{x}\in\mathcal{X}$ 的 $\mathrm{negl}(\lambda)$ 分数引发。



## 6	减少 Groth 的 CRS 的深度

在本章中，我们假设读者对二次算术程序 (Quadratic Arithmetic Programs) [23] 和 Groth 的工作 [27] 已经熟悉。正如在 [27] 中一样，我们先描述非交互式线性证明 (NILP)，zk-SNARK 就是在它之上建立的。

**扩展的 Groth CRS：**令 $\{u_i,v_i,w_i\}_{i\in[0..m]} \cup \{t\}$ 是在 $\mathbb{F}_p$ 上的 $n$ 阶 QAP 的多项式，其中 $t$ 是 QAP 的 $n$ 阶目标多项式，且其他多项式的阶数要小于 $n$。假设 $1,...,\ell<m$ 是公共输入的索引。

对于 $\alpha,\beta,\delta,x\in\mathbb{F}_p$，定义 $Groth(\alpha,\beta,\delta,x)$ 是下面的一组元素：
$$
\beta,\delta,\{x^i\}_{i\in[0..2n-2]},\{\alpha x^i\}_{i\in[0..n-1]},\{\beta x^i\}_{i\in[1..n-1]},\\
\{x^i\cdot t(x)/\delta\}_{i\in[0..n-2]},\\
\{\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\delta}\}_{i\in[\ell+1..m]}
$$
跟 [27] 对比，额外的元素是 $\{x^i\}_{i\in[n..2n-2]},\{\alpha x^i\}_{i\in[1..n-1]},\{\beta x^i\}_{i\in[1..n-1]}$​。另一方面，在 [27] 的 CRS 中出现的元素 
$$
\{\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\gamma}\}_{i\in[0..\ell]}, \gamma
$$
在这里消失了；验证者需要它们在那里来计算
$$
\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))
$$
而我们可以使用上述增加了元素的 CRS 的线性组合来计算。

根据 4.1 节中电路深度的定义，我们声明能够用一个深度为二的电路来计算 $Groth$：

- $C_1$：该层输入是 $\mathrm{x^1}=\{x,\alpha,\beta\}$。该层输出的 $\{x^i\}_{i\in[0..2n-2]},\{\alpha x^i\}_{i\in[0..n-1]},\{\beta x^i\}_{i\in[0..n-1]}$ 都是 $\mathrm{x^1}$ 中的输入的积。
- $L_1$：我们用 $\{x^i\}_{i\in[0..2n-2]}$ 的线性组合来计算 $\{x^i\cdot t(x)\}_{i\in[0..n-2]}$，因为 $t$ 的阶数是 $n$。我们也可以计算 $(\beta u_i(x)+\alpha v_i(x)+w_i(x))_{i\in[0..m]}$ 因为它是第一层元素的线性组合。
- $C_2$：该层的输入是 $\mathrm{x^2}=\{\delta\}$。我们计算 $\delta,\{x^i\cdot t(x)/\delta\}_{i\in[0..n-2]},\{\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\delta}\}_{i\in[\ell+1..m]}$。

**Groth 中的证明者和验证者：**固定公共输入 $a_1,...,a_\ell$。证明者随机选择 $r,s\in\mathbb{F}_p$ 并从 CRS 和 他的见证 (witness) $a_{\ell+1},...,a_m$ 计算元素
$$
A=\alpha+\sum_{i=0}^m a_iu_i(x)+r\delta,\\
B=\beta+\sum_i=0^m b_iv_i(x)+s\delta,\\
C=\frac{\sum_{i=\ell+1}^m a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))+h(x)t(x)}{\delta}+As+Br-rs\delta
$$
而验证者使用 $A,B,C$ 来检查
$$
A\cdot B=\alpha\cdot\beta+\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))+C\cdot\delta
$$
**证明知识可靠性：**从 [27] 中足够证明我们能够从给出的 CRS 元素线性组合 $A,B,C$​ 来为 QAP 抽取出一个见证。也就是说，我们假设有
$$
A=A_\alpha(x)\alpha+A_\beta(x)\beta+A_\delta\delta+A(x)+\sum_{i=\ell+1}^m \frac{A_i\cdot(\beta u_i(x)+\alpha v_i(x)+w_i(x))}{\delta}+A_h(x)\frac{t(x)}{\delta}
$$
其中，$A_\alpha,A_\beta$ 是已知的阶数至多为 $n-1$ 的多项式，$A$ 是阶数至多为 $2n-2$ 的多项式，$A_h$ 的阶数至多为 $n-2$，$A_i,\{A_i\}_{i\in[\ell+1..m]},A_\delta$ 是已知的域元素。$B$ 和 $C$​ 的定义也相似。我们假设对于给出的这些多项式和常数，有
$$
A\cdot B\equiv \alpha\cdot\beta+\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))+C\cdot\delta
$$
是一个关于 $x,\alpha,\beta,\delta$ 的有理函数 (rational functions)。对于一个给定的 $C$，让我们用 $C^*$ 来表示上述等式的右边；例如
$$
C^*=\alpha\cdot\beta+\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))+C\cdot\delta
$$
并用 $C_0$ 来表示“$C^*$ 中没有 $C$ 的部分”；例如
$$
C_0=\alpha\cdot\beta+\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))
$$
所以，这里我们认为 $A\cdot B\equiv C^*\equiv C_0+C\cdot\delta$ 是 $x,\alpha,\beta,\delta$ 的有理函数。

从现在开始，当我们讨论单项式时，我们其实是指用 $x,\alpha,\beta,\delta$ 表示的没有共同因子的两个多项式的商；例如 $\frac{\alpha}{\delta}$。对于一个多项式 $M$，让我们用 $M\in A$ 来表示 $M$ 在 $A$ 中有一个非零系数；例如，当把 $A$ 写作由 $x,\alpha,\beta,\delta$ 表示的单项式的 (唯一) 线性组合时，$M$ 完整出现在其中且没有非零系数。 对 $B,C,A\cdot B,C_0,C^*$ 采用相同的表示法。

当我们说一个单项式*在 CRS 中*，我们是指当我们把一个 CRS $groth(\alpha,\beta,\delta,x)$ 中的元素写作单项式的组合时，它带有非零系数出现在这个元素中。

我们想要展示的是我们新加入到 CRS 中的单项式 - $\{x^i\}_{i\in[n..2n-2]},\{\alpha x^i\}_{i\in[1..n-1]},\{\beta x^i\}_{i\in[1..n-1]}$ 没有在 $A,B,C$ 中使用；这暗示了使用 [27] 的正确性，因为这证明了对于给出的作为原始 CRS 元素线性组合的 $A,B,C$ 来说验证是成立的，能够从中抽取一个见证。

因为 $\alpha\beta\in A\cdot B$，所有我们必须有 $\alpha\in A,\beta\in B$ 或者 $\beta\in A,\alpha\in B$。不失一般性 (w.l.g., without loss of generality)，我们假设是前者。假设对于某些 $i\ge0$ 有 $\beta x^i\in A$，且令其中的 $i$ 最大化。令 $j\ge0$ 最大化且有 $\beta x^j\in B$。令 $k:=i+j$。

那么有 $\beta^2 x^k\in A\cdot B\equiv C^*$。这表明要么 $\beta^2 x^k\in C$ - 但是这个单项式对于任何整数 $k$ 都不存在于 CRS 中；要么 $\beta^2 x^k\in C_0$ 是错误的。所以这样的 $i$ 不存在。

一个类似的论点表明对于任何整数 $i$ 有 $\alpha x^i\notin B$。

现在我们令  $i\ge0$ 最大化且有 $\alpha x^i\in A$，令 $j\ge0$ 最大化且有 $\beta x^j\in B$。那么有 $\alpha\beta x^{i+j}\in A\cdot B\equiv C^*$。既然对于任何 $k$，$\alpha\beta x^k/\delta$ 都不在 CRS 中，且只有 $k=0$ 时 $\alpha\beta x^k\in C_0$，我们有 $i+j=0$，所以 $i,j=0$。

现在假设 $\alpha x^i\in C$ - 那么有 $\alpha x^i\beta\in A\cdot B$，这表明 $\alpha x^i$ 在 $A$ 或 $B$ 中；我们已经知道了这只有在 $i=0$ 时才可能。对 $\beta x^i\in C$ 也是类似的。总之，我们已经展示了这些新的项 $\{\alpha x^i,\beta x^i\}_{i\in[1..n-1]}$ 不会出现在证明中。

现在，令 $i$ 最大化且 $x^i\in A$。那么有 $\beta x^i\in C^*$，即

-  $\beta x^i/\delta\in C$，这只有在 $i\le n-1$ 时才成立，因为这样的单项式只可能在 CRS 元素 $\{\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\delta}\}_{i\in[\ell+1..m]}$ 中出现，它们最高都只包含 $x$ 的 $n-1$ 次方。或者
-  $\beta x^i\in C_0$，这只可能在 $i\le n-1$ 时作为 $C_0$ 中的元素 $\sum_{i=0}^\ell a_i(\beta u_i(x)+\alpha v_i(x)+w_i(x))$ 的一部分而成立。

类似地，令 $j$ 最大化且 $x^j\in B$。那么 $\alpha x^j\in C^*$ 意味着要么 $\alpha x^j/\delta\in C$，要么 $\alpha x^j\in C_0$，两者都只有在 $j\le n-1$ 时才能成立。

如果 $x^i\in C$，那么 $x^i\beta\in A\cdot B$，意味着 $x^i\in A$ 或者 $x^i\in B$，所以 $i<n$。因此，新的项 $\{x^i\}_{i\in[n..2n-1]}$ 没有在证明中被使用。



## 7	对 Groth 的 zk-SNARK 的多方计算

我们现在实例化第 4 章的协议，以获得一个用于计算 zk-SNARK 的 CRS 的协议，对应于第 6 章中描述的 NILP 的 CRS。

输出将具有以下形式
$$
\{[x^i]\}_{i\in[0..n-1]},\{[x^i]_1\}_{i\in[n..2n-2]},\{[\alpha x^i]_1\}_{i\in[0..n-1]},\\
[\beta],\{[\beta x^i]_1\}_{i\in[1..n-1]},\{[x^i\cdot t(x)/\delta]\}_{i\in[0..n-2]},\\
\{[\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\delta}]_1\}_{i\in[l+1..m]}
$$

请注意到一些输出值只在 $\mathbb{G}_1$ 群中给出，然而第 4 章的协议描述中在两个群中给出了所有的输出值。可以直观地看出，如果这种情况仅适用于后来用作输入的输出，也仅适用于仅在 $\mathbb{G}_1$ 中给出的其他输出，安全证明也采用同样的方法。

在下面的协议中，如果 $M$ 是一个我们想要在群 $\mathbb{G}_1, \mathbb{G}_2$ 或者 $\mathrm{G}$ 中计算的输出值，且 $j\in [N]$，我们将把它表示为 $[M]^j$，即在参与者 $P_1,...,P_j$ 作出了他们的贡献以后的 “部分 $M$”。$[M]^0$ 将被设置成某些初始值作为协议描述的一部分。我们假设 $\mathrm{g}$ 是公开的。

### 7.1	第1轮：‘Powers of $\tau$’

我们需要计算
$$
\mathrm{M}_1=\left\{\begin{array}{l} \{[x^i]\}_{i\in[0..n-1]},\{[x^i]_1\}_{i\in[n..2n-2]},\\ 
\{[\alpha x^i]_1\}_{i\in[0..n-1]},[\beta],[\delta],\{[\beta x^i]_1\}_{i\in[1..n-1]} \end{array}\right\}
$$

**初始化：**

1. $[x^i]^\mathbf{0}:=\mathbf{g}, i\in[1..n-1]$。
2. $[x^i]^\mathbf{0}:=g_1, i\in[n..2n-2]$。
3. $[\alpha x^i]^\mathbf{0}:=g_1, i\in[0..n-1]$。
4. $[\beta]^\mathbf{0}:=\mathbf{g}$。
5. $[\beta x^i]^\mathbf{0}:=g_1, i\in[1..n-1]$。

**计算：**对每个 $j\in [N], P_j$ 输出：

1. $[\alpha_j]_1,[\beta_j]_1,[x_j]_1$。
2. $y_{\alpha,j}:=\mathrm{POK}(\alpha_j,\mathrm{transcript}_{1,j-1})$。
3. $y_{\beta,j}:=\mathrm{POK}(\beta_j,\mathrm{transcript}_{1,j-1})$。
4. $y_{x,j}:=\mathrm{POK}(x_j,\mathrm{transcript}_{1,j-1})$。
5. 对每一个 $i\in[1..2n-2],[x^i]^\mathbf{j}:=x^i_j\cdot[x^i]^\mathbf{j-1}$。
6. 对每一个 $i\in[0..n-1],[\alpha x^i]^\mathbf{j}:=\alpha_j x^i_j\cdot[\alpha x^i]^\mathbf{j-1}$。
7. 对每一个 $i\in[0..n-1],[\beta x^i]^\mathbf{j}:=\alpha_j x^i_j\cdot[\beta x^i]^\mathbf{j-1}$。

令 $J-1$ 作为 $P_N$ 在此阶段中应该广播消息的时隙。令 $(x',\alpha',\beta'):=\mathsf{RB}(J,3)$。我们定义：

1. $[x^i]:=x'^i\cdot[x^i]^\mathbf{N}, i\in[1..2n-2]$。
2. $[\alpha x^i]:=\alpha' x'^i\cdot[\alpha x^i]^\mathbf{N},i\in[0..n-1]$。
3. $[\beta x^i]:=\beta' x'^i\cdot[\beta x^i]^\mathbf{N},i\in[0..n-1]$。

**验证：**协议验证者对每个 $j\in [N]$ 计算 
$$
r_{\alpha,j}:=\mathcal{R}([\alpha_j]_1,\mathsf{transcript}_{1,j-1}),\\
r_{\beta,j}:=\mathcal{R}([\beta_j]_1,\mathsf{transcript}_{1,j-1}),\\
r_{x,j}:=\mathcal{R}([x_j]_1,\mathsf{transcript}_{1,j-1})
$$
 并检查以下这些：

1. $\mathsf{CheckPOK}([\alpha_j]_1,\mathsf{transcript}_{1,j-1},y_{\alpha,j})$，
2. $\mathsf{CheckPOK}([\beta_j]_1,\mathsf{transcript}_{1,j-1},y_{\beta,j})$，
3. $\mathsf{CheckPOK}([x_j]_1,\mathsf{transcript}_{1,j-1},y_{x,j})$，
4. $\mathsf{consistent}([\alpha]^\mathbf{j-1}-[\alpha]^\mathbf{j};(r_{\alpha,j},y_{\alpha,j}))$，
5. $\mathsf{consistent}([\beta]^\mathbf{j-1}-[\beta]^\mathbf{j};(r_{\beta,j},y_{\beta,j}))$，
6. $\mathsf{consistent}([x]^\mathbf{j-1}-[x]^\mathbf{j};(r_{x,j},y_{x,j}))$，
7. 对每一个 $i\in[1..2n-2],\mathsf{consistent}([x^{i-1}]^\mathbf{j}-[x^i]^\mathbf{j};[x]^\mathbf{j})$，
8. 对每一个 $i\in[1..n-1],\mathsf{consistent}({[x^{i}]^\mathbf{j}}_1-[\alpha x^i]^\mathbf{j};[\alpha]^\mathbf{j})$，
9. 对每一个 $i\in[1..n-1],\mathsf{consistent}({[x^{i}]^\mathbf{j}}_1-[\beta x^i]^\mathbf{j};[\beta]^\mathbf{j})$。

### 7.2	在阶段之间的线性组合

对 $i\in[1..2n-2]$，我们把计算的 $H'_i:=[t(x)x^i]_1$ 作为 $\{[x^i]_1\}_{i\in[0..2n-2]}$ 的线性组合。

令 $w\in\mathbb{F}_p$ 是一个 $n=2^t$ 次本原单位根；典型地，$n$ 是二的所有指数中最小的大于等于电路尺寸的。

对 $i\in[1..n]$，我们定义 $L_i$ 是在点集 $\{\omega^i\}_{i\in[1..n]}$ 上的第 $i$ 个拉格朗日多项式 (Lagrange polynomial)。也就是说，$L_i$ 是对于 $j\in[1..n]\backslash\{i\}$ 满足 $L_i(\omega^i)=1$ 和 $L_i(\omega^j)=0$ 的阶数小于 $n$ 的唯一多项式。对于 $x\in\mathbb{F}^*_p$，我们用 $\mathrm{LAG}_x\in\mathbf{G}^n$ 来表示向量
$$
\mathrm{LAG}_x:=([L_i(x)])_{i\in[1..n]}
$$
正如在 [16] 的第 3.3 节中描述的， $\mathrm{LAG}_x$ 可以在一个对 $\{[x^i]\}_{i\in[0..n-1]}$ 使用 $O(n\mathrm{log}n)$ 个群操作的快速傅里叶变换 (FFT) 中计算得出。相似地，因为 FFT 是线性的，在 $\mathbb{G}_1$ 群中对 $\{[\alpha x^i]_1\}_{i\in[0..n-1]}$ 和 $\{[\beta x^i]_1\}_{i\in[0..n-1]}$ 使用相同的操作，我们就能得到 $(\alpha\cdot\mathrm{LAG}_x)_1$ 和 $(\beta\cdot\mathrm{LAG}_x)_1$。

因为每个 QAP 多项式 $\{u_i,v_i,w_i\}_{i\in[0..m]}$ 通常是最多三个不同 $L_i$ 的线性组合，我们现在能使用 $O(m)$ 个群操作来计算元素 $\{[\beta u_i(x)]_1\}_{i\in[0..m]}$，$\{[\alpha v_i(x)]_1\}_{i\in[0..m]}$ 和 $\{[w_i(x)]_1\}_{i\in[0..m]}$。

最后，对于 $i\in[\ell+1..m]$ 我们计算元素的线性组合 $K'_i:=[\beta u_i(x)+\alpha v_i(x)+w_i(x)]_1$。

我们也要输出元素 ${\{[u_i(x)]_1\}}_{i\in[0..m]}$，${\{[v_i(x)]_2\}}_{i\in[0..m]}$ 作为 $\mathrm{LAG}_x$ 的线性组合 (是为了更快的验证计算。不难看出对 CRS 元素增加线性组合并不会改变其安全性)。

### 7.3	第二轮

对于 $i\in[\ell+1..m]$​， 有
$$
K_i:=\frac{\beta u_i(x)+\alpha v_i(x)+w_i(x)} {\delta}
$$
对于 $i\in[0..n-2]$​， 有
$$
H_i:= \frac{t(x)x^i}{\delta}
$$
我们需要计算
$$
M_2=\left\{\begin{array}{l} [\delta],{\{[K_i]_1\}}_{i\in[l+1..m]},{\{[H_i]_1\}}_{i\in[0..n-2]}\end{array}\right\}
$$
**初始化：**

1. $[K_i]^{\mathbf{0}}:=K'_i,i\in[\ell+1..m]$。
2. $[H_i]^{\mathbf{0}}:=H'_i,i\in[\ell+1..m]$。
3. $[\delta]^{\mathbf{0}}:=\mathbf{g}$。

**计算：**对每个 $j\in [N], P_j$ 输出：

1. $[\delta_j]_1$。
2. $y_{\delta,j}:=\mathsf{POK}(\delta_j,\mathsf{transcript}_{2,j-1})$ 。
3. $[\delta]^\mathbf{j}:=[\delta]^\mathbf{j-1}/\delta_j$ 。
4. 对每一个 $i\in[\ell+1..m],[K_i]^\mathbf{j}:=([K_i]^\mathbf{j-1})/\delta_j$ 。
5. 对每一个 $i\in[0..n-2],[H_i]^\mathbf{j}:=([H_i]^\mathbf{j-1})/\delta_j$ 。

最后，令 $J-1$ 作为 $P_N$ 在此阶段中应该广播消息的时隙。令 $\delta':=\mathsf{RB}(J,1)$。我们定义：

1. $[\delta]:=[\delta]^\mathbf{N}/\delta'$ 。
2. ${[K_i]}_1:=[K_i]^\mathbf{N}/\delta'$ 。
3. ${[H_i]}_1:=[H_i]^\mathbf{N}/\delta'$ 。

**验证：**协议验证者对每个 $j\in [N]$ 计算 
$$
r_{\delta,j}:=\mathcal{R}([\delta_j]_1,\mathsf{transcript}_{2,j-1}),
$$
 并检查以下这些：

1. $\mathsf{CheckPOK}({[\delta_j]}_1,\mathsf{transcript}_{2,j-1},y_{\delta,j})$，
2. 对于 $j\in [N]$，$\mathsf{consistent}([\delta]^\mathbf{j-1}-[\delta]^\mathbf{j};(r_{\delta,j},y_{\delta,j}))$，
3. 对每一个 $i\in[\ell+1..m],j\in[N],\mathsf{consistent}([K_i]^\mathbf{j}-[K_i]^\mathbf{j-1};[\delta_\mathbf{j}])$，
4. 对每一个 $i\in[0..n-2],j\in[N],\mathsf{consistent}([H_i]^\mathbf{j}-[H_i]^\mathbf{j-1};[\delta_\mathrm{j}])$。



## 8	BLS12-381

在 zk-SNARK 软件中最常用的支持配对的椭圆曲线构造是在 [13] 中设计的有着 254 位基础域和群阶的 Barreto-Naehrig (BN) 构造。该构造对于高效的多项式评估给 $\mathbb{F}_p$ 装备了一个很大的 $2^n$ 单位根。即使该构造最初的目标是 128 位安全级别，最近对于 Number Field Sieve 算法 [30] 的优化已经降低了它实际的安全性。

后续的分析 [34] 推荐为了保证 128 位安全性而带有内嵌的 $k=12$、有着大约 384 位基础域的 BN 曲线和 Barreto-Lynn-Scott (BLS) 曲线。因此 BN 曲线不适合我们的目标，因为这些更大的基础域伴随着相似的更大的群阶，实质上增加了多重指数计算和快速傅里叶变换的开销，并损害了使用 $\mathbb{F}_p$ 来编码关键材料的协议的可用性。相反地，有着 384 位基础域的 BLS12 曲线，只有 256 位的群阶，使得它们非常适合在 zk-SNARKs 中使用。在更保守的环境中，推荐在 [6] 中提出的更大的构造。

有 $k=12$ 的 BLS 曲线是由一个整数 $x$ 参数化的。现有的 BN 曲线有 $2^{28}|p-1$ 来保证一个 $2^{28}$ 单位根是可用的。我们通过保证 $2^{14}|x$ 来达到相同的目标。我们选定小于 $2^{255}$ 的质数 $p$ 来适应高效的近似算法和约简。根据 [5] 的建议，我们需要有效的扩张域和扭曲同构。另外，我们需要 $x$ 有小的 Hamming 权重来获得最好的配对效率。

有着最小 Hamming 权重的满足我们需求的最大构造是 $x=-2^{63}-2^{62}-2^{60}-2^{57}-2^{48}-2^{16}$，我们命名它为 **BLS12-381**。如 [21] 中所述，该曲线存在于一个曲线集合的子集中，它们刚刚被确定了曲线参数。我们用 Rust 语言提供一个该曲线的实现 [2]。

## 9	实现和实验

在本章中，我们评估我们对 MMORPG 的实现。我们对 MMORPG 和配对库的实现都是用的 Rust 语言。所有对于阶段 1 和阶段 2 的基准测试是在一台 Intel(R) Core(TM) i7-3770S CPU @ 3.10GHz 带有 32GB 内存的运行 Arch Linux 的机器上完成的。

因为我们协议的性能和参与者数量是无关的，所以我们的实验设置极其简单。我们只需要衡量单个用户在每个阶段的表现。通过 zk-SNARKs 证明的陈述是用一个算术电路来表示的。用乘法门的数量来表示的电路大小取决于被证明的陈述的复杂性。我们的实验设置包含在三种电路规模 $2^{10},2^{17},2^{21}$ 个门下运行 MMORPG 并测量运行时间和带宽。$2^{21}$ 是使用 [16] 公开生成的最大电路的规模并且它对应于 60 次 SHA256 的调用。$2^{17}$ 对应于下一代 Zcash [17] 协议的规模，$2^{10}$ 是非常小的电路。测试性能结果在图 7.1 中给出。每一个阶段的带宽和所选的电路规模在表 1 中给出。(图表见原文)

为了完整起见，我们还分析了协调员在阶段之间的计算。这个步骤开支很大。我们强调，这个计算不包含秘密数据，并且只需要完成一次。在现实中，可以租用一个大的 AWS EC2 实例来完成这个计算。

这些结果展示了这个协议是实用的。一个用户只需要花费 15 分钟来做一个计算，然后就不需要继续参与了。着意味着参与者只需要很小的投入，并且不需要保持数小时或数天的高度安全状态。此外，它对在现实世界中执行的 [16] 的每个用户计算时间有着 X 的提升。我们还要强调的是这不是采用一条新曲线的结果，因为那条曲线有着更高的计算复杂度，并且若采用相同的实现语言，要比在 [16] 中的 BN128 慢。这反而是由在协议移除了预先承诺阶段及其引发的空闲时间、和软件优化提升实际计算时间共同作用的结果。

## 致谢

感谢 Paulo Barreto 关于 BLS12-381 椭圆曲线的有用反馈。感谢 Daniel Benarroch、Daira Hopwood 和 Antoine Rondelet 的有用评论。感谢 S&P 2018 的匿名审稿人提出的评论。

## 参考

[1] Joseph bonneau - personal communcation.

[2] Pairing library. url=https://github.com/ebfull/pairing.

[3] Zcash. url=https://z.cash.

[4] Scott Ames, Carmit Hazay, Yuval Ishai, and Muthuramakrishnan Venkitasubramaniam. Ligero: Lightweight sublinear arguments without a trusted setup. In *Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security, CCS 2017, Dallas, TX, USA, October 30 - November 03, 2017*, pages 2087-2104, 2017.

[5] Diego F. Aranha, Laura Fuentes-Castaneda, Edward Knapp, Alfred Menezes, and Francisco Rodriguez-Henriquez. Implementing pairings at the 192-bit security level. Cryptology ePrint Archive, Report 2012/232, 2012. http://eprint.iacr.org/2012/232.

[6] Razvan Barbulescu and Sylvain Duquesne. Updating key size estimations for pairings. Cryptology ePrint Archive, Report 2017/334, 2017. http://eprint.iacr.org/2017/334.

[7] Paulo S. L. M. Barreto, Ben Lynn, and Michael Scott. Constructing elliptic curves with prescribed embedding degrees. Cryptology ePrint Archive, Report 2002/088, 2002. http://eprint.iacr.org/2002/088.

[8] Paulo S. L. M. Barreto and Michael Naehrig. Pairing-friendly elliptic curves of prime order. Cryptology ePrint Archive, Report 2005/133, 2005. http://eprint.iacr.org/2005/133.

[9] E. Ben-Sasson, A. Chiesa, M. Green, E. Tromer, and M. Virza. Secure sampling of public parameters for succinct zero knowledge proofs. In *2015 IEEE Symposium on Security and Privacy, SP 2015, San Jose, CA, USA, May 17-21, 2015*, pages 287-304, 2015.

[10] Eli Ben-Sasson, Iddo Bentov, Alessandro Chiesa, Ariel Gabizon, Daniel Genkin, Matan Hamilis, Evgenya Pergament, Michael Riabzev, Mark Silberstein, Eran Tromer, and Madars Virza. Computational in- tegrity with a public random string from quasi-linear pcps. In *Advances in Cryptology - EUROCRYPT 2017 - 36th Annual International Conference on the Theory and Applications of Cryptographic Tech- niques, Paris, France, April 30 - May 4, 2017,  proceedings, Part III*, pages 551-579, 2017.

[11] Eli Ben-Sasson, Alessandro Chiesa, Christina Garman, Matthew Green, Ian Miers, Eran Tromer, and Madars Virza. Zerocash: Decentralized anonymous payments from bitcoin. In *2014 IEEE Symposium on Security and Privacy, SP 2014, Berkeley, CA, USA, May 18-21, 2014*, pages 459-474, 2014.

[12] Eli Ben-Sasson, Alessandro Chiesa, Daniel Genkin, Eran Tromer, and Madars Virza. SNARKs for C: verifying program executions succinctly and in zero knowledge. In *Proceedings of the 33rd Annual International Cryptology Conference*, CRYPTO '13, pages 90-108, 2013.

[13] Eli Ben-Sasson, Alessandro Chiesa, Daniel Genkin, Eran Tromer, and Madars Virza. Snarks for c: Verifying program executions succinctly and in zero knowledge. Cryptology ePrint Archive, Report 2013/507, 2013. http://eprint.iacr.org/2013/507.

[14] Bryan Bishop. Review of bitcoin scaling proposals. In *Scaling Bitcoin Workshop Phase*, volume 1, 2015.

[15] Nir Bitansky, Alessandro Chiesa, Yuval Ishai, Rafail Ostrovsky, and Omer Paneth. Succinct non- interactive arguments via linear interactive proofs. In *Proceedings of the 10th Theory of Cryptography Conference*, TCC '13, pages 315-333, 2013.

[16] S. Bowe, A. Gabizon, and M. D. Green. A multi-party protocol for constructing the public parameters of the pinocchio zk-snark. *IACR Cryptology ePrint Archive*, 2017:602, 2017.

[17] Sean Bowe. Cultivating sapling: Faster zk-snarks. https://z.cash/blog/cultivating-sapling-faster-zksnarks.html, September 2017.

[18] Benedikt Bunz, Steven Goldfeder, and Joseph Bonneau. Proofs-of-delay and randomness beacons in ethereum. In *S&B '17: Proceedings of the 1st IEEE Security & Privacy on the Blockchain Workshop*, April 2017.

[19] Vitalik Buterin and Joseph Poon. Plasma: Scalable autonomous smart contracts. http://plasma.io/plasma.pdf, August 2017.

[20] Alessandro Chiesa, Matthew Green, Jingcheng Liu, Peihan Miao, Ian Miers, and Pratyush Mishra. Decentralized anonymous micropayments. In *Advances in Cryptology - EUROCRYPT 2017 - 36th Annual International Conference on the Theory and Applications of Cryptographic Techniques, Paris, France, April 30 - May 4, 2017, Proceedings* Part II, pages 609-642, 2017.

[21] Craig Costello, Kristin Lauter, and Michael Naehrig. Attractive subfamilies of bls curves for implementing high-security pairings. Cryptology ePrint Archive, Report 2011/465, 2011. http://eprint.iacr.org/2011/465.

[22] Georg Fuchsbauer. Subversion-zero-knowledge snarks. Cryptology ePrint Archive, Report 2017/587, 2017. http://eprint.iacr.org/2017/587.

[23] R. Gennaro, C. Gentry, B. Parno, and M. Raykova. Quadratic span programs and succinct NIZKs without PCPs. In *Advances in Cryptology - EUROCRYPT 2013, 32nd Annual International Conference on the Theory and Applications of Cryptographic Techniques, Athens, Greece, May 26-30, 2013. Proceedings*, pages 626-645, 2013.

[24] Rosario Gennaro, Craig Gentry, Bryan Parno, and Mariana Raykova. Quadratic span programs and succinct NIZKs without PCPs. In *Proceedings of the 32nd Annual International Conference on Theory and Application of Cryptographic Techniques*, EUROCRYPT '13, pages 626-645, 2013.

[25] Yossi Gilad, Rotem Hemo, Silvio Micali, Georgios Vlachos, and Nickolai Zeldovich. Algorand: Scaling byzantine agreements for cryptocurrencies. In *Proceedings of the 26th Symposium on Operating Systems Principles, Shanghai, China, October 28-31, 2017*, pages 51-68, 2017.

[26] Sha Goldwasser, Yael Kalai, Raluca Ada Popa, Vinod Vaikuntanathan, , and Nickolai Zeldovich. How to run turing machines on encrypted data. Cryptology ePrint Archive, Report 2013/229, 2013. https://eprint.iacr.org/2013/229.

[27] J. Groth. On the size of pairing-based non-interactive arguments. In *Advances in Cryptology - EUROCRYPT 2016 - 35th Annual International Conference on the Theory and Applications of Cryptographic Techniques, Vienna, Austria, May 8-12, 2016, Proceedings, Part II*, pages 305-326, 2016.

[28] Jens Groth. Short pairing-based non-interactive zero-knowledge arguments. In *Proceedings of the 16th International Conference on the Theory and Application of Cryptology and Information Security*, ASIACRYPT '10, pages 321-340, 2010.

[29] Joe Kilian. A note on efficient zero-knowledge proofs and arguments (extended abstract). In *Proceedings of the 24th Annual ACM Symposium on Theory of Computing, May 4-6, 1992, Victoria, British Columbia, Canada*, pages 723-732, 1992.

[30] Taechan Kim and Razvan Barbulescu. Extended tower number eld sieve: A new complexity for the medium prime case. Cryptology ePrint Archive, Report 2015/1027, 2015. http://eprint.iacr.org/2015/1027.

[31] Ahmed Kosba, Andrew Miller, Elaine Shi, Zikai Wen, and Charalampos Papamanthou. Hawk: The blockchain model of cryptography and privacy-preserving smart contracts. In *Security and Privacy (SP), 2016 IEEE Symposium on*, pages 839-858. IEEE, 2016.

[32] Helger Lipmaa. Progression-free sets and sublinear pairing-based non-interactive zero-knowledge arguments. In *Proceedings of the 9th Theory of Cryptography Conference on Theory of Cryptography*, TCC '12, pages 169189, 2012.

[33] Helger Lipmaa. Succinct non-interactive zero knowledge arguments from span programs and linear error-correcting codes. In *Proceedings of the 19th International Conference on the Theory and Application of Cryptology and Information Security*, ASIACRYPT '13, pages 41-60, 2013.

[34] Alfred Menezes, Palash Sarkar, and Shashank Singh. Challenges with assessing the impact of nfs
advances on the security of pairing-based cryptography. Cryptology ePrint Archive, Report 2016/1102, 2016. http://eprint.iacr.org/2016/1102.

[35] Silvio Micali. Computationally sound proofs. *SIAM J. Comput.*, 30(4):1253-1298, 2000.

[36] Bryan Parno, Craig Gentry, Jon Howell, and Mariana Raykova. Pinocchio: nearly practical verifiable computation. In *Proceedings of the 34th IEEE Symposium on Security and Privacy*, Oakland '13, pages 238252, 2013.

[37] Morgen E. Peck. The crazy security behind the birth of zcash, the inside story. https://spectrum.ieee.org/tech-talk/computing/networks/the-crazy-security-behind-the-birth-of-zcash ,December 2016.

[38] Ethereum Team. Byzantium hf announcement. https://blog.ethereum.org/2017/10/12/byzantium-hf-announcement/, October 2017.





