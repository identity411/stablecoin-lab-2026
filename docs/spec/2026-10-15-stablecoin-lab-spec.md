# 法币抵押型稳定币实验 —— 作业规范

**Fiat-Collateralized Stablecoin Lab — Assignment Specification**

| 项目 | 内容 |
|---|---|
| 规范英文名 | Fiat-Collateralized Stablecoin Lab — Assignment Specification |
| 规范编号 | SPEC-2026-10-15-STABLE-COIN-LAB |
| 版本 | v1.1 |
| 发布日期 | 2026-10-15 |
| 适用课程 | COMP7810 · Lab 1「How to Create a Stablecoin」 |
| 仓库 | `stablecoin-lab-2026` |
| 上游材料 | `README.md`、`EXERCISES.md`、`STUDENT-QUESTIONS.md`、`Makefile` |
| 状态 | 生效（本文档为作业要求的唯一权威汇总） |

> 本文档汇总当前作业的背景、要求、任务清单、**阶段出口**、提交物与验收标准。当本文档与仓库内其他说明文件冲突时，以本文档为准，并以 `make` 目标的实际行为作为最终裁决。
>
> 阅读顺序建议：§1–§4 建立背景 → **§5 看怎么做、做到什么程度算过** → §6 查每个任务的具体要求 → §9 核对评分权重。

---

## 1. 背景

### 1.1 问题从哪来

稳定币的「稳定」并不来自 ERC-20 代码。ERC-20 只是一套记账格式，任何人十分钟就能写出来。真正决定一枚币是否锚定 1 美元的，是它背后的**铸造—赎回闭环（mint/redeem loop）**，以及围绕这个闭环设计的**权限体系**。

本实验要求从零实现一个**法币抵押型稳定币**：存入抵押物才能铸造，销毁稳定币才能赎回，且**任何时刻都保持 100% 足额抵押**。

### 1.2 核心命题

```
vault.totalCollateral() == stable.totalSupply()
```

只要这个等式成立，每一枚流通中的币都有对应的抵押物在金库中。一旦有人能在没有抵押物进入金库的情况下凭空铸币，等式破裂，币就不再稳定——**这就是脱锚（depeg）**。

### 1.3 现实参照

| 事件 | 现象 | 与本实验的对应 |
|---|---|---|
| 2008 货币市场基金「跌破面值」 | 赎回被冻结数日 | 讨论题 B2：暂停赎回的代价 |
| 2023 USDC 脱锚至 $0.87 | 储备银行倒闭，但**赎回通道保持开放** | 讨论题 B2：开放赎回 vs 冻结赎回 |
| MakerDAO | 超额抵押 + 清算机制 | 练习 Ex5：150% 铸造线 / 120% 清算线 |

### 1.4 与后续课程的关系

本实验是下周 **RWA（真实世界资产）实验**的入口。当抵押物从「链上现金」变成美国国债或不动产时，`totalCollateral()` 这个不变量还能不能这样写——这是讨论题 D1/D2 的核心，也是后续课程的起点。

---

## 2. 实验目标

完成本实验后，应当能够：

1. 说清**稳定币的稳定性来自铸赎闭环而非代币代码**，并在命令行上亲手走通一次完整闭环；
2. 识别稳定币合约中的**权限设计风险**（谁能铸、谁能烧、谁能暂停），并用测试把后门「证明」出来，而不是辩解它；
3. 处理**多套十进制精度并存**的陷阱（6 / 8 / 18 位小数）；
4. 实现**超额抵押与清算**机制，并理解清算中「坏账」为何无法避免；
5. 用**不变量测试（invariant testing）**让机器随机调用合约、自动搜索反例，逼近真实安全审计的方法；
6. 在一个真实靶场（Damn Vulnerable DeFi）中定位并利用一个金库的设计缺陷。

---

## 3. 系统组成

### 3.1 三份核心合约

| 文件 | 角色 | 关键设计 |
|---|---|---|
| `src/MockUSDC.sol` | **抵押物**。模拟 USDC：6 位小数，带测试水龙头 `faucet()` | 与 18 位小数的直觉形成对照 |
| `src/SimpleStablecoin.sol` | **稳定币本体**。ERC-20（`sUSD`，6 位小数）+ `MINTER_ROLE`（谁能铸）+ `PAUSER_ROLE`（是否冻结） | `_update` 被覆写并加 `whenNotPaused`，因此暂停会同时冻结转账、铸造与赎回 |
| `src/Vault.sol` | **闭环本体**。`deposit()` 存抵押物 1:1 铸币；`redeem()` 烧币取回抵押物 | 先 burn 后 transfer（CEI 顺序），防重入 |

