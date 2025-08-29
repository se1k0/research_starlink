### TBILL 在线文档精要

涵盖范对应章节: Introduction / Product Structuring / Investor Onboarding / Subscriptions / Redemptions / Fees / Token Price / Risks / Trust & Transparency / Smart Contract Addresses / On-chain Governance & Controls  (I) / Off-chain Governance & Controls  (II) / FAQ

---

### 1. 产品总览 (Introduction)

- TBILL 是以美国国债 (U.S. T-Bills)为底层的链上代币化产品 定位为合规面向机构/合格投资者的“链上类基金份额”
- 投资者完成合规 Onboarding 后 可在链上以 USDC 进行申购 (Subscription)与赎回 (Redemption)
- 代币 TBILL 的每份价格 (NAV)随底层资产净值每日更新 管理费 (TER)按日计提 订阅/赎回收取交易费 (Txn Fee)
- 合约层面通过喂价 (Oracle)、KYC、暂停开关、运营角色拆分等方式提供完整的风险控制与治理路径

开发要点: 在链上交互前 需保证调用地址已通过 KYC 价格读取来自链上 Oracle 申赎路径分别支持“队列处理”与“即时赎回 (若开启)”

---

### 2. 产品结构 (Product Structuring)

- 资产持有与管理: 底层持仓为短久期美债组合 目标维持一定的存续期与流动性 收益滚动复投
- 代币经济: 
  - 代币 TBILL 表征对组合的份额 价格 (NAV)= (总资产 − 待提管理费)/ 流通供应
  - 初始价格 1.000000 (USDC 面值) 后续随 NAV 变动
- 关键参与方: 发行/投资管理人、托管机构、审计与安全服务商、合规模块 (KYC/AML)

开发要点: 链上侧关注“价格/NAV/供应/费用”的换算关系 链下侧 (资产、托管、审计)通过治理与透明度机制对外披露 不影响链上接口调用方式

---

### 3. 投资者入场 (Investor Onboarding)

- 仅白名单地址可交互: 
  - 地址需完成合规 Onboarding (KYC/AML) 通过后被加入白名单
  - 任一方向 (from/to)未通过 KYC 或被封禁 将导致转账/申赎失败
- 开发流程: 
  - 前端/后台需接入 Onboarding 流程 (通常为平台侧) 
  - 上链交互前可查询 KYC 状态 (在 Vault 的 `_beforeTokenTransfer` 强制检查)

---

### 4. 申购 (Subscriptions)

- 投资者使用 USDC 申购 TBILL: 
  - 步骤: `approve -> deposit (assets, receiver)` 
  - 金额限制: 最小申购、首次申购额 (由 FeeManager 配置) 
  - 费用: 交易费按 bps 计 (OpenEden 费 + 可能的伙伴费 最低手续费约束) 
  - 资金流向: 净额 (扣费后)进入 `treasury` 按当前价格铸造 TBILL 到 `receiver`
- 价格预览: 
  - `previewDeposit (assets) -> shares` 
  - 实际份额基于当前 `tbillUsdcRate ()` 与费用后净额

开发要点: 提交交易前可用 `previewDeposit` 估算份额 若支持签名授权/批处理 可减少额外授权交易 遵守最小金额限制 (来自 FeeManager)

---

### 5. 赎回 (Redemptions)

- 两种模式: 
  1) 队列赎回 (默认): `redeem (shares, receiver)` → 进入 FIFO 队列 运营按可用流动性逐笔处理 (通常 T+1 美东工作日)
  2) 即时赎回 (可选): `redeemIns (shares, receiver)` → 通过配置的 `IRedemption` 路径立刻置换 USDC 并结算
- 赎回结算: 
  - `USDC_Received = shares * exchangeRate - txnFee` (以处理时点的价格与费率)
  - 队列处理时价格/费率取决于处理时刻 即时赎回以 `IRedemption` 实际置换结果为准 (超额归集到财库)
- 限制与提示: 
  - 需满足最小赎回金额 (典型为 >= 1 USDC) 
  - 仅白名单地址可赎回 
  - 处理完成与否有链下通知 (平台侧) 链上以事件为准

开发要点: 
  - 队列查询: `getWithdrawalQueueLength ()`、`getWithdrawalQueueInfo (index)`、`getWithdrawalUserInfo (user)` 
  - 运营处理: `processWithdrawalQueue (len)` 
  - 取消: `cancel (len)` (维护者权限) 份额返还 
  - 即时赎回需先配置 `setRedemption (address)`

---

### 6. 费用 (Fees)

- 管理费 (TER): 年化 0.30% 按日计提 (`/365`) Vault 中以 `updateEpoch` 触发结转累计到 `unClaimedFee` 由 `claimServiceFee` 提取至管理费财库
- 交易费 (Txn Fee): 订阅与赎回均收取 默认 5 bps (可由 FeeManager 配置) 存在最低手续费 可叠加或被伙伴费 (可能为负)抵扣 合并后不得低于最低值

开发要点: 
  - 计提在 `updateEpoch` 需按 `timeBuffer` 控制频率 
  - 交易费计算使用 `txsFee (ActionType, sender, assets)` 对即时赎回/队列赎回均适用 
  - 财库地址: `mgtFeeTreasury/oplTreasury/treasury/qTreasury` 可在治理下变更

---

### 7. 价格 (Token Price)与喂价 (On-Chain Oracle)

- 价格来源: 
  - Oracle 合约 (`TBillPriceOracle`)按日更新 `answer` (TBILL/USD) 带最大偏离保护 (相对 `closeNavPrice`) 
  - Vault 侧读取 `latestRoundData ()` 要求 `answer >= 1e8` 且 `block.timestamp - updatedAt <= 7 days` 否则拒绝 
  - 汇率: `tbillUsdcRate = answer * 10^underlying.decimals / 1e8` (USDC/USD 视为 1)
