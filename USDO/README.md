### USDO 合约说明  

---

### 一、合约定位
- **USDO 是可升级的、支持 EIP-2612 的 ERC-20 代币** 采用“份额-代币”二层记账模型 支持暂停与封禁发送方 并通过角色进行精细化权限控制
- **份额 (shares)作为内部结算单位**: 用户对外看到的是代币余额 (tokens) 实际内部持仓以份额计 二者通过 `bonusMultiplier` (18 位精度)换算 以支持协议层的收益/利率/奖励等机制

---

### 二、采用的协议 / 标准
- **ERC-20 接口**: 实现 `totalSupply/balanceOf/transfer/approve/transferFrom/allowance` 等
- **EIP-2612 Permit**: 免 on-chain 授权的签名许可 (`permit/nonces/DOMAIN_SEPARATOR`) 配合 EIP-712 域
- **EIP-712**: 结构化签名域 (`EIP712Upgradeable`) 对外暴露 `eip712Domain ()`
- **AccessControl**: 基于角色的权限控制 (`DEFAULT_ADMIN_ROLE` 等)
- **Pausable**: 可暂停 暂停时所有转移/铸造/销毁动作拒绝
- **UUPS (EIP-1822 + EIP-1967 槽位)**: 实现 `upgradeTo/upgradeToAndCall` 升级权限由 `_authorizeUpgrade` 控制

---

### 三、核心实现概要
- **初始化 (initialize)**
  - 设置 `name/symbol`、初始化 `bonusMultiplier=1e18` 
  - 初始化 `AccessControl/Pausable/UUPS/EIP712` 
  - 授予 `DEFAULT_ADMIN_ROLE` 给 `owner`
- **份额-代币换算**
  - `convertToShares (amount) = amount * 1e18 / bonusMultiplier`
  - `convertToTokens (shares) = shares * bonusMultiplier / 1e18`
  - `totalSupply ()` 返回代币量 (由 `_totalShares` 换算而来)
- **转账/铸造/销毁 (均以份额流转)**
  - `_transfer`: 按代币数量换算成份额 减少发送方份额、增加接收方份额
  - `_mint/_burn`: 在 `_totalShares` 与账户 `_shares` 上以份额加减
  - `_afterTokenTransfer` 统一发出 `Transfer` 事件 (amount 为代币量)
- **安全钩子 (转账前)**
  - `_beforeTokenTransfer (from, to, amount)`: 若 `from` 被封禁或合约处于暂停态则回退 仅检查发送方 不检查接收方
- **升级授权**
  - `_authorizeUpgrade (address)`: 仅 `UPGRADE_ROLE` 可执行升级
- **EIP-2612**
  - `permit` 消费 nonce、校验签名者等于 `owner` 后 `_approve`

---

### 四、角色与权限
- **DEFAULT_ADMIN_ROLE**: 默认管理员 能管理其它角色
- **MINTER_ROLE**: 可执行 `mint`
- **BURNER_ROLE**: 可执行 `burn`
- **BANLIST_ROLE**: 可执行 `banAddresses/unbanAddresses`
- **MULTIPLIER_ROLE**: 可执行 `addBonusMultiplier` (增量更新)
- **UPGRADE_ROLE**: 可通过 UUPS 接口升级实现
- **PAUSE_ROLE**: 可执行 `pause/unpause`

---

### 五、公开状态变量与内部存储
- **`_name/_symbol` (私有存储)**: 通过 `name ()/symbol ()` 读取
- **`bonusMultiplier` (public)**: 18 位精度 最小为 `1e18`
- **`_BASE=1e18` (常量)**: 换算基数
- **`_totalShares` (私有)**: 全网份额 通过 `totalSupply ()` 以 `bonusMultiplier` 换算为代币总供给
- **`_shares[address]` (私有)**: 账户份额 通过 `balanceOf ()` 换算为代币余额
- **`_bannedList[address]` (私有)**: 账户封禁标记 仅拦截为发送方时的操作
- **`_allowances[owner][spender]` (私有)**: 标准 ERC-20 授权 (以代币量计)
- **`_nonces[address]` (私有)**: EIP-2612 nonce 计数

---

### 六、只读 (Read) 接口
- **ERC-20 基础**
  - `name () -> string`、`symbol () -> string`、`decimals () -> uint8  (固定 18)`
  - `totalSupply () -> uint256`
  - `balanceOf (address) -> uint256`
  - `allowance (address owner, address spender) -> uint256`
- **份额/换算**
  - `totalShares () -> uint256`
  - `sharesOf (address) -> uint256`
  - `convertToShares (uint256 amount) -> uint256`
  - `convertToTokens (uint256 shares) -> uint256`
- **EIP-2612 / EIP-712**
  - `nonces (address owner) -> uint256`
  - `DOMAIN_SEPARATOR () -> bytes32`
  - `eip712Domain () ->  (bytes1, string, string, uint256, address, bytes32, uint256[])`
- **合规模块**
  - `isBanned (address account) -> bool`
  - `paused () -> bool`

---