### 3.2 练习用第二套系统

`src/exercises/OverCollateralizedVault.sol` 是 Ex5 的战场，与上面三者并列：

- 抵押物 `MockWETH`（18 位小数），价格源 `MockPriceFeed` / `IPriceFeed`（8 位小数）；
- 常量：`RATIO_PRECISION = 100`、`MIN_COLLATERAL_RATIO = 150`、`LIQUIDATION_RATIO = 120`、`LIQUIDATION_BONUS = 10`；
- 其中留有 **4 个 TODO**，即 Ex5.1–Ex5.4。

### 3.3 部署脚本

`script/Deploy.s.sol` 一条命令部署三份合约，并在最后把 `MINTER_ROLE` 授予 Vault。

> ⚠️ 这一步不能省。**金库没有铸币权，每一次 `deposit` 都会 revert**——这是本实验最高频的翻车点。

---

## 4. 技术栈与环境约束

| 项目 | 约定 |
|---|---|
| 工具链 | Foundry（`forge` / `cast` / `anvil`） |
| Solidity | `0.8.24`（`foundry.toml` 指定） |
| OpenZeppelin | **v5.0.2，固定版本，不得升级**（v5 有破坏性变更，网上 v4 教程会直接报错） |
| 测试库 | `forge-std` |
| 依赖位置 | `lib/` **随仓库分发**，普通 `git clone` 即可编译，无需 `--recursive`，也无需 `make setup` |
| 编译参数 | optimizer 开启，`optimizer_runs = 200` |
| 格式化 | `forge fmt`，`line_length = 100` |
| 本地链 | Anvil，默认 RPC `http://127.0.0.1:8545` |

**环境路线（二选一，之后内容完全相同）**

- **路线 A（macOS / Linux）**：本地安装 Foundry。**不要用 `foundryup`**，使用仓库自带脚本 `bash scripts/install-foundry-cn.sh`（自动走 `gh-proxy.com` 镜像）。
- **路线 B（Windows）**：Foundry 无原生 Windows 版，走 GitHub Codespaces，本地零安装。

**环境自检**

```bash
make doctor    # 约 30 秒，检查工具链、依赖并实际跑一次核心测试；✗ 项自带修复命令
```

`make doctor` 全绿、且 `make test` 显示 `7 passed; 0 failed`，才算具备开工条件。若无法修复，把 `make doctor` 的**完整输出**发给助教。

---

## 5. 阶段路线图：每阶段的出口与意义

§6 给出的是**任务清单**（做什么），本节给出的是**执行顺序**（按什么次序做、做到什么程度算过）。

每个阶段都有一个**出口（Exit）**：一条可执行、有明确通过/不通过判定的命令或交付物。**出口未过，不进入下一阶段**——这正是本实验「红转绿」的含义：红色不是提醒，是闸门。

### 5.1 阶段总览

| 阶段 | 包含 | 出口（判定条件） | 意义（一句话） |
|---|---|---|---|
| **P0** 环境就绪 | Ex0 | `make doctor` 无 ✗，`make test` = 7 passed | 一切命令的前提；自检先于动手的习惯 |
| **P1** 闭环直觉 | Ex1 | 命令行跑完一次完整往返，三个数字相等 | 「稳定来自铸赎闭环，而非代币代码」——后面所有判断的地基 |
| **P2** 用测试钉死机制 | Ex2 + Ex4 | `01_*` 组 7 个测试全绿 | 两个认知拐点：不变量成立 ≠ 系统健康；钥匙交给了谁 |
| **P3** 亲手脱锚 | Ex3 | 一张 `supply >> collateral` 的截图 | 不改一行代码就能打死系统：权限即攻击面 |
| **P4** 超额抵押与清算 | Ex5 | `03_*` 组全绿 | 从稳定币跨进借贷协议；认识坏账为何无法避免 |
| **P5** 不变量测试 | Ex6 | `02_*` 组全绿 | 从「我构造场景」到「机器搜索反例」的方法论跃升 |
| **P6** 威胁模型作答 | 讨论题 A–E | 9 处作答完毕，每题 150–300 词 | 25% 权重；把前五个阶段的动手经验兑换成判断力 |
| **P7** 上测试网 | Tier 2 | Sepolia 三地址 + Etherscan 源码已验证 | 真实成本、不可逆性、公开可核对 |
| **P8** 挑战 | Tier 3 | `make challenge` 转绿，或三选一 | 从防守位换到进攻位 |

