### KYC Manager

---

### 一、合约定位

- `KycManager` 提供地址级合规管理: 记录地址的 KYC 类型 (`NON_KYC`/`US_KYC`/`GENERAL_KYC`)与封禁状态 仅合规且未封禁的地址可参与链上交互
- 权限模型: `Ownable` 所有管理操作仅 `owner` 可执行 方便托管至多签/治理

---

### 二、采用的协议 / 模式

- **Ownable**: `owner ()`、`onlyOwner`、`transferOwnership`、`renounceOwnership`
- **IKycManager 接口**: 
  - 枚举/结构: `KycType`、`User{kycType,isBanned}`
  - 读: `isKyc (address)`、`isBanned (address)`、`isUSKyc (address)`、`isNonUSKyc (address)`、`isStrict ()`
  - 断言: `onlyKyc (address)`、`onlyNotBanned (address)` (不满足则 revert)

---

### 三、核心实现概要

- 批量授予 KYC: `grantKycInBulk (address[] investors, KycType[] types)` (onlyOwner)→ 内部 `_grantKyc` 仅允许 `US_KYC` 或 `GENERAL_KYC` 事件 `GrantKyc`
- 批量撤回 KYC: `revokeKycInBulk (address[] investors)` (onlyOwner)→ 内部 `_revokeKyc` 置为 `NON_KYC` 事件 `RevokeKyc`
- 批量封禁/解禁: `bannedInBulk (address[])` / `unBannedInBulk (address[])` (onlyOwner)→ 内部 `_bannedInternal` 事件 `Banned`
- 严格模式: `setStrict (bool)` (onlyOwner)→ 事件 `SetStrict` `isStrict ()` 可供前端策略参考
- 只读/断言: `isKyc/isBanned/isUSKyc/isNonUSKyc/isStrict` 与 `onlyKyc/onlyNotBanned`

---

### 四、与 TBILL 系列合约的关系与调用

- 与 `TBILL Vault`: 
  - Vault 在转账/申赎等路径通过 `_validateKyc (from,to)` 强制校验: `isKyc (from && to)` 且 `!isBanned (from && to)` 
  - Vault 需在初始化或后续通过 `setKycManager (address)` 注入本合约地址 
  - 影响函数: `deposit/redeem/redeemIns/_beforeTokenTransfer/processWithdrawalQueue/cancel/mintTo/reIssue` 等

- 与 `TBILL Price Oracle`: 
  - Oracle 的喂价更新/读取不依赖 KYC (由 `AccessControl` 控制权限) 与本合约无直接调用关系

- 与 `ERC1967/UUPS 代理基建`: 
  - 生产需要采用代理升级模式 与 Vault/Oracle 的接口契约保持不变

---

### 五、只读 (Read) 接口

- `isKyc (address) -> bool`
- `isBanned (address) -> bool`
- `isUSKyc (address) -> bool`
- `isNonUSKyc (address) -> bool`
- `isStrict () -> bool`

---

### 六、写入 (Write) 接口 (onlyOwner)

- `grantKycInBulk (address[] investors, KycType[] types)`
- `revokeKycInBulk (address[] investors)`
- `bannedInBulk (address[] investors)` / `unBannedInBulk (address[] investors)`
- `setStrict (bool status)`

---

### 七、事件

- `GrantKyc (address investor, KycType)`
- `RevokeKyc (address investor, KycType)`
- `Banned (address investor, bool status)`
- `SetStrict (bool status)`

---

### 八、与源码映射

- 接口: `contracts/interfaces/IKycManager.sol`
- 实现: `contracts/KycManager.sol` (基于 OZ `Ownable`)
- Vault 使用点: `OpenEdenVaultV5Impl.sol` 的 `_validateKyc/_beforeTokenTransfer`、`setKycManager`

---

### 九、注意事项

- 以多签/治理持有 `owner` 批量 KYC/封禁要有审计记录与可追溯性
- 前端/后台需在变更后刷新本地缓存 必要时提供白名单/封禁同步接口

