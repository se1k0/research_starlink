### TBILL Vault 合约说明  

---

### 关联与跳转

- **TBILL Price Oracle 说明**: 见 [TBILL Price Oracle/README.md](../TBILL%20Price%20Oracle/README.md)
- **TBILL Vault ERC1967Proxy 基建说明**: 见 [TBILL Vault ERC1967Proxy/README.md](../TBILL%20Vault%20ERC1967Proxy/README.md)

- **三者关系简述**: 
  - **TBILL Vault (本文件)**: 核心金库/份额代币 (ERC-20) 非 ERC-4626 实现但具备等价的存取款与价格换算接口 负责 KYC、费用、赎回队列与即时赎回、管理费计提与供应上限、UUPS 升级等
  - **TBILL Price Oracle**: 链上喂价源 负责更新 TBILL/USD 价格 (带偏离保护与权限管理) 供 Vault 的 `tbillUsdPriceFeed` 使用
  - **ERC1967/UUPS 代理基建**: 为 Vault (UUPS)提供可升级部署与升级安全的底层能力

---

### 一、合约定位

- `OpenEdenVaultV5` 是一个基于 ERC-20 的可升级金库份额合约 代币名称/符号固定为 `"OpenEden T-Bills"/"TBILL"` 底层资产为某标准 `ERC20` (如 USDC) 地址保存在 `underlying`
- 合约非 ERC-4626 标准实现 但提供 `previewDeposit/previewRedeem` 与 `_convertToShares/_convertToAssets` 等等价接口 支持“资产 ↔ 份额”的双向换算
- 存款会收取链上计算的交易费 (OpenEden 费 + 可选的合作方费) 净额划入 `treasury` 赎回支持排队 (FIFO)与“即时赎回”两种模式
- 金库价格由 `tbillUsdPriceFeed` 提供的 TBILL/USD 价格与 USDC/USD=1 (文档定义)共同决定 价格若过旧/不合法将被拒绝使用

---

### 二、采用的协议 / 模式

- **ERC-20**: 份额代币实现 转账/授权/余额等标准行为 (`ERC20Upgradeable`)
- **UUPS 升级 (EIP-1967 槽)**: 通过 `UUPSUpgradeable` 提供 `upgradeTo/upgradeToAndCall` `_authorizeUpgrade` 仅 `owner` 允许
- **OwnableUpgradeable**: `owner` 管理核心治理参数 (价格源、控制器、财库、KYC/Fee 管理等)
- **安全工具**: `SafeERC20Upgradeable` (安全转账)、`MathUpgradeable.mulDiv` (精确换算)
- **外部接口依赖**: 
  - `IPriceFeed`: 喂价接口 (`latestRoundData ()` 返回 ` (roundId,int256 answer,startedAt,updatedAt,answeredInRound)` 兼容 Chainlink 形态)
  - `IController`: 外部“暂停开关” (禁止存款/赎回)治理
  - `IKycManager`: 投资者 KYC 与封禁校验
  - `IFeeManager`: 交易费/管理费/最小额等参数源
  - `IPartnerShipV4`: 合作渠道费 (可为负值以抵扣 OpenEden 费)
  - `IRedemption`: 即时赎回路径 (如 BUIDL/USYC 置换器) 可插拔配置

---

### 三、核心实现概要

- **价格换算 (Token Price / On-Chain Oracle)**
  - `tbillUsdcRate ()` 从 `tbillUsdPriceFeed.latestRoundData ()` 读取 `answer` 与 `updatedAt` 要求 `answer >= 1e8` 且 `block.timestamp - updatedAt <= 7 days` 否则回退 (过期/非法)返回 `rate = answer * 10^underlying.decimals / 1e8`
  - `totalAssets () = totalSupply () * tbillUsdcRate () / tbillDecimalScaleFactor` 体现 NAV/供应关系 (参见官网“Token Price”章节: NAV 每份价 USDC/USD 视为 1:1)
  - 参考: `IPriceFeed` 对标“On-Chain Price Oracle”与“Price Guards” (偏离/延迟约束由 Oracle 与合约共同落实)

