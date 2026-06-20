# Reputation / Credit 调研报告 — SuperPaymaster × AirAccount

> 调研日期：2026-06-20 · 范围：AAStarCommunity 组织下 `SuperPaymaster`、`airaccount-contract`、`AirAccount`、`registry`、`CoS72` 仓库（GitHub HEAD，2026-06）+ 本地 `PhD/SuperPaymaster` v1.x 设计稿。
> 目的：为 **MyNFT / OpenEvent NFT (OEN)** 确定「NFT 作为社区声誉工具」的可复用基础设施与设计落点。

---

## 0. 一句话结论

现有生态**已经把「NFT → 声誉 → credit」这条链路造好了大半**，而且明确区分了你说的 **community inner reputation（社区内声誉）** 与 **global reputation（全局声誉）**。MyNFT 不需要新发明声誉合约——它要做的是「**多入口发放活动 NFT**」这个上游，把发出去的 NFT 喂给已有的 `ReputationSystem`，让 NFT 自动转化为分数。**零新合约**即可接入（与 PGL/Mycelium 的设计原则一致）。

---

## 1. 最关键的认知校正：「reputation」在本生态有两层完全不同的含义

调研时必须先分清，否则会读错代码：

| 层 | 谁的声誉 | 用途 | 代码位置 | 标准 |
|---|---|---|---|---|
| **A. 服务节点声誉** | Paymaster / DVT / KMS **运营节点** | 评估节点可靠性、调度选节点、slash 作恶 | `Registry.sol` 角色质押/惩罚、SuperPaymaster 论文 §4.7 | EIP-7562 |
| **B. 用户/社区声誉** ⭐ | **普通用户（参与者）** | 记录参与行为、算 credit、解锁权益 | `ReputationSystem.sol` + `MySBT.sol` | 自研 + ERC-8004 |

> 本地 `PhD/SuperPaymaster/v1.8` 文档里出现的 "reputation" **几乎全是 A 层**（节点声誉、trust flywheel、EIP-7562）。**MyNFT 要用的是 B 层**——别被论文误导。

---

## 2. B 层（用户/社区声誉）核心组件逐个拆解

### 2.1 MySBT — 身份/会员层（`SuperPaymaster/contracts/src/tokens/MySBT.sol`，v3 + v2.4.5-optimized）

- **每个用户一枚 SBT**（`ERC721 "Mycelium Soul Bound Token" / MySBT`，灵魂绑定不可转）。
- 记录用户在**多个社区**的 membership：`CommunityMembership{ community, joinedAt, isActive, metadata }`，`getUserSBT(user)`、`verifyCommunityMembership(user, community)`。
- 由 **Registry** 在角色注册时铸造（`mintForRole` / `airdropMint`），财务（质押/销毁）全在 Registry，SBT 只管身份。
- **链上活动打点**：`recordActivity(user)` 由「社区合约」调用，按周记一次（`MIN_INT` 节流），存 `lastActivityTime[tokenId][community]` → 这是「最近活跃度」的数据源。
- ⚠️ **NFT 绑定函数已从 MySBT 移除**（合约体积优化，v2.4.5）：`// NFT binding functions removed`。声誉计算也移出 MySBT，改为外挂 `reputationCalculator` 合约（`setReputationCalculator`，onlyDAO）。**→ NFT→声誉的逻辑现在住在 ReputationSystem，不在 SBT 里。**

### 2.2 ReputationSystem — 计分层（`SuperPaymaster/contracts/src/modules/reputation/ReputationSystem.sol`，v0.3.2）⭐ 核心

实现 `IReputationCalculator`，**把行为/NFT 折算成分数**。要点：

- **双分数输出**正好对应你的需求：
  - `calculateReputation(user, community, sbtTokenId) → (communityScore, globalScore)`
  - 默认规则（接口注释）：**基础 20 分**（有 SBT 会员）+ **每枚绑定 NFT +3 分** + **每个活跃周 +1 分（近 4 周）**。