### 七、写入 (Write) 接口
- **ERC-20 基础**
  - `transfer (address to, uint256 amount) -> bool`
  - `approve (address spender, uint256 amount) -> bool`
  - `transferFrom (address from, address to, uint256 amount) -> bool`
  - `increaseAllowance (address spender, uint256 addedValue) -> bool`
  - `decreaseAllowance (address spender, uint256 subtractedValue) -> bool`
- **铸造/销毁 (需角色)**
  - `mint (address to, uint256 amount)` (仅 `MINTER_ROLE`)
  - `burn (address from, uint256 amount)` (仅 `BURNER_ROLE`)
- **封禁管理 (需角色)**
  - `banAddresses (address[] calldata)` (仅 `BANLIST_ROLE`)
  - `unbanAddresses (address[] calldata)` (仅 `BANLIST_ROLE`)
- **暂停控制 (需角色)**
  - `pause ()`、`unpause ()` (仅 `PAUSE_ROLE`)
- **激励因子 (需角色)**
  - `updateBonusMultiplier (uint256 newValue)` (仅 `DEFAULT_ADMIN_ROLE` 不得小于 `1e18`)
  - `addBonusMultiplier (uint256 increment)` (仅 `MULTIPLIER_ROLE` 增量必须 > 0)
- **EIP-2612**
  - `permit (address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)`
- **UUPS 升级 (仅代理上下文可用 + 需 `UPGRADE_ROLE`)**
  - `upgradeTo (address newImplementation)`
  - `upgradeToAndCall (address newImplementation, bytes data)`

---

### 八、事件 (Events)
- **USDO 自定义**: `AccountBanned (address)`、`AccountUnbanned (address)`、`BonusMultiplier (uint256)`、`Mint (address,uint256)`、`Burn (address,uint256)`
- **标准事件**: `Transfer (address,address,uint256)`、`Approval (address,address,uint256)`、`Paused (address)`、`Unpaused (address)`
- **升级事件 (EIP-1967)**: `Upgraded (address implementation)`

---

### 九、错误 (Errors)
- **ERC-20 相关**: `ERC20InsufficientBalance`、`ERC20InvalidSender`、`ERC20InvalidReceiver`、`ERC20InsufficientAllowance`、`ERC20InvalidApprover`、`ERC20InvalidSpender`
- **EIP-2612 相关**: `ERC2612ExpiredDeadline`、`ERC2612InvalidSignature`
- **USDO 自定义**: `USDOInvalidMintReceiver`、`USDOInvalidBurnSender`、`USDOInsufficientBurnBalance`、`USDOInvalidBonusMultiplier`、`USDOBannedSender`、`USDOInvalidBlockedAccount`、`USDOPausedTransfers`

---

### 十、交互 & 集成
- **份额换算与舍入**: 全部向下取整 长周期内可能累计至“至多 1 wei”级别的损耗 有利于协议安全
- **仅拦截发送方**: 允许向已封禁账户转入 但该地址因被封禁无法转出 (风险: 资金可能无法再转移)
- **暂停态**: `paused ()` 为真时 转移/铸造/销毁均会回退
- **授权语义**: 授权额度与转账参数均以“代币量”计 (非份额量)
- **Permit 集成**: 用 `permit` + `transferFrom` 减少一次授权交易 前端需确保 EIP-712 域与链 ID 配置一致
- **升级治理**: 建议将 `UPGRADE_ROLE` 授权给受治理/多签控制的运营账户 升级前完成测试与审计

---

### 十一、OpenEden Open Dollar  (USDO)  (Proxy)
- **共同点: EIP-1967 槽位**  
  `USDO` 采用 `UUPSUpgradeable` (内部集成 EIP-1967 槽位) `OpenEden Open Dollar  (USDO)  (Proxy)` 目录提供 `TransparentUpgradeableProxy/ProxyAdmin/UpgradeableBeacon` 等基础设施 同样基于 EIP-1967
- **部署选型**
  - **UUPS (推荐与本实现契合)**: 以 `ERC1967Proxy` 挂载 UUPS 实现 升级入口在实现合约 (`upgradeTo*`) 权限由 `UPGRADE_ROLE` 控制
  - **Transparent 模式**: 可用 `TransparentUpgradeableProxy + ProxyAdmin` 承载 UUPS 实现 此时升级由 `ProxyAdmin.owner` 发起 代理侧调用 `_upgradeToAndCallUUPS` 做安全校验
  - **Beacon 模式**: 当需要多实例共享一套实现逻辑时 使用 `UpgradeableBeacon + BeaconProxy` 统一升级
- **权限边界**
  - UUPS 下的“合约逻辑权限” (`AccessControl` 的各角色 含 `UPGRADE_ROLE`)与“代理升级权限” (`ProxyAdmin.owner` 或 `UpgradeableBeacon.owner`)应清晰隔离 推荐分别由多签/治理托管
- **实践建议**
  - 若仅需单实例 UUPS + `ERC1967Proxy` 已满足需求   
  - 若生态已有统一的 `ProxyAdmin` 运维体系 可用 Transparent 模式承载 UUPS 实现 以保持一致的升级入口

---