- **存款 (Subscriptions)**
  - `deposit (assets,receiver)`: 
    1) `Controller.requireNotPausedDeposit ()`、KYC 校验 
    2) 检查最小/首次最小额度 (`IFeeManager.getMinMaxDeposit ()` 与 `getFirstDeposit ()`) 
    3) 计算交易费 `txsFee (ActionType.DEPOSIT, sender, assets)` (OpenEden 费 + 合作方费 合并后最低不低于 `getMinTxsFee ()`) 
    4) 将 `assets - totalFee` 安全转账至 `treasury` 并按当前价格铸造份额给 `receiver`
  - 相关官网参考: [Subscriptions] (https://docs.openeden.com/tbill/subscriptions)、[Fees] (https://docs.openeden.com/tbill/fees)、[Investor Onboarding] (https://docs.openeden.com/tbill/investor-onboarding)

- **赎回队列 (Redemptions / Redemption Queue)**
  - `redeem (shares,receiver)`: 
    1) `Controller.requireNotPausedWithdraw ()`、KYC 校验 
    2) 计算对应资产 `assets=_convertToAssets (shares)` 并检查最小赎回额 (`IFeeManager.getMinMaxWithdraw ()`) 
    3) 将赎回请求编码入 `withdrawalQueue` (`DoubleQueueModified`) 按 FIFO 处理 
    4) 份额先转入金库地址 待运营 `processWithdrawalQueue (len)` 时逐笔扣费结算并转资产给 `receiver`
  - FIFO 与 T+1 典型处理期符合官网描述 (见 [Redemptions] (https://docs.openeden.com/tbill/redemptions))最小赎回额通常为 1 USDC (官网“Restrictions”) 代码侧通过 `IFeeManager` 统一约束

- **即时赎回 (Instant Redemption / Off-chain Liquidity)**
  - `redeemIns (shares,receiver)`: 
    1) 校验与费率处理后 调用 `redemptionContract.redeem (assets+1e6)` 获取 USDC 
    2) 将净额 (扣费后)直接划转给用户 超额部分通过 `offRamp` 归集至财库
  - 适用于跨场景 (链上/场外)流动性的即时满足 具体承接逻辑通过 `IRedemption` 可插拔配置

