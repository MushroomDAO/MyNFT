# MyNFT Submodules —— 开源参考索引

> 本目录用 **git submodule** 引入经调研筛选（GitHub `gh` 搜索 + 验证存在 + star/活跃度/许可证筛选）的开源项目，作为三大能力的**代码与思路借鉴**。每个子模块按「子能力」归类。
> 拉取：`git submodule update --init --recursive`（已用 `--depth 1` 浅克隆）。
> ⚠️ 这些是**参考**，不是依赖。借鉴其设计/接口/流程，不直接 import；注意各自许可证（见下表 License 列）。

能力一（生态嵌入）不在此目录——它借鉴的是**我们自己生态的本地最新代码**，整理在 `docs/Capability1-Ecosystem-Integration.md`。

---

## 能力二 · NFT 发布系统（`capability2-publishing/`）

子能力拆解 → 借鉴的子模块：

| 子能力 | 子模块 | 借鉴点 | ★ | License |
|---|---|---|---|---|
| 无 gas mint 核心 (ERC-4337) | `permissionless.js` (pimlicolabs) | viem 原生智能账户 + paymaster client，gasless mint 主干 | 252 | MIT |
| 白名单 Merkle 空投 | `openzeppelin-merkle-tree` (OpenZeppelin) | 生成 merkle root/proof 的标准库，配 `MerkleProof.sol` | 532 | MIT |
| NFT 领取 dApp | `hashlips-minting-dapp` (HashLips) | 可 fork 的合约+React 领取前端 | 1k+ | MIT |
| 现场 QR 领取 | `poap-qr-kiosk` (poap-xyz) | **轮换二维码签到领取 NFT**——几乎正中用例 | — | 见仓库 |
| POAP 发行 | `poap-sdk` (poap-xyz) | 官方 POAP 发行/查询 SDK，元数据兼容参考 | 16 | MIT |
| IPFS/Arweave 上传 | `pinata-ipfs` (PinataCloud) | 当前在维护的 Pinata SDK，上传图+元数据取 CID | — | MIT |
| Telegram 入口 | `grammY-telegram` (grammyjs) | 最佳 TS Telegram bot 框架，做「一句话发 NFT」入口适配器 | 3.6k | MIT |
| 生成式 NFT 美术 | `hashlips-art-engine` (HashLips) | 图层组合生成引擎（注：非 text-to-image）| 7.2k | MIT |

> 文本生图（AI Studio）无强势 >100★ 专用仓库：用外部 SD/FLUX 端点 + dappuniversity/ai_nft_generator 流程蓝图自建。
> Chrome 插件框架用 **PlasmoHQ/plasmo**（13k★，作工具不 vendor）+ viem 自行脚手架。

---

## 能力三 · 管理与校验 / Attestation（`capability3-verification/`）

| 子能力 | 子模块 | 借鉴点 | ★ | License |
|---|---|---|---|---|
| Attestation 框架 | `eas-contracts` (EAS) | SchemaRegistry + 链上 attestation 主干——「NFT 须背书才计入声誉」的基座 | 318 | MIT |
| Attestation SDK | `eas-sdk` (EAS) | dapp 端创建/验证/撤销 attestation | 137 | MIT |
| ERC-8004 信任层 | `erc-8004-contracts` (erc-8004) | Identity/Reputation/Validation 注册表官方合约 | 222 | 待核实 |
| 人格证明 / 防女巫 | `worldcoin-idkit` (worldcoin) | World ID 工具包，「一人一次」最强反女巫闸 | 483 | MIT |
| ZK 成员证明 | `semaphore` (semaphore-protocol) | 匿名群成员证明，私密白名单/到场证明 | 1k+ | MIT |
| 多签治理 | `safe-smart-account` (safe-fndn) | Gnosis Safe 合约——collection owner 移交多签 | 2.1k | LGPL-3.0 |
| 管理后台基座 | `scaffold-eth-2` (scaffold-eth) | Next.js+wagmi/viem，内置合约调试/admin UI，做活动+声誉控制台 | 2k | MIT |

> 活动签到/票务校验类无高质量专用仓库；用 `dappuniversity/tokenmaster`（票务脚手架）+ EAS 在扫码时铸 attestation 组合实现。
> 链上声誉评分另可参考 `coinbase/verifications`（EAS 门控解锁权益，122★ MIT）。

---

## 硬件子方案 · NFC 必须到场（`hardware/`）

详见 `docs/hardware/README.md`。

| 子模块 | 借鉴点 | License |
|---|---|---|
| `libhalo` (arx-research) | 核心库：WebNFC 免装 App / RN / CLI / PC-SC，驱动签名 NFC 卡 | 见仓库 |
| `libhalo-example-pcsc` (arx-research) | PC/SC 读卡 CLI 示例——RaspberryPi/PC 签到台起步 | 见仓库 |
| `poirl` (arx-research) | **「到场证明」蓝图**：芯片签名 + password 轮换防重放 + 时空校验（Solana Anchor，逻辑移植 EVM）| 见仓库 |
| `chiru-labs-PBT` (chiru-labs) | ERC-5791 物理背书 Token 官方实现（Azuki），芯片签名才能转移 | 见仓库（beta）|

---

## 维护说明

- 更新某子模块：`cd submodules/<path> && git fetch --depth 1 && git checkout <ref>`，回主仓 `git add` 记录指针。
- 移除：`git submodule deinit -f <path> && git rm -f <path>`。
- 许可证合规：本项目 Apache-2.0；子模块仅作**参考引用**（独立 repo，不并入我方代码）。若要复制 GPL/AGPL/LGPL 代码进 MyNFT 源码树，须先评估兼容性（Safe=LGPL、部分为 GPL）。