- NAV 关系: 
  - `totalAssets = totalSupply * tbillUsdcRate / scale` 与“Token Price =  (TotalAssets − FeeClaimable) / CirculatingSupply”一致

开发要点: Oracle 的 `decimals` 与 `DEVIATION_FACTOR` 要求与 Vault 的换算保持一致 注意 7 天更新时效要求 (节假日留有缓冲)

---

### 8. 风险 (Risks)

- 市场与利率风险: 底层 T-Bills 的收益与价格受利率曲线影响
- 流动性与结算风险: 队列赎回依赖链上/场外流动性调度 即时赎回依赖外部兑付路径稳定性
- 价格/喂价风险: 喂价出错或不更新会影响交互 合约通过偏离保护与过期保护降低风险
- 合规风险: 仅对完成 KYC 的地址开放 跨司法辖区合规需遵守平台条款
- 智能合约风险: 建议使用多签/治理、严格升级流程与持续审计

开发要点: 在前端/SDK 中对失败路径 (暂停、喂价过期、未 KYC、额度不足)进行预检查与错误提示

---

### 9. 信任与透明 (Trust & Transparency)

- 资产透明: 定期披露资产报告、托管与审计信息 (平台侧)
- 合约透明: 全部合约开源 事件与状态可链上验证 代理/升级事件可被监控
- 治理透明: On-chain 与 Off-chain 控制权及流程清晰划分 变更可追溯

开发要点: 在运维与监控中订阅关键事件: `Upgraded/AdminChanged/BeaconUpgraded`、`Set*` 配置变更、`UpdateEpoch/ClaimServiceFee`、`ProcessWithdrawalQueue` 等

---

### 10. 合约地址 (Smart Contract Addresses)

- 主网/测试网地址由官方文档发布 开发/集成时: 
  - Vault (代理地址 = 代币地址) 
  - Oracle 地址 
  - ProxyAdmin/Beacon (如使用 Transparent/Beacon)
- 建议在部署脚本/前端配置以 `.env`/配置中心方式管理 提供网络切换与校验脚本 (读取 `implementation/admin` 槽)

---

### 11. 链上治理与控制 (On-chain Governance & Controls I)

- 角色与权限划分: 
  - Vault: `owner/maintainer/operator` + 外部 `controller` 
  - Oracle: `DEFAULT_ADMIN_ROLE/OPERATOR_ROLE` 
  - Proxy: `ProxyAdmin.owner` 或 UUPS 的实现侧 `owner`
- 关键流程: 
  - 参数更新: `set*` 系列仅限对应角色 
  - 升级: UUPS `upgradeTo*` 或 Transparent `upgrade/upgradeAndCall` 
  - 暂停: `controller` 控制 `requireNotPausedDeposit/Withdraw`

---

### 12. 链下治理与控制 (Off-chain Governance & Controls II)

- 多签/治理: 关键权限交由多签或治理模块持有 降低单点
- 审批与合规模块: 申赎、参数变更、角色授予需按既定流程审批与记录
- 审计与合规: 定期安全审计、资产审计与合规评估

开发要点: 在 CICD/运维脚本中引入“变更前后快照 + 事件校验 + 多签确认”的流水机制

---

### 13. 常见问题 (FAQ)

- 谁可以购买 TBILL？——完成 Onboarding 的合规地址
- 如何计算价格？——日更 NAV Oracle 提供 TBILL/USD USDC/USD=1
- 为什么有赎回排队？——跨链/场外流动性调度与合规流程需要 典型 T+1
- 手续费有哪些？——交易费 (订阅/赎回)与管理费 (TER 日计提)
- 是否可升级？——合约可升级 (UUPS/Transparent/Beacon) 升级前需审计与治理批准

---

### 14. 开发者速查 (接口 & 集成)

- 主要合约 & 接口: 
  - Vault: `deposit/redeem/redeemIns/preview*/totalAssets/tbillUsdcRate/txsFee/updateEpoch/processWithdrawalQueue/cancel/set*`
  - Oracle: `latestRoundData/latestAnswer/updatePrice/updateCloseNavPrice/updateMaxPriceDeviation`
  - Proxy: `upgradeTo*/admin/implementation/ProxyAdmin` 系列

- 关键依赖: 
  - `IPriceFeed` (喂价接口)、`IFeeManager` (费率/最小额)、`IKycManager` (KYC/封禁)、`IController` (暂停)、`IRedemption` (即时赎回)

- 常见失败原因与自查: 
  - `TBillInvalidateKyc`: 未完成 KYC 或被封禁 
  - `TBillPriceOutdated/TBillInvalidPrice`: 喂价过期/非法 
  - `TBillLessThanMin`: 金额不满足最小申赎限制 
  - `TotalSupplyCapExceeded`: 超过总供应上限 
  - `Controller` 暂停: 存取被禁止 
  - 代理/权限错误: 升级或管理调用由错误地址发起

---

### 15. 与源码的关键映射

- Vault: `OpenEdenVaultV5Impl.sol` 中的 `deposit/redeem/redeemIns/processWithdrawalQueue/updateEpoch/txsFee/tbillUsdcRate/_beforeTokenTransfer/_authorizeUpgrade` 等函数 对应上述流程与规则
- Oracle: `TBillPriceOracle.sol` 中的 `updatePrice/updateCloseNavPrice (_Manually)/_isValidPriceUpdate/latestRoundData` 等函数 对应日更、偏离保护、价格读取
- Proxy: `ERC1967Proxy/TransparentUpgradeableProxy/ProxyAdmin` 以及实现侧 `UUPSUpgradeable` 对应部署与升级路径

