### USDO - cUSDO 快速导航

- **USDO（可升级 ERC-20 + Permit + 份额记账）**  
  - 作用 可升级的 ERC-20 代币，实现 EIP-2612 Permit，内部以份额记账并通过 `bonusMultiplier` 映射为对外余额；支持暂停/封禁与基于角色的权限控制
  - 关联 作为 `cUSDO` 金库的底层资产；与代理基建配合进行 UUPS/Transparent 部署与升级
  - 文档 [`USDO/README.md`](USDO/README.md)

- **cUSDO（USDO 的 ERC-4626 金库包装）**  
  - 作用 将 `USDO` 作为底层资产的 ERC-4626 金库，用户存入 `USDO` 获得 `cUSDO` 份额；支持 EIP-2612 Permit；暂停与封禁状态与 `USDO` 联动；可 UUPS 升级
  - 关联 底层依赖 `USDO`；可由代理基建承载部署与升级
  - 文档 [`cUSDO/README.md`](cUSDO/README.md)

- **OpenEden Open Dollar (USDO) (Proxy)（代理/升级基建）**  
  - 作用 基于 OpenZeppelin 的可升级基建，提供 EIP-1967、Transparent Proxy、Beacon 等组件与 `ProxyAdmin` 管理
  - 关联 用于承载并升级 `USDO`、实现合约；与 UUPS 升级模式相互兼容
  - 文档 [`OpenEden Open Dollar (USDO) (Proxy)/README.md`](OpenEden%20Open%20Dollar%20(USDO)%20(Proxy)/README.md)

- **EVM2EVMOffRamp（Chainlink CCIP 目标链执行端）**  
  - 作用 在目标链验证并执行来自源链的跨链消息与代币释放/铸造；内置聚合限速（美元价值）、路由回调与 OCR 配置
  - 关联 可将 `USDO/cUSDO` 等资产纳入 CCIP 流转规则；按需通过代理承载升级 
  - 文档 [`EVM2EVMOffRamp/README.md`](EVM2EVMOffRamp/README.md)


### 合约关系速览

- **USDO ↔ cUSDO** `cUSDO` 将 `USDO` 作为底层资产的 ERC-4626 金库；暂停与封禁联动
- **PROXY ↔ USDO** 使用 EIP-1967/Transparent/Beacon/UUPS 进行可升级部署与运维
- **EVM2EVMOffRamp ↔ 资产** 在跨链转移中按配置释放/铸造目标链资产（可包含 USDO/cUSDO）

---

### TBILL Vault 快速导航

- **TBILL Vault（OpenEden TBILL 份额金库）**  
  - 作用 基于 ERC-20 的可升级金库份额合约（非 ERC-4626），实现申购/赎回（含排队与即时赎回）、费用（交易费/管理费）、KYC 校验、价格换算（对接 Oracle）、供应上限等；UUPS 升级
  - 关联 依赖 `TBILL Price Oracle` 获取价格；依赖 `KYC Manager` 做地址合规校验；可通过代理基建承载部署升级
  - 文档 [`TBILL Vault/README.md`](TBILL%20Vault/README.md) ・ [`TBILL 在线文档精要`](TBILL%20Vault/TBILL_Online_Docs_Summary.md)

- **TBILL Vault ERC1967Proxy（TBILL 代理/升级基建说明）**  
  - 作用 说明 TBILL Vault 与 EIP-1967/Transparent/Beacon/UUPS 的关系与部署/升级建议
  - 关联 为 `TBILL Vault` 的生产部署提供推荐的代理栈与运维指引
  - 文档 [`TBILL Vault ERC1967Proxy/README.md`](TBILL%20Vault%20ERC1967Proxy/README.md)

- **TBILL Price Oracle（价格喂价）**  
  - 作用 基于 `AccessControl` 的链上喂价合约，维护每日价格回合并带最大偏离保护；对外以 Chainlink 兼容接口输出最新价格
  - 关联 被 `TBILL Vault` 读取以计算兑换比例与总资产；与 Vault 的过期保护共同形成价格守护
  - 文档 [`TBILL Price Oracle/README.md`](TBILL%20Price%20Oracle/README.md)

- **KYC Manager（地址合规模块）**  
  - 作用 记录地址的 KYC 类型与封禁状态，仅合规且未封禁地址可交互；`Ownable` 权限模型便于多签/治理托管
  - 关联 `TBILL Vault` 在转账与申赎路径中强制调用校验；通过 `setKycManager` 注入依赖
  - 文档 [`KYC Manager/README.md`](KYC%20Manager/README.md)
  
  
### 合约关系速览
  - **PROXY ↔ TBILL Vault** 使用 EIP-1967/Transparent/Beacon/UUPS 进行可升级部署与运维
  - **TBILL Vault ↔ TBILL Price Oracle/KYC Manager** Vault 读取 `TBILL Price Oracle` 的价格，强制通过 `KYC Manager` 校验地址合规