- **管理费 (Total Expense Ratio, TER)与周期 (Epoch)**
  - `updateEpoch (isWeekend)`: 满足 `lastUpdateTS + timeBuffer` 才可调用 `unClaimedFee += _calServiceFee (totalAssets, feeRate)` 每日计提 (`feeRate/365`)
  - `claimServiceFee (amount)`: `operator` 将 `unClaimedFee` 划转至 `mgtFeeTreasury`
  - 官网参考: 管理费 0.30% 年化、按日计提 (见 [Fees] (https://docs.openeden.com/tbill/fees) 的 Total Expense Ratio 部分)

- **KYC 与转账限制 (Address Level Restrictions)**
  - `_beforeTokenTransfer` 对所有非铸销/内部转账进行 KYC+封禁校验 `_validateKyc (from,to)` 同时要求 `isKyc (from && to) 且 !isBanned (from && to)`
  - 官网一致性: 仅白名单地址可发起赎回 需先完成 Onboarding (见 [Investor Onboarding] (https://docs.openeden.com/tbill/investor-onboarding)、[Redemptions] (https://docs.openeden.com/tbill/redemptions))

- **供应上限与运维工具**
  - `setTotalSupplyCap (cap)`、`mintTo/burnFrom/reIssue` (仅 `maintainer`) `offRamp/offRampQ` (仅 `operator`)
  - 外部开关 `IController` 提供应急暂停 (存款/赎回)

---

### 四、角色与权限

- **owner (Ownable)**: 设置 `FeeManager`/`KycManager`/`PriceFeed`/`Controller`/`Treasury`/`QTreasury`/`Maintainer` 等 授权 UUPS 升级 (`_authorizeUpgrade`)
- **maintainer**: `setPartnerShip/setOperator/setTimeBuffer/setTotalSupplyCap/setRedemption` 以及 `burnFrom/mintTo/reIssue` 等日常维护
- **operator**: `offRamp/offRampQ/processWithdrawalQueue/updateEpoch/claimServiceFee`
- **controller (外部合约)**: `requireNotPausedDeposit/Withdraw` 应急开关

---

### 五、公开状态变量  

- `IERC20MetadataUpgradeable public underlying`: 底层资产 (如 USDC)
- `IPriceFeed public tbillUsdPriceFeed`: 价格喂价源
- `IController public controller`、`IFeeManager public feeManager`、`IKycManager public kycManager`、`IPartnerShipV4 public partnerShip`
- `address public treasury`、`oplTreasury`、`qTreasury`、`mgtFeeTreasury`
- `bool public isWeekend`、`uint256 public timeBuffer/epoch/unClaimedFee/totalSupplyCap`

---

### 六、只读 (Read) 接口

- 资产/价格
  - `onchainAssets () -> uint256`
  - `totalAssets () -> uint256`
  - `tbillUsdcRate () -> uint256`
  - `previewDeposit (uint256 assets) -> uint256 shares`
  - `previewRedeem (uint256 shares) -> uint256 assets`

- 队列/信息
  - `getWithdrawalQueueLength () -> uint256`
  - `getWithdrawalQueueInfo (uint256 index) ->  (address sender,address receiver,uint256 shares,bytes32 id)`
  - `getWithdrawalUserInfo (address user) -> uint256 shares`

---

### 七、写入 (Write) 接口

- 用户交互
  - `deposit (uint256 assets, address receiver)`
  - `redeem (uint256 shares, address receiver)` (入队)
  - `redeemIns (uint256 shares, address receiver) -> uint256 assetsToUser` (即时赎回)

- 运营/治理
  - `processWithdrawalQueue (uint256 len)` (operator)
  - `cancel (uint256 len)` (maintainer)
  - `offRamp (uint256 amt)` / `offRampQ (address token,uint256 amt)` (operator)
  - `updateEpoch (bool isWeekend)` / `setWeekendFlag (bool)` (operator)
  - `claimServiceFee (uint256 amt)` (operator)
  - `setOplTreasury/setMgtFeeTreasury/setFeeManager/setKycManager/setTBillPriceFeed/setController/setTreasury/setQTreasury` (owner)
  - `setPartnerShip/setMaintainer/setOperator/setTimeBuffer` (maintainer/owner)
  - `setRedemption (address)` (maintainer)
  - `setTotalSupplyCap (uint256)` (maintainer)
  - `burnFrom/mintTo/reIssue` (maintainer)

- 升级 (UUPS 仅代理上下文)
  - `upgradeTo (address)`、`upgradeToAndCall (address,bytes)` (`onlyOwner`)

---

### 八、事件  

- `Deposit/Withdraw`
- `ProcessDeposit/ProcessWithdraw/ProcessRedeemCancel/ProcessWithdrawalQueue`
- `SetOplTreasury/SetMgtFeeTreasury/UpdateTreasury/UpdateQTreasury`
- `SetBuidl/SetBuidlTreasury/SetFeeManager/SetKycManager/SetMaintainer/SetTBillPriceFeed/SetController/SetPartnerShip/SetTimeBuffer`
- `UpdateEpoch/SetWeekendFlag/OffRamp/OffRampQ/ClaimServiceFee/TotalSupplyCap/BurnFrom/MintTo/ReIssue`

---

### 九、错误  

- `TBillNoPermission (address)`、`TBillInvalidateKyc (address,address)`
- `TBillLessThanFirstDeposit (uint256,uint256)`、`TBillLessThanMin (uint256,uint256)`、`TBillInvalidInput (uint256)`
- `TBillUpdateTooEarly (uint256)`、`TBillZeroAddress ()`、`TBillReceiveUSDCFailed (uint256,uint256)`
- `TBillInvalidPrice (int256)`、`TBillPriceOutdated (uint256)`、`TotalSupplyCapExceeded (uint256,uint256,uint256)`

---

### 十、交互 & 集成

- **价格与喂价**: Vault 强依赖 `tbillUsdPriceFeed` 的日更价格 官网指明 TBILL 价格等于 NAV/流通供应 (见“Token Price”) USDC/USD 固定 1:1合约侧添加 7 天过期保护与负值防护
- **赎回流程**: 默认进入 FIFO 队列 通常 T+1 美东工作日处理 即时赎回可通过 `IRedemption` 实现 (参考“Redemptions”)
- **费用**: 链上交易费 (默认 5bps)在订阅/赎回时收取 管理费按日计提 (0.30% 年化) (参考“Fees”)
- **KYC/白名单**: 地址需完成 Onboarding 后方可交互 (参考“Investor Onboarding”、“Address Level Restrictions”)
- **风控**: 外部 `Controller` 可暂停 Oracle 自带偏离保护 (详见 Oracle 文档)

---

### 十一、合约调用关系 & 依赖

- 对 `TBILL Price Oracle`: 
  - 读取 `IPriceFeed.latestRoundData ()`、`decimals ()` 以计算 `tbillUsdcRate ()` 并对价格负值/过期 (> 7 天)进行防护
- 对 `KYC Manager`: 
  - 在 `deposit/redeem/redeemIns/_beforeTokenTransfer/processWithdrawalQueue/cancel/mintTo/reIssue` 路径中调用 `isKyc/isBanned` 进行双向地址校验 
  - 通过 `setKycManager (address)` 注入依赖
- 对 `Controller`: 
  - `requireNotPausedDeposit/Withdraw` 控制申赎开关
- 对 `FeeManager/PartnerShip`: 
  - `txsFee` 计算申赎交易费 伙伴费可能为负 
  - `getMinMaxDeposit/Withdraw/getFirstDeposit/getMinTxsFee` 等参数读取
- 对 `IRedemption`: 
  - 即时赎回调用 `redeem (assets+1e6)` 完成外部置换

---

### 十二、安全注意事项

- 严格区分 `owner/maintainer/operator` 权限并建议交由多签治理 升级前务必审计与回归测试 遵循可升级存储布局规范 (`__gap`)
- 喂价来源需稳定可靠并保持日更 若长时间未更新 (>7 天)将导致价格不可用而影响交互
- 赎回与即时赎回均涉及费用结算与外部流动性对接 需关注边界与失败回退处理

---

### 十三、参考与外部文档

- 官网说明: 
  - [Introduction] (https://docs.openeden.com/tbill/introduction)
  - [Product Structuring] (https://docs.openeden.com/tbill/product-structuring)
  - [Investor Onboarding] (https://docs.openeden.com/tbill/investor-onboarding)
  - [Subscriptions] (https://docs.openeden.com/tbill/subscriptions)
  - [Redemptions] (https://docs.openeden.com/tbill/redemptions)
  - [Fees] (https://docs.openeden.com/tbill/fees)
  - [Token Price] (https://docs.openeden.com/tbill/token-price)
  - [Risks] (https://docs.openeden.com/tbill/risks)
  - [Trust & Transparency] (https://docs.openeden.com/tbill/trust-and-transparency)
  - [Smart Contract Addresses] (https://docs.openeden.com/tbill/smart-contract-addresses)
  - [On-chain Governance & Controls  (I)] (https://docs.openeden.com/tbill/on-chain-governance-and-controls-i)
  - [Off-chain Governance & Controls  (II)] (https://docs.openeden.com/tbill/off-chain-governance-and-controls-ii)
  - [FAQ] (https://docs.openeden.com/tbill/faq)

---

### 十四、目录结构 & 依赖

```text
TBILL Vault/
  ├─ contracts/OpenEdenVaultV5Impl.sol
  ├─ contracts/interfaces/{IPriceFeed.sol, IController.sol, IFeeManagerV3.sol, IKycManager.sol, IOpenEdenVaultV4.sol, IRedemption.sol, ITypes.sol}
  ├─ contracts/DoubleQueueModified.sol
  └─ @openzeppelin/contracts-upgradeable/* (ERC20Upgradeable/UUPSUpgradeable/OwnableUpgradeable 等)
```

---

### 十五、合约关键方法  

- `OpenEdenVaultV5Impl.sol`: 
  - 关键方法: `initialize`、`deposit/redeem/redeemIns`、`processWithdrawalQueue/cancel`、`txsFee`、`totalAssets/tbillUsdcRate`、`updateEpoch/claimServiceFee`、`set*`、`_beforeTokenTransfer/_authorizeUpgrade`