- **多租户社区规则**：每个社区（`ROLE_COMMUNITY` 持有者）可自设打分规则 `setRule(ruleId, base, bonus, max, desc)`——即「社区自己定义什么活动值多少分」。
- **NFT Boost（全局）**：`setNFTBoost(collection, boost)` 给某个 NFT collection 设加分；用户**持有 ≥7 天**且 `balanceOf > 0` 才计入（防刷）。最多 50 个 boosted collection。
- **Entropy Factor（熵因子）**：`setEntropyFactor(community, factor)`（1e18=1.0），可调低让某社区声誉「更难获得」——治理旋钮，呼应 Mycelium「真菌/熵」叙事。
- **链上落账**：`computeScore(...)` 是 view 计算；`syncToRegistry(...)` 把分数经 **BLS proof** 写回 `Registry.batchUpdateGlobalReputation`。可信来源由 `Registry.isReputationSource()` / DVT 白名单授权（`setCommunityReputation`）。

> **这就是 MyNFT 的接入点**：MyNFT 发的活动 NFT → 在 ReputationSystem 里 `setNFTBoost` 注册该 collection（或社区 `setRule`）→ 用户领到 NFT 后声誉自动+分，且区分 community / global。

### 2.3 Registry — 角色/质押/惩罚（`SuperPaymaster/contracts/src/core/Registry.sol` + `docs/Registry_Role_Mechanism.md`）

- 统一角色体系（`RoleConfig{ minStake, entryBurn, slashThreshold, slash*, exitFee*, isActive }`），预置角色：`ROLE_PAYMASTER_AOA/SUPER`、`ROLE_ANODE`、`ROLE_KMS`、**`ROLE_COMMUNITY`（minStake 10 ether）**、**`ROLE_ENDUSER`（minStake 0.3 ether，无 slash）**。
- `createNewRole(roleId, config, roleOwner)`（onlyOwner 动态加角色，无需重部署）→ 理论上可为 MyNFT 加 `ROLE_ORGANIZER` 之类。
- 持有「最终声誉分数」存储 + `batchUpdateGlobalReputation`（BLS 验证）。

### 2.4 ERC-8004 — 反馈式声誉（`airaccount-contract/src/interfaces/IERC8004*Registry.sol`）

AirAccount 侧实现了 **ERC-8004「Trustless Agents」** 三件套：`IdentityRegistry` / `ReputationRegistry` / `ValidationRegistry`。

- 这是**面向 AI Agent / 服务方**的声誉：客户端对一次交互 `giveFeedback(agentId, value, decimals, tag1, tag2, ...)`（带签名、可 revoke、可 appendResponse），`getSummary()` 聚合跨客户端的信任分。
- 官方跨链同地址部署（CREATE2）：主网 `0x8004BAa1...`、Sepolia `0x8004B663...`。
- SuperPaymaster v5.3 用它做 **「声誉驱动的分级 gas 赞助」**（Agent Sponsorship 模式）。
- **与 MyNFT 关系**：这是「服务被评价」的声誉模型，**和「参与活动得 NFT」是正交的**。MyNFT 主用 2.2 的 ReputationSystem；ERC-8004 仅在未来「组织者/社区作为可被评价的服务方」场景才相关。

### 2.5 CoS72 — 已有 NFT 发放 UI 参照（`AAStarCommunity/CoS72`）

已有 `CreateCommunityNFTDialog`、`SendNFTDialog`、`CommunityNFT.json`、`AAStarDemoNFT.json`——**社区创建/发放 NFT 的前端流程已有现成参照**，MyNFT 可借鉴交互与 ABI。

---

## 3. 「NFT → reputation → credit」完整链路（现状拼图）