**依赖关系**

```
P0 ──> P1 ──> P2 ──┬──> P4 ──> P5 ──┐
                   │                 ├──> P6 ──> 提交
P0 ──> P3 ─────────┘                 │
                                     └──> P7 / P8（彼此独立，任意时点可插入）
```

- **P0 → P1 → P2 是硬链条**，不能跳。
- **P3 只依赖 P0**（需要一条跑着的链），可与 P2 并行，但它产出的截图是 P6 中 A1 / C2 的实证基础。
- **P4、P5 依赖 P2**（要先会用 `vm.expectRevert` 和 fuzz 断言）。
- **P6 依赖 P2–P5 的亲身经验**——这些题靠查资料答不好。
- **P7、P8 独立**，可在任何时点插入，不影响主线。

---

### 5.2 P0 · 环境就绪（Ex0）

**出口**

- [ ] `make doctor` 输出 `N passed, 0 failed`，没有任何一行 `✗`
- [ ] `make test` 输出 `7 passed; 0 failed`
- [ ] `forge --version` 有输出（证明 PATH 已生效）

**意义**

这是唯一一个「不做完就什么都做不了」的阶段：后续每一条命令都依赖 `forge` / `cast` / `anvil`。

更要紧的是 `make doctor` 的**结构**。它把「工具链在不在」「依赖目录是否完整」「能否编译并跑通测试」拆成三段分别报告，所以任何一个 `✗` 都直接指向一个具体的修复动作，而不是笼统的「跑不起来」。**先自检、再动手**——这个习惯本身就是本实验要训练的东西之一，后面 Ex6 的反例序列、Tier 2 的验证失败，都是同一个思路的延伸。

技术细节见 §6.1。

---

### 5.3 P1 · 闭环直觉（Ex1）

**出口**

- [ ] 在 Anvil 上完成一次完整往返：`faucet` → `approve` → `deposit`（铸币）→ `redeem`（取回）
- [ ] 亲眼确认三个数字相等：`totalSupply()` == `totalCollateral()` == `balanceOf(you)`
- [ ] 能说清 `approve` 为什么是必需的（金库用 `safeTransferFrom` 主动拉走你的钱，而不是你推给它）

**意义**

这是唯一一个**不写代码**的阶段，却是后面所有阶段的认知地基。ERC-20 代码本身不含任何「稳定」——稳定是铸赎闭环的产物。只有当你在区块浏览器/终端里看到「抵押物进金库」和「币出现在你账上」是**同一次交易的因果**，Ex2 的小数陷阱、Ex6 的不变量才会有直觉支撑。

跳过这一步直接写测试，等于在背答案：测试会绿，但你说不清绿的是什么。

技术细节见 §6.1。

---

### 5.4 P2 · 用测试把机制钉死（Ex2 + Ex4）

**出口**

- [ ] `test/exercises/01_LoopTasks.t.sol` 中 2 个 `test_Ex2_*` 全部转绿
- [ ] 同文件中 5 个 `test_Ex4_*` 全部转绿
- [ ] `make exercise` 中 `01_*` 组全绿
- [ ] 能用一句话回答 Ex2.2 的追问：**操作没 revert、不变量也成立，那到底哪里错了？**

**意义**

这一阶段藏着本实验的**两个认知拐点**。

**拐点一（Ex2）：「不变量成立 ≠ 系统健康」。** `1000e18` 在 6 位小数下是 10^15 枚币——一个足以买下整个演示系统的数量——但它同时进入等式两边（存进去多少就铸出来多少），所以 `totalCollateral() == totalSupply()` 照样成立。这说明**一条不变量不足以保护系统**：你还必须约束「量」的语义。这个认识直接决定讨论题 C1 能答到什么深度。

**拐点二（Ex4.5）：「代码写对了」与「钥匙交给了谁」是两个独立问题。** 五个测试里的最后一个——《金库能烧掉任何人余额》——不是为了给这个设计辩护，而是为了**证明后门真实存在**。前四个测试证明守卫写得没错，第五个证明钥匙被交给了金库。守卫正确 + 权限过大，系统依然可以被单点摧毁。

技术细节见 §6.2。⚠️ `test/Stablecoin.t.sol` 内含参考答案，动手前不要打开。

---

### 5.5 P3 · 亲手脱锚（Ex3）

**出口**

- [ ] 用 `cast` 给攻击者授予 `MINTER_ROLE`，再让攻击者凭空铸币成功
- [ ] **一张你自己跑出来的**截图：`totalSupply()` 远大于 `totalCollateral()`

