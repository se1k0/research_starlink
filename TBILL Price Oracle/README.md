### TBill Price Oracle 合约说明  

---

### 一、合约定位

- `TBillPriceOracle` 是一个基于 `AccessControl` 的链上价格喂价合约: 
  - 维护“回合” (round)的价格数据 (`answer/startedAt/updatedAt`) 并暴露最新回合的读接口 
  - 由 `admin` 或 `operator` 更新“日更价格” 更新时内置最大偏离约束 (根据最近一次 `closeNavPrice` 约束) 
  - 允许更新 `closeNavPrice` (带偏离检查)或“手动无检查地”更新 (仅 `admin`) 作为下一次更新偏离锚点 
  - 由 `TBILL Vault` 作为 `IPriceFeed` 使用 (接口形态与 Chainlink 兼容): `latestRoundData ()` / `latestAnswer ()` / `decimals ()`

---

### 二、采用的协议 / 标准

- **AccessControl**: 基于角色的权限控制: 
  - 角色: `DEFAULT_ADMIN_ROLE`、`OPERATOR_ROLE`
  - `onlyAdmin`: 仅 `DEFAULT_ADMIN_ROLE`
  - `onlyAdminOrOperator`: `admin` 或 `operator`

- **接口兼容 (IPriceFeed 形态)**: 
  - `latestRoundData () ->  (uint80 roundId, uint256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)`
  - `latestAnswer () -> uint256`、`decimals () -> uint8`

---

### 三、核心实现

- **初始化**: 
  - 构造参数: `decimals_`、`maxPriceDeviation_` (基数 10000 对应 bps)、`initPrice_`、`closeNavPrice_`、`operator_`、`admin_`
  - 设置初始回合 (`_latestRound = 1`)、初始价格与 `closeNavPrice` 并授予角色

- **价格更新 (日更 + 偏离保护)**: 
  - `updatePrice (price)` (`admin` 或 `operator`): 
    - 调用 `_isValidPriceUpdate (price)` 以“对称相对偏差”比较 `price` 与 `_closeNavPrice`: 
      - `priceDeviation = |new - ref| /  ( (new + ref)/2) * 10^decimals`
      - 与 `_maxPriceDeviation * 10^decimals / DEVIATION_FACTOR` 比较
    - 若通过 则推进 `_latestRound` 并记录新回合数据 触发 `RoundUpdated` 与 `UpdatePrice` 事件

- **Close NAV 更新 (带/不带检查)**: 
  - `updateCloseNavPrice (price)` (`admin` 或 `operator`): 先 `UpdateCloseNavPrice` 事件 再执行偏离检查并更新 `_closeNavPrice`
  - `updateCloseNavPriceManually (price)` (仅 `admin`): 直接更新 `_closeNavPrice` 触发 `UpdateCloseNavPriceManually`

- **偏离阈值**: 
  - `updateMaxPriceDeviation (newDeviation)` (仅 `admin`): 以 bps 为单位更改最大偏离阈值

- **读取接口**: 
  - `decimals ()`、`latestRound ()`、`latestRoundData ()`、`latestAnswer ()`、`closeNavPrice ()`、`maxPriceDeviation ()`

---

### 四、与 TBILL Vault 的对接

- `TBILL Vault` 通过 `IPriceFeed.latestRoundData ()` 读取 `answer` 与 `updatedAt`: 
  - 要求 `answer >= 1e8` 且 `block.timestamp - updatedAt <= 7 days` 否则回退 
  - `tbillUsdcRate = answer * 10^underlying.decimals / 1e8`
- 因此 Oracle 侧“日更 + 偏离保护”与 Vault 侧“过期保护”共同构成价格防护

---

### 五、与官网文档的映射

- **Token Price**: TBILL 价格为 NAV/供应 (见官网“Token Price”) 链上以 `latestRoundData ().answer` 表征 USDC/USD 固定 1
  - 参考: [Token Price] (https://docs.openeden.com/tbill/token-price)

- **Redemptions**: 赎回时按“赎回份额 × 汇率 − 交易费”结算 (见官网“Redemptions”) 其中汇率来自本 Oracle (或其上游喂价服务)
  - 参考: [Redemptions] (https://docs.openeden.com/tbill/redemptions)

- **Fees**: 链上交易费/管理费计算与价格读数解耦 Oracle 专注价格 费率由 `IFeeManager` 等在 Vault 中处理
  - 参考: [Fees] (https://docs.openeden.com/tbill/fees)

- **Risks**: 价格受市场与基金净值变化影响 Oracle 通过“价格守护 (偏离阈值)+ 角色管理”降低操作风险
  - 参考: [Risks] (https://docs.openeden.com/tbill/risks)

---

### 六、事件

- `UpdatePrice (uint256 oldPrice, uint256 newPrice)`
- `UpdateCloseNavPrice (uint256 oldPrice, uint256 newPrice)`
- `UpdateCloseNavPriceManually (uint256 oldPrice, uint256 newPrice)`
- `RoundUpdated (uint80 roundId)`
- `UpdateMaxPriceDeviation (uint256 oldDeviation, uint256 newDeviation)`

---

### 七、权限与治理

- `DEFAULT_ADMIN_ROLE`: 
  - 管理 `OPERATOR_ROLE` 可 `updateMaxPriceDeviation`、`updateCloseNavPriceManually` 具备全部 Oracle 配置修改权
- `OPERATOR_ROLE`: 
  - 可 `updatePrice` 与 `updateCloseNavPrice` (受偏离保护) 适合运营日更
- 治理建议: 
  - `admin` 与关键权限应由多签/治理托管 按“On-chain/Off-chain Governance & Controls”进行流程管理与审计

---

### 八、与其它合约的关联

- `TBILL Vault`: 通过 `setTBillPriceFeed (address)` 注入本合约地址 作为价格来源
- `IFeeManager/IPartnerShipV4`: 与价格读取无直接耦合 价格仅影响 `convertTo*` 与 `totalAssets`

---

### 九、合约间调用关系与依赖

- 被 `TBILL Vault` 调用: 
  - `latestRoundData ()` (读取 `answer/updatedAt`)与 `decimals ()` 用于计算 `tbillUsdcRate ()` 
  - Vault 侧添加 7 天过期与负值防护 Oracle 侧提供最大偏离保护 两者共同形成价格守护
- 权限与治理: 
  - 价格更新 `updatePrice/updateCloseNavPrice` 由 `DEFAULT_ADMIN_ROLE/OPERATOR_ROLE` 控制 与 Vault 的所有权/KYC/暂停无直接依赖

---

### 十、目录结构

```text
TBILL Price Oracle/
  ├─ contracts/TBillPriceOracle.sol
  └─ @openzeppelin/contracts/{access/AccessControl.sol, utils/*}
```

---

### 十一、参考链接

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

