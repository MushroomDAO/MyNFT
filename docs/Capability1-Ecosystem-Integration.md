# 能力一 · 生态嵌入集成规范（基于本地最新代码）

> 来源：实读 `/Users/jason/Dev/aastar/` 下**最新本地代码**（SuperPaymaster @2026-06-16、registry @2026-06-13、airaccount-contract @2026-06-20）。
> 与 `docs/ReputationResearch.md`（概念调研）互补：本文是**可直接照着写代码的集成规范**——合约地址、函数签名、权限门、接入路径。

---

## 1. 已部署合约地址（从本地 `config.*.json` / deployments 提取）

| 合约 | 网络 | 地址 |
|---|---|---|
| Registry | Sepolia | `0x3F920B25f8b65988359C372F66F036E48adFc556` |
| ReputationSystem | Sepolia | `0x7fEd690E1663755e24a1C9d6164336809d68a578` |
| MySBT | Sepolia | `0x072A0D12f4212B6baD7c6d0A633eaffbDE9105bF` |
| Registry | OP Sepolia / Optimism | `0x997686219F31405503D32728B1f094F115EF24e7` |
| ERC-8004 (Identity/Reputation/Validation) | 多链同址(CREATE2) | `0x8004…`（Sepolia `0x8004B663…`、主网 `0x8004BAa1…`）|

> ⚠️ 接入前用 `cast code <addr>` 或本地 `deployments/` 复核当前 beta 版地址（SuperPaymaster 处于 v5.4.0-beta，可能 redeploy）。

---

## 2. ReputationSystem —— MyNFT 主接入点

合约：`contracts/src/modules/reputation/ReputationSystem.sol`（version `Reputation-0.3.2`）。

| 函数 | 权限门 | 作用 |
|---|---|---|
| `setNFTBoost(address collection, uint256 boost)` | **onlyOwner（协议方）** | 给某 NFT collection 设全局加分；持有≥7天 + `balanceOf>0` 自动计入 |
| `setRule(bytes32 ruleId, uint256 base, uint256 bonus, uint256 max, string desc)` | owner **或** `REGISTRY.hasRole(keccak256("COMMUNITY"), msg.sender)` | 社区自设计分规则（多租户）|
| `setCommunityReputation(address community, address user, uint256 score)` | owner **或** `REGISTRY.isReputationSource(msg.sender)` | **可信源直接写社区声誉，无需 BLS/预言机** |
| `computeScore(user, communities[], ruleIds[][], activities[][])` | public view | 链下/前端预算分数 |
| `syncToRegistry(user, …, epoch, proof)` | 任意调用，但回写经 `Registry.batchUpdateGlobalReputation` 需 **BLS proof** | 把分数写回 Registry 全局声誉 |
| `setEntropyFactor(address community, uint256 factor)` | onlyOwner | 熵因子(1e18=1.0)，调节社区声誉获取难度 |

**默认计分**（`IReputationCalculator` 注释）：有 SBT +20 / 每枚绑定 NFT +3 / 每活跃周 +1。`computeScore` 内对每个 community 应用 entropyFactor，再叠加全局 NFT boost。

---

## 3. Registry —— 角色、声誉源白名单、全局声誉存储

合约：`contracts/src/core/Registry.sol`。

| 函数 | 权限门 | 作用 |
|---|---|---|
| `registerRole(bytes32 roleId, …)` | 用户自助（需质押/burn）| 注册角色（如 `ROLE_COMMUNITY` / `ROLE_ENDUSER`）|
| `setReputationSource(address, bool)` | onlyOwner | **把 MyNFT 后端/合约加入声誉源白名单**（关键开关）|
| `isReputationSource(address) → bool` | view | 校验某地址能否直接写声誉 |
| `hasRole(bytes32 roleId, address) → bool` | view | 角色判定 |
| `batchUpdateGlobalReputation(proposalId, users[], scores[], epoch, proof)` | 需 BLS proof | 批量写全局声誉 |
| `createNewRole(bytes32, RoleConfig, roleOwner)` | onlyOwner | 动态加角色（可为 MyNFT 加 `ROLE_ORGANIZER`）|

预置角色：`ROLE_PAYMASTER_AOA/SUPER`、`ROLE_ANODE`、`ROLE_KMS`、`ROLE_COMMUNITY`(质押10 ether)、`ROLE_ENDUSER`(质押0.3 ether，无 slash)。

---

## 4. MySBT —— 身份/会员/活跃度

合约：`contracts/src/tokens/MySBT.sol`（`Mycelium Soul Bound Token`，灵魂绑定）。

| 函数 | 谁能调 | 作用 |
|---|---|---|
| `mintForRole(user, roleId, roleData)` / `airdropMint(...)` | **仅 Registry** | 角色注册时铸 SBT（财务在 Registry）|
| `recordActivity(address user)` | **持有 `ROLE_COMMUNITY` 的社区合约** | 给用户打周活跃点（`MIN_INT` 节流）|
| `verifyCommunityMembership(user, community) → bool` | view | 会员校验 |
| `getUserSBT(address user) → uint256` | view | 取用户 SBT tokenId（0=无）|
| `setReputationCalculator(address)` | onlyDAO | 切换外挂计分合约 |

> NFT 绑定与计分逻辑已移出 MySBT（体积优化），改由 §2 的 ReputationSystem 承担。

---

## 5. ERC-8004（airaccount-contract）—— 反馈式声誉（正交备选）

`src/interfaces/IERC8004ReputationRegistry.sol` 等：`giveFeedback(agentId, value, decimals, tag1, tag2, endpoint, uri, hash)`、`getSummary(agentId, clients[], tag1, tag2)`、`revokeFeedback`、`appendResponse`。面向「服务被客户端打分」，MyNFT 仅在「组织者/社区作为可评价服务方」时才用；活动参与声誉主走 §2。

---

## 6. 两个缺口 —— 已用代码确认的结论

| 缺口 | 结论（引代码权限门） | MyNFT 落地路径 |
|---|---|---|
| 社区能否自助给自己 NFT 注册 boost？ | ❌ `setNFTBoost` 是 **onlyOwner**，社区不能自助 | 需协议方治理动作；或走下一行的 `setCommunityReputation` 路径绕过 |
| 声誉能否不靠 DVT/预言机直接上链？ | ✅ **能**：`setCommunityReputation` 允许 `owner() ∥ isReputationSource(msg.sender)` | **MyNFT 申请 `Registry.setReputationSource(MyNFT, true)` → 直接写 community 声誉，无需 BLS** |

### MyNFT 推荐接入步骤（最小依赖）
1. 部署 MyNFT 发放合约 + 后端校验服务。
2. 治理动作：`Registry.setReputationSource(MyNFT_后端地址, true)`（onlyOwner，需 SuperPaymaster 团队/DAO 执行一次）。
3. 用户领 NFT / 现场刷卡校验通过后 → MyNFT 后端调 `ReputationSystem.setCommunityReputation(community, user, score)` 直接写社区声誉（无 BLS、无预言机）。
4. 若希望「持有即加全局分」：另向治理申请 `setNFTBoost(MyNFT_collection, boost)`。
5. 全程 mint/写链经 **SuperPaymaster** 代付；身份用 **MySBT**（`ROLE_ENDUSER` 自助或 gasless airdrop）。

> 这条 `isReputationSource` 路径正是硬件方案（`docs/hardware/README.md`）里「现场刷卡 = 可信 attestation」的落点：签到设备/后端即被白名单的 reputation source。

---

*以上签名与权限门均取自本地最新 Solidity 源码；接入前请对照部署网络复核地址与 beta 版本号。*