**意义**

Ex4.5 证明「后门存在」，Ex3 用这个后门**把系统打死**。两次操作之间没有改一行代码，只多了一次 `grantRole`——这是本实验对「权限即攻击面」最直白的一次示范。

它同时是讨论题 A1 和 C2 的**实证基础**：只有先亲手制造一次脱锚，你写「如何改 Vault 和 SimpleStablecoin 来消除这个风险」时说的才是自己见过的东西，而不是抄来的方案。先做出来，再回头写分析，答案的质地完全不同。

技术细节见 §6.1（Ex3 行）。

---

### 5.6 P4 · 超额抵押与清算（Ex5）

**出口**

- [ ] `src/exercises/OverCollateralizedVault.sol` 的 4 个 TODO 全部实现
- [ ] `make exercise` 中 `03_*` 组全绿，**含 fuzz 测试** `testFuzz_Ex5_SupplyBackedByValue`
- [ ] 能独立复述三套小数的换算：`18 + 8 - 6 = 20`，即 `amount * price / 1e20`
- [ ] 能回答：清算奖励超过剩余抵押物时，缺口由谁承担？

**意义**

从 1:1 抵押跨到「抵押率」，是**稳定币到借贷协议的分界线**。在 1:1 的世界里不存在资不抵债——抵押物和负债永远等额。一旦引入价格源，抵押物价值开始波动，于是必须回答三个新问题：

1. 铸币时留多少缓冲？（150%）
2. 跌破多少可以强制平仓？（120%）
3. 凭什么有人愿意来干这件事？（10% 清算奖励）

而 Ex5.4 里那个不可避免的 `min()` 封顶，会把你带到本阶段最不舒服的结论：**价格跌得够狠时，抵押物付不起清算奖励，缺口就是坏账——真实借贷协议同样逃不掉**。这不是代码没写好，是机制的内在边界。这个认识是下周 RWA 课程的入场券。

技术细节见 §6.3。

---

### 5.7 P5 · 不变量测试（Ex6）

**出口**

- [ ] handler 的 `redeem` 实现完毕（含「用户余额为 0 则提前返回」分支）
- [ ] `invariant_CollateralBacksSupply` 改写成真正要捍卫的性质
- [ ] `invariant_VaultHoldsNoStablecoin` 同上，并说清它**为什么必须成立**
- [ ] `make exercise` 中 `02_*` 组全绿

**意义**

这是**方法论的跃升**。到 P4 为止，所有测试都是「我构造一个场景，然后检查结果」——它只能覆盖你想得到的攻击，而你想不到的恰恰是最危险的。

不变量测试把方向反过来：**让机器随机、连续地调用操作，然后问「无论怎么折腾，这条性质还在不在」**。没人能猜出攻击者的操作顺序，但机器可以穷举。当断言失败时，Foundry 打印的那串**反例调用序列**，就是一份可以直接复现的审计报告雏形——这是本实验中最接近真实安全审计的环节。

第二个不变量还训练另一件事：**知道一条性质为什么必须成立，比知道怎么写断言更重要**。金库只把 sUSD 铸给用户、自己不该留存——如果它真持有了 sUSD，那意味着什么？这个问题没有标准答案，但你必须能回答。

技术细节见 §6.2。

---

### 5.8 P6 · 威胁模型作答（讨论题 A1–D2 + E）

**出口**

- [ ] `STUDENT-QUESTIONS.md` 全部 9 处作答完毕，每题 150–300 词
- [ ] E 组额外补一个「你认为最可能被攻击的场景」并说明选择理由
- [ ] 每个论断都能指回 P1–P5 中你亲眼看过的某个现象或某次测试

**意义**

**权重 25%，是全表第二高的单项，且不需要写一行代码。**

它是把前五个阶段的动手经验**兑换成判断力**的地方，也是本实验真正的重点（README 原话：这些题没有标准答案）。B2（2008 冻结赎回 vs 2023 USDC 保持赎回开放）和 D1/D2（抵押物换成国债、换成一栋楼）已经越出了代码，进入**机制设计与法律结构**——那才是稳定币在现实中真正被卡住的地方。

⚠️ **不能留到最后一天赶。** 这些题要的是你做过的事，不是查得到的资料；截至 P5 你已经具备全部作答素材，拖到 P7 之后只会遗忘细节。

---

### 5.9 P7 · 上测试网（Tier 2，加分）

**出口**

- [ ] `script/Deploy.s.sol` 在 Sepolia 广播成功，拿到三个合约地址
- [ ] 至少 `SimpleStablecoin` 在 Etherscan 上显示为**已验证源码**
- [ ] 全程使用**全新的丢弃式钱包**，私钥只存在于 `.env`