```
[活动发生]
   │  MyNFT 要补的上游：多入口发放
   ▼
发放活动 NFT (ERC-721/1155, POAP-compatible)  ←── MyNFT 负责
   │
   ├─(a) NFT collection 在 ReputationSystem.setNFTBoost 注册 → 持有≥7天自动 +boost 分
   ├─(b) 社区 setRule 定义"参加X活动=N分" + MySBT.recordActivity 打点
   ▼
ReputationSystem.computeScore → (communityScore, globalScore)   ←── 已存在
   │  syncToRegistry (BLS proof)
   ▼
Registry 存储全局声誉分数                                          ←── 已存在
   │
   ▼
credit 应用：分级 gas 赞助 / 治理权重 / 解锁权益 / PNTs           ←── 已存在(v5.3)/规划中
```

**结论**：链路里**绿色「已存在」占了下游 70%**，MyNFT 只要把最上游的「**把 NFT 发出去、并登记到声誉系统**」这一段做扎实，整条 credit 飞轮就转起来。

---

## 4. 对 MyNFT 的直接启示（设计落点）

1. **不要自己写声誉合约**。MyNFT 发的 collection 走 `ReputationSystem.setNFTBoost` / 社区 `setRule` 接入，天然获得 community + global 双分数。与 `docs/ProductDesign.md` 里的 `Registry` 融合点一致。
2. **NFT 必须 POAP-compatible + 可被 ReputationSystem 识别**：标准 `IERC721.balanceOf` 即可被 boost 逻辑读取；元数据按 POAP（`event_id/city/country`）对齐。
3. **「自动签到 → 发 NFT」对应 `recordActivity` + mint**：MyNFT 的签到入口在发 NFT 的同时，可由社区合约调 `MySBT.recordActivity(user)` 打活跃度点。
4. **多入口发放是 MyNFT 的独有价值**：现有生态只有 CoS72 的 dialog 式单入口。MyNFT 要做的 telegram / Chrome 插件 / 白名单 CSV / claim link 多入口，是**上游缺口**，不与现有合约重复。
5. **身份用 MySBT，不要另造**：用户领 NFT 时若无 SBT，可走 Registry 的 `ROLE_ENDUSER` 自助或 gasless airdrop（见 `registry/docs/MINT-SBT-BUSINESS-DESIGN.md` 的 self-service / gasless 两流程）。
6. **gasless 发放复用 SuperPaymaster**：mint/claim 与 AA 账户部署都走 Paymaster 代付，零 ETH 门槛（与 OEN 设计完全吻合）。

---

## 5. 待确认 / 缺口

- ReputationSystem `setNFTBoost` 当前是 **onlyOwner（协议方）**，社区不能自助给自己的 NFT 加 boost——MyNFT 发的 collection 要进 boost 列表，需要协议方治理动作或改权限。**这是接入时要和 SuperPaymaster 团队对齐的点。**
- `computeScore` 的 `activities` 入参依赖**链下喂数据 + DVT/BLS 回写**，不是纯链上自动累加。MyNFT 若要「实时声誉」，需想清楚 off-chain oracle 这一环（或先只用 NFT balance boost 这条纯链上路径）。
- community vs global 的具体权重/熵因子默认值，需读 `ReputationSystem` 部署脚本 `07a_DeployReputationSystem.s.sol` 确认线上配置（本报告未展开）。

---

## 6. 关于 Telegram 发 NFT 子模块

你提到「通过 telegram 说一句话就能发布 NFT」的现成子模块——**等你给代码**。届时计划：
1. 作为 git submodule / 子目录克隆进 MyNFT。
2. 抽取其「自然语言意图 → mint 调用」的技术（解析、签名、Paymaster 代付路径）。
3. 泛化为 MyNFT 的**多入口适配层**：Telegram / Chrome 插件 DOM 抓取 / CSV 白名单 / claim link 共用同一套「发放核心」。

---

*Sources: `AAStarCommunity/SuperPaymaster@HEAD`（ReputationSystem.sol v0.3.2, MySBT.sol, IReputationCalculator.sol, IMySBT.sol, Registry_Role_Mechanism.md, README v5.4.0-beta）, `airaccount-contract@HEAD`（IERC8004ReputationRegistry.sol）, `registry@HEAD`（MINT-SBT-BUSINESS-DESIGN.md）, `CoS72@HEAD`, 本地 `PhD/SuperPaymaster/v1.8`.*
