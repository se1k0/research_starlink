### TBILL Vault ERC1967Proxy 合约说明 (代理/升级基建)

---

### 一、合约定位

- 提供三套常用可升级模式与底层基建: 
  - **EIP-1967 槽位与基础代理**: `proxy/ERC1967/{ERC1967Proxy.sol, ERC1967Upgrade.sol}`
  - **透明代理 (Transparent)**: `proxy/transparent/{TransparentUpgradeableProxy.sol, ProxyAdmin.sol}`
  - **Beacon 模式**: `proxy/beacon/{UpgradeableBeacon.sol, BeaconProxy.sol}`
- 其它: `access/Ownable.sol`、`proxy/Proxy.sol`、`utils/{Address.sol, StorageSlot.sol, Context.sol}`

---

### 二、采用的协议 / 模式

- **EIP-1967**: 统一实现地址/Admin/Beacon 的存储槽规范与升级事件 避免与实现合约存储冲突
- **透明代理 (Transparent)**: Admin 行为与用户行为分离 (Admin 仅可执行管理函数 普通用户直达实现合约)
- **Beacon**: 多代理共享单一实现地址 由 Beacon 统一升级
- **UUPS (EIP-1822)**: 实现合约自身提供升级入口 (`proxiableUUID`) 由实现侧的 `UUPSUpgradeable` 负责安全检查

---

### 三、与 TBILL Vault 的关系

- `TBILL Vault` 采用 **UUPS** 升级模式 (合约内继承 `UUPSUpgradeable` 并实现 `_authorizeUpgrade`)
- 部署建议: 
  - 使用 `ERC1967Proxy` 承载 `OpenEdenVaultV5` 实现地址 并在构造中通过 `_data` 进行 `initialize (...)` 一次性初始化 
  - 或者使用 Transparent Proxy 也同样基于 EIP-1967 槽位 但注意 Transparent 的 Admin 仅用于升级与管理 不可直接调用实现逻辑 
  - 多实例统一升级的场景可考虑 Beacon (`UpgradeableBeacon + BeaconProxy`)
- 升级建议: 
  - 对于 UUPS 调用实现合约的 `upgradeTo/upgradeToAndCall` (需通过代理调用 且实现侧会在 `_authorizeUpgrade` 中受控 `TBILL Vault` 为 `onlyOwner`)
  - 对于 Transparent 建议借助 `ProxyAdmin` 执行 `upgrade/upgradeAndCall` `ProxyAdmin.owner` 置为多签/治理

---

### 四、核心组件 & 函数  

- `proxy/ERC1967/ERC1967Proxy.sol`
  - 作用: EIP-1967 基础代理 构造参数: `_logic` (实现地址)、`_data` (初始化 calldata)
  - 关键: `_implementation ()` 读取实现槽 构造时 `_upgradeToAndCall (_logic, _data, false)`

- `proxy/ERC1967/ERC1967Upgrade.sol`
  - 作用: 实现/Beacon/Admin 槽的读写与升级事件
  - 关键函数: `_upgradeTo`、`_upgradeToAndCall`、`_upgradeToAndCallSecure`、`_changeAdmin`、`_upgradeBeaconToAndCall`

- `proxy/transparent/TransparentUpgradeableProxy.sol` 与 `ProxyAdmin.sol`
  - 透明代理: 非 Admin 调用 → 代理转发 Admin 调用 → 仅能调用管理函数 (`changeAdmin/upgradeTo/upgradeToAndCall/admin/implementation`)
  - `ProxyAdmin` (Ownable): 外部管理端 `owner` 可 `upgrade/upgradeAndCall`、`changeProxyAdmin` 并查询实现/Admin

- `proxy/beacon/*`
  - `UpgradeableBeacon` (Ownable): 保存实现地址 `owner` 可 `upgradeTo (newImplementation)`
  - `BeaconProxy`: 从 Beacon 获取实现地址并转发

---

### 五、安全 & 运维

- 严格区分“实现合约逻辑权限” (`owner/maintainer/operator`)与“代理升级权限” (`ProxyAdmin.owner` 或实现侧 UUPS 升级权限)
- 升级前进行审计/回归 遵循存储布局兼容性 (保留 `__gap`)
- UUPS: 实现侧仅 `owner` 可升级 `owner` 建议为多签/治理 
- Transparent: 使用 `ProxyAdmin` 其 `owner` 为多签/治理 避免直接用 EOA 做 Admin

---

### 六、只读 (Read) 接口

- 透明代理 (仅 Admin)
  - `admin () -> address`
  - `implementation () -> address`

- ProxyAdmin (onlyOwner)
  - `getProxyImplementation (address proxy) -> address`
  - `getProxyAdmin (address proxy) -> address`

---

### 七、写入 (Write) 接口

- 透明代理 (仅 Admin)
  - `changeAdmin (address newAdmin)`
  - `upgradeTo (address newImplementation)`
  - `upgradeToAndCall (address newImplementation, bytes data)`

- ProxyAdmin (onlyOwner)
  - `upgrade (address proxy, address implementation)`
  - `upgradeAndCall (address proxy, address implementation, bytes data)`
  - `changeProxyAdmin (address proxy, address newAdmin)`

---

### 八、与官网文档的对应

- 官网强调“Trust & Transparency / On-chain Governance & Controls”: 建议以多签/治理托管升级与关键参数权限 链上事件 (`Upgraded/AdminChanged/BeaconUpgraded`)可用作运维审计与监控
  - 参考: 
    - [Trust & Transparency] (https://docs.openeden.com/tbill/trust-and-transparency)
    - [On-chain Governance & Controls  (I)] (https://docs.openeden.com/tbill/on-chain-governance-and-controls-i)
    - [Off-chain Governance & Controls  (II)] (https://docs.openeden.com/tbill/off-chain-governance-and-controls-ii)

---

### 九、目录结构  

```text
TBILL Vault ERC1967Proxy/
  └─ @openzeppelin/contracts/
      ├─ access/Ownable.sol
      ├─ proxy/
      │   ├─ Proxy.sol
      │   ├─ ERC1967/{ERC1967Proxy.sol, ERC1967Upgrade.sol}
      │   ├─ transparent/{TransparentUpgradeableProxy.sol, ProxyAdmin.sol}
      │   └─ beacon/{IBeacon.sol, UpgradeableBeacon.sol, BeaconProxy.sol}
      └─ utils/{Address.sol, StorageSlot.sol, Context.sol}
```

---

### 十、与 TBILL 合约间的依赖说明

- 对 `TBILL Vault`: 提供可升级承载 (UUPS/Transparent/Beacon)与事件 Vault 实现侧的 `owner` 或 `ProxyAdmin.owner` 应为多签/治理
- 对 `TBILL Price Oracle` 与 `KYC Manager`: 
  - 这两者与代理无直接耦合 但在生产中同样可以采用代理升级 (若未来需要) 
  - 与 Vault 的依赖通过 Vault 的 `set*` 注入 (如 `setTBillPriceFeed/setKycManager`) 不经由代理直接耦合