**意义**

本地链与真实网络之间隔着的不是技术，而是**成本与不可逆性**。Anvil 上重启一次就干净了；Sepolia 上每一笔交易都花掉真金白银的测试 ETH、都留下永久记录、都无法撤回。

这会让你切身体会到一件在本地永远感受不到的事：**为什么「暂停」「权限」「升级」这些设计在真实系统里代价高昂**——因为它们全都要求你拥有在不可逆环境里改变规则的能力。

Etherscan 验证带来的**公开可读性**也是稳定币的核心诉求：储备与代码能不能被外人独立核对，正是这个行业信任的落点。

技术细节见 §6.4。⚠️ **提交私钥本项计零分。**

---

### 5.10 P8 · 挑战（Tier 3，可选）

**出口**（任选其一）

- [ ] `make challenge` 转绿——在 `test_unstoppable()` 中让金库**停止提供闪电贷**
- [ ] 或：实现 PSM（1:1 兑换通道）/ 收紧 burn 权限 / 接入真实 Chainlink（**先读 `decimals()`，不得硬编码 8**）

**意义**

前面所有阶段你都在**防守**：写测试、建不变量、推演威胁模型。这一阶段换到**进攻位**：面对一个别人写好的、看起来完全正常的金库，找出它的设计假设在哪里不成立。

Ex7 的提示只有一句「这个金库对自己的余额做了什么假设？」——答案在 `UnstoppableVault.sol:82`：

```solidity
if (convertToShares(totalSupply()) != balanceBefore) revert InvalidBalance();
```

金库把「我持有的代币数量」直接等同于「我发行的份额对应的资产」。这个等式只在**所有代币都经由 `deposit` 流入**时才成立；任何人绕过 `deposit` 直接转账进来，等式立刻崩掉，闪电贷从此永久失效——攻击者只需要 1 枚 DVT 和一次普通转账。

**从「我知道怎么防」到「我知道怎么破」，是合约开发者与安全工程师的分水岭。**

技术细节见 §6.5。

---

## 6. 作业要求

### 6.1 Tier 1（必做）— Ex0 至 Ex6

| # | 任务 | 验收命令 / 交付物 | 卡住时看哪里 |
|---|---|---|---|
| **Ex0** | 环境搭建，核心测试全绿 | `make doctor` 全绿 → `make test`（7 绿） | `README.md` §1 Setup |
| **Ex1** | 在命令行上完整走一遍铸赎闭环：水龙头领 mUSDC → `approve` → `deposit` 铸币 → `redeem` 取回抵押物 | `make mint AMOUNT=1000000000` 后 `make balance TO=<地址>`；三个数字必须相等 | `README.md` §2、`EXERCISES.md` Ex1 |
| **Ex2** | 在测试中捕获 **6 位 vs 18 位小数陷阱** | `test/exercises/01_LoopTasks.t.sol` 中两个 `test_Ex2_*` 转绿 | Ex2 提示、`test/exercises/01_LoopTasks.t.sol` |
| **Ex3** | **亲手打掉锚定**：用 `cast` 给攻击者授予 `MINTER_ROLE` 后铸币 | 一张截图：`totalSupply()` 远大于 `totalCollateral()` | `EXERCISES.md` Ex3 的四条命令 |
| **Ex4** | 为权限与暂停写 5 个测试 | 同文件中五个 `test_Ex4_*` 转绿 | `STUDENT-QUESTIONS.md` A1/B1 |
| **Ex5** | 实现超额抵押金库与清算（4 个 TODO） | `test/exercises/03_OverCollateralTasks.t.sol` 全绿 | `src/exercises/OverCollateralizedVault.sol` |
| **Ex6** | 实现不变量测试：补全 handler 的 `redeem`，改写两个 `invariant_*` | `test/exercises/02_InvariantTasks.t.sol` 全绿 | `EXERCISES.md` Ex6 提示 |
| **Ex7（挑战，计入 Tier 3）** | 让一个真实金库停止提供服务 | `make challenge` 转绿 | `test/challenges/Unstoppable.t.sol` |

**Tier 1 不要求部署到测试网**，本地 Anvil 足够。所有练习的统一验收命令是：

```bash
make exercise    # 未完成时是红的——红色就是你的任务清单，不是 bug
```

> 注意区分两个命令：`make test` 是「一切正常」的检查点（应始终全绿）；`make exercise` 是你的作业清单（完成后转绿）。