### 十二、cUSDO (USDO 的 ERC-4626 包装)
- **`cUSDO` 将 `USDO` 作为底层资产**: 存入 `USDO` 获得 `cUSDO` 份额 随底层资产增减体现收益
- **风险与联动**
  - `cUSDO.paused () = USDO.paused () || cUSDO.super.paused ()`: 任一侧暂停均视为暂停
  - `cUSDO._beforeTokenTransfer` 会检查 `USDO.isBanned (from)`: 被封禁地址无法转出 `cUSDO`
- **Permit 区分**
  - 针对 `USDO` 资产授权: 使用 `USDO.permit` 以便后续 `cUSDO.deposit` 等资产转移 
  - 针对 `cUSDO` 份额授权: 使用 `cUSDO.permit` 以便第三方代为 `redeem/withdraw`

---

### 十三、常见调用示例
```solidity
// 1) 授权与转账 (代币量)
USDO.approve (spender, 1000e18);
USDO.transfer (recipient, 10e18);

// 2) 免授权转账 (先前端签名 后链上提交)
USDO.permit (owner, spender, 500e18, deadline, v, r, s);
USDO.transferFrom (owner, recipient, 5e18);

// 3) 管理: 封禁与解封 (仅 BANLIST_ROLE)
USDO.banAddresses ([addr1, addr2]);
USDO.unbanAddresses ([addr1]);

// 4) 管理: 暂停/恢复 (仅 PAUSE_ROLE)
USDO.pause ();
USDO.unpause ();

// 5) 管理: 发行与销毁 (仅 MINTER_ROLE/BURNER_ROLE)
USDO.mint (to, 100e18);
USDO.burn (from, 20e18);

// 6) 管理: 收益系数 (仅 DEFAULT_ADMIN_ROLE/MULTIPLIER_ROLE)
USDO.updateBonusMultiplier (1_050000000000000000); // 1.05x
USDO.addBonusMultiplier (1_0000000000000000);      // +0.01x

// 7) 升级 (UUPS 仅代理上下文 + UPGRADE_ROLE)
USDO.upgradeTo (newImplementation);
```

---

### 十四、目录结构 & 依赖  
```text
USDO/
  ├─ contracts/tokens/USDO.sol
  └─ @openzeppelin/contracts-upgradeable/
     ├─ access/{AccessControlUpgradeable.sol, IAccessControlUpgradeable.sol}
     ├─ proxy/
     │   ├─ ERC1967/ERC1967UpgradeUpgradeable.sol
     │   └─ utils/{Initializable.sol, UUPSUpgradeable.sol}
     ├─ security/PausableUpgradeable.sol
     ├─ token/ERC20/extensions/{IERC20MetadataUpgradeable.sol, IERC20PermitUpgradeable.sol}
     └─ utils/
         ├─ cryptography/{ECDSAUpgradeable.sol, EIP712Upgradeable.sol}
         ├─ introspection/{ERC165Upgradeable.sol, IERC165Upgradeable.sol}
         ├─ math/{MathUpgradeable.sol, SignedMathUpgradeable.sol}
         ├─ {AddressUpgradeable.sol, StorageSlotUpgradeable.sol, StringsUpgradeable.sol}
         └─ CountersUpgradeable.sol
```

- **关键依赖**
  - `UUPSUpgradeable`: 提供 `upgradeTo/upgradeToAndCall` 与 `onlyProxy`/`_authorizeUpgrade` 用于 UUPS 升级
  - `ERC1967UpgradeUpgradeable`: EIP-1967 槽与升级事件 被 UUPS 复用
  - `AccessControlUpgradeable`: 角色管理 (`MINTER_ROLE/BURNER_ROLE/...`)
  - `PausableUpgradeable`: 应急暂停
  - `EIP712Upgradeable` + `ECDSAUpgradeable` + `IERC20PermitUpgradeable`: Permit 签名域与校验
  - `CountersUpgradeable`: `nonces` 计数

---

### 十五、安全注意事项
- **角色最小授权**: 分散 `UPGRADE_ROLE/PAUSE_ROLE/MINTER_ROLE` 等关键权限到多签/治理 减少单点攻击面
- **封禁语义**: 仅限制为发送方 向被封禁账户转入的资产可能无法再转出
- **升级前置**: 严格遵循测试/审计/灰度流程 避免存储布局冲突并保留 `__gap`
- **Permit 风险**: 校验签名域与链 ID 处理过期与重放 前后端域名、合约地址与链 ID 必须一致

---

### 十六、总结
- `USDO` 通过“份额-代币 + bonusMultiplier”实现可扩展的供应/收益调节 配合 `AccessControl/Pausable/Permit/UUPS` 形成完备的生产级能力
- 与 `OpenEden Open Dollar  (USDO)  (Proxy)` 的代理基建天然兼容: 可选 UUPS/Transparent/Beacon 三种部署管线 均遵循 EIP-1967 槽位与安全升级最佳实践
- `cUSDO` 作为 `USDO` 的 ERC-4626 包装 直接复用 USDO 的合规与暂停态 便于上层 DeFi 集成