### 6.2 Ex2 / Ex4 / Ex6 详细要求

**Ex2 · 小数陷阱**（两个测试）

1. `test_Ex2_DepositIncreasesSupplyByExactly`：对任意合法金额 x，`deposit(x)` 后供应量必须**恰好**增加 x（提示：用 `vm.assume` 排除 x == 0，并先给 alice 领足 mUSDC）。
2. `test_Ex2_DecimalsTrap`：用 `1000e18`（而非 `1000e6`）跑一遍，断言你观察到的结果。

> 关于第 2 题：**这笔操作没有 revert，不变量也仍然成立。**那么到底哪里错了？
> 提示：`MockUSDC` 是 6 位小数，`1000e18` 是 **10^15** 枚币而不是 1000 枚，差了 10^12 倍。这道题没有标准答案，重点是你亲手跑、亲眼看数字。

**Ex4 · 权限与暂停**（五个测试）

| 函数 | 要证明的事 |
|---|---|
| `test_Ex4_Mint_RevertsForNonMinter` | 没有 `MINTER_ROLE` 就不能铸币 |
| `test_Ex4_Pause_BlocksTransfers` | `pause()` 后转账 revert |
| `test_Ex4_Pause_BlocksRedeem` | **赎回也被一并冻结** |
| `test_Ex4_AttackerCannotBurnOthersBalance` | 攻击者动不了别人的余额 |
| `test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance` | **但金库可以** |

最后一条不是为了给它辩护，而是为了**证明这个后门真实存在**。它与 `STUDENT-QUESTIONS.md` 的 A1 对应：先想清楚，再回头作答。第三条对应 B1/B2：若想实现「冻结转账但允许赎回」，代码该怎么改？

**Ex6 · 不变量测试**（三件事）

1. 实现 handler 的 `redeem`（两个参数尚无名字，先命名）：随机选用户 → 若其余额为 0 则提前返回 → 将金额约束在 `[1, 余额]` → 以该用户身份调用 `vault.redeem(amount)`。
2. 把 `invariant_CollateralBacksSupply` 改成你真正想捍卫的性质（当前断言故意写错，所以它是红的）。
3. 对 `invariant_VaultHoldsNoStablecoin` 做同样的事。

> 提示：第二个不变量为什么必须成立？金库只把 sUSD 铸给用户，自己不该留存任何 sUSD。如果它真的持有了，那意味着什么？

当断言失败时，Foundry 会打印**反例调用序列**——照着这个序列手工走一遍，你就能看到不变量到底在哪一步、被哪组参数打破。这是本实验中最接近真实安全审计的环节。

### 6.3 Ex5 · 超额抵押与清算（四个 TODO）

| TODO | 要做的事 | 难点 |
|---|---|---|
| Ex5.1 `collateralValue` | 把抵押物换算成 sUSD 计价 | 18 位 × 8 位 → 6 位：除以 10 的几次方？ |
| Ex5.2 `mintStable` | 铸币后抵押率不得低于 150% | 先记账还是先检查？ |
| Ex5.3 `redeemCollateral` | 提取抵押物后抵押率不得低于 150% | 同上 |
| Ex5.4 `liquidate` | 低于 120% 时任何人可清算 | 奖励可能超过剩余抵押物 |

**三套十进制必须分清**（这是 Ex2 的续集）：

```
抵押物 mWETH   18 位小数
稳定币 sUSD     6 位小数
价格源          8 位小数   →   18 + 8 - 6 = 20
```

> 关于 Ex5.4 的思考题：当价格跌得足够狠，「债务价值 × 110%」可能超过用户存入的全部抵押物。**差额由谁承担？为什么真实借贷协议也逃不掉这个问题？**

### 6.4 Tier 2（加分）— 测试网部署与源码验证

把合约部署到 **Sepolia** 并在 Etherscan 上验证源码，提交合约链接。

```bash
cp .env.example .env    # 填入私钥、RPC URL、Etherscan API Key
source .env
forge script script/Deploy.s.sol:Deploy \
  --rpc-url $SEPOLIA_RPC_URL --broadcast --verify --private-key $PRIVATE_KEY
```

- `--verify` 需要 `ETHERSCAN_API_KEY`（<https://etherscan.io/myapikey> 免费申请，几秒钟）。
- **API Key 留空不影响部署**：合约照样上链，只有验证这一步报错。**不要因此重新部署**，事后单独验证即可：

```bash
forge verify-contract <address> src/SimpleStablecoin.sol:SimpleStablecoin \
  --chain sepolia --etherscan-api-key $ETHERSCAN_API_KEY
```

- ⚠️ **必须使用全新的、只放测试币的丢弃式钱包**。私钥只能留在 `.env`（已在 `.gitignore` 中）——**提交私钥本项计零分**。
- Sepolia 测试 ETH 水龙头：<https://cloud.google.com/application/web3/faucet/ethereum/sepolia>

### 6.5 Tier 3（挑战，可选）— 任选其一

| 选项 | 内容 |
|---|---|
| **Ex7 · 打垮金库** | `make challenge` → `test/challenges/Unstoppable.t.sol`（移植自 Damn Vulnerable DeFi v4）。你持有 10 DVT，目标是让该金库**停止提供闪电贷**。只需改 `test_unstoppable()` 一个函数，提示已写在文件里：*这个金库对自己的余额做了什么假设？* |
| **PSM（锚定稳定模块）** | 增加一条对另一种稳定币的 1:1 兑换通道，让套利者替你拉回锚定价格。 |
| **收紧 burn 权限** | 消除「金库可以烧掉任何人余额」的中心化风险——即讨论题 A1 的代码版本。 |
| **接入真实 Chainlink** | 用 Sepolia 的 `AggregatorV3Interface` 替换 `MockPriceFeed`。**先读 `decimals()`，不要硬编码 8**。 |

---

## 7. 讨论题要求（`STUDENT-QUESTIONS.md`）

在每道题下方直接作答，**每题 150–300 词，重推理而非篇幅**。

| 分组 | 题号 | 主题 |
|---|---|---|
| A. 权限设计 | A1 / A2 | 金库持有 `MINTER_ROLE` 可烧任何人余额的风险与改法；`DEFAULT_ADMIN_ROLE` / `MINTER_ROLE` / `PAUSER_ROLE` 在生产线应如何拆分、分别由谁持有 |
| B. 暂停与赎回 | B1 / B2 | 如何实现「暂停转账但允许赎回」；对比 2008 冻结赎回与 2023 USDC 保持赎回开放两种应对 |
| C. 脱锚分析 | C1 / C2 | 本币在什么条件下脱锚（至少区分两类成因，并说明各自如何体现在不变量上）；内鬼获得 `MINTER_ROLE` 后凭空铸币并全部赎回的资金流向，以及本可拦截的那一步 |
| D. 走向 RWA | D1 / D2 | 抵押物换成美国国债后，不变量还能否这样写、会出现哪些新问题；抵押物是一栋楼时如何装进金库、需要引入哪些链下角色与法律结构 |
| E. 测试 | E1 | 填写 Ex4 五个测试的函数名，并**自行补充一个你认为最可能被攻击的场景**，说明选择理由 |

> 这些题目**没有标准答案**，它们才是本实验真正的重点。

---

## 8. 提交物清单

1. **仓库链接**，附一张 `forge test` 通过的截图；
2. **Tier 2**：Sepolia 合约地址 + Etherscan 链接；
3. **讨论题作答**：填写完成的 `STUDENT-QUESTIONS.md`；
4. **架构图**：放进你的 `README.md`（手绘拍照也可以）；
5. **Ex3 截图**：`totalSupply()` 远大于 `totalCollateral()` 的那一张。

---

## 9. 评分标准

| 评分项 | 权重 |
|---|---|
| 合约正确性 + 测试覆盖率 | **40%** |
| 部署与 Etherscan 源码验证（Tier 2） | **20%** |
| 威胁模型分析（讨论题） | **25%** |
| 代码风格与文档 | **15%** |

---

## 10. 验收标准与红线

### 10.1 完成定义（Definition of Done）

> 本清单是 §5 各阶段出口的**并集**：每通过一个阶段出口，勾掉对应的几项。

- [ ] `make doctor` 全绿
- [ ] `make test` 全绿（`7 passed; 0 failed`）
- [ ] `make exercise` 全绿（Ex2 / Ex4 / Ex5 / Ex6 全部转绿）
- [ ] Ex1 的闭环已在命令行上实跑过一次，且 `totalSupply()` / `totalCollateral()` / `balanceOf(you)` 三者相等
- [ ] Ex3 的脱锚截图已产出
- [ ] `STUDENT-QUESTIONS.md` 全部作答（每题 150–300 词）
- [ ] `README.md` 中已补上架构图
- [ ] `forge fmt` 已执行，格式符合 `line_length = 100`
- [ ] （Tier 3 选做）`make challenge` 转绿 / 或完成 PSM、权限收紧、Chainlink 接入之一

### 10.2 红线（违反即扣分或计零）

1. **不得提交私钥**——`.env` 已在 `.gitignore` 中；提交私钥本项**计零分**。Tier 2 只能用全新的、仅含测试币的丢弃式钱包。
2. **不得修改 `lib/`**——依赖随仓库分发，改坏了用 `git checkout -- lib` 还原。
3. **不得升级 OpenZeppelin**——本仓库锁定 `v5.0.2`。
4. **先自己写，再看答案**——`test/Stablecoin.t.sol` 内含参考答案，请勿在动手前打开。

### 10.3 高频问题速查

| 症状 | 原因与处理 |
|---|---|
| `Ownable` / 角色相关报错 | OZ v5 有破坏性变更，网上 v4 教程会失败；本仓库固定 v5.0.2 |
| `ds-test/test.sol` 找不到 / `lib/` 为空 | 执行 `git checkout -- lib` |
| 部署后 `deposit` 一直 revert | 十有八九漏了 `stable.grantRole(stable.MINTER_ROLE(), address(vault))` |
| 金额差了一个数量级 | `1000e6 = 1_000_000_000`；两种代币都是 6 位小数，**不要按 18 位算** |
| 重启 Anvil 后地址变了 | Anvil 每次从干净状态启动，必须重新部署；需保留状态用 `anvil --load-state demo-state.json` |
| `make exercise` 全红 | 这是**预期行为**——`test/exercises/` 下的测试故意是红的，它们就是任务清单 |
| `make exercise` 卡住 / 不变量失败结果诡异 | Foundry 会缓存反例到 `cache/invariant/`，先 `rm -rf cache/invariant` 再跑 |
| 想保留 / 恢复演示链状态 | `make snapshot`（Ctrl-C 时存盘）、`make restore` |

---

## 11. 常用命令速查

```bash
make doctor        # 环境自检（约 30 秒）
make test          # 核心测试，应始终全绿（7 passed）
make exercise      # 作业清单，完成后转绿
make challenge     # Ex7 挑战，解开前一直是红的
make fmt           # 格式化合约
make anvil         # 终端 A：启动本地链
make deploy-anvil  # 终端 B：部署到本地链（会打印三个地址）
make mint AMOUNT=1000000000   # 需先 export VAULT=/SUSD=/USDC=
make balance TO=<地址>         # 查询稳定币余额
make snapshot / make restore  # 保存 / 恢复链上状态
```

---

## 12. 关键概念与精度对照

| 概念 | 含义 |
|---|---|
| **抵押率（collateral ratio）** | 抵押物价值 ÷ 债务价值。本实验中 150% 是铸造下限，120% 是清算线 |
| **清算（liquidation）** | 抵押率跌破 120% 后，任何人可代为偿还债务并按 110%（含 10% 奖励）夺取抵押物 |
| **坏账（bad debt）** | 价格暴跌时抵押物不足以支付清算奖励的缺口 |
| **脱锚（depeg）** | `totalCollateral() < totalSupply()` 或赎回通道受阻，币价脱离 1 美元 |
| **不变量测试** | 让机器随机、连续地调用合约操作，检验某条性质是否恒成立；失败时给出反例调用序列 |
| **PSM** | 锚定稳定模块：与另一稳定币的 1:1 兑换通道，靠套利把价格拉回锚定 |

**小数位对照表**

| 对象 | 小数位 | 备注 |
|---|---|---|
| `MockUSDC` (mUSDC) | 6 | 法币稳定币惯例 |
| `SimpleStablecoin` (sUSD) | 6 | 本实验的稳定币 |
| `MockWETH` (mWETH) | 18 | Ex5 抵押物，故意与上面形成反差 |
| `MockPriceFeed` | 8 | 模拟 Chainlink ETH/USD |
| Ex5 换算 | — | `18 + 8 - 6 = 20`，即除以 `1e20` |

---

## 13. 变更记录

| 版本 | 日期 | 说明 |
|---|---|---|
| v1.1 | 2026-10-15 | 新增 §5「阶段路线图」：把任务清单重排为 P0–P8 九个阶段，每阶段给出**出口（Exit）**与**意义**，并标注依赖关系；后续各节顺序号相应后移 |
| v1.0 | 2026-10-15 | 首版：汇总 README / EXERCISES / STUDENT-QUESTIONS / Makefile 中的作业要求，形成单一权威规范 |

**后续规范文件计划**（占位，待撰）

- 架构与不变量形式化说明
- 威胁模型与权限矩阵
- 测试策略与验收细则
- RWA 扩展（下周课程）
