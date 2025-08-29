### cUSDO 合约说明

---

### 关联

- **USDO 合约说明**: 详见 [USDO/README.md](../USDO/README.md)
- **Proxy 基建说明**: 详见 [OpenEden Open Dollar (USDO) (Proxy)/README.md](../OpenEden%20Open%20Dollar%20(USDO)%20(Proxy)/README.md)

- **三者关系简述**: 
  - **USDO**: 核心代币与权限/暂停/封禁/UUPS 升级逻辑实现
  - **cUSDO**: `USDO` 的 ERC-4626 金库包装 存入 `USDO` 获得 `cUSDO` 份额 暂停与封禁与 `USDO` 联动
  - **Proxy**: 提供透明代理/Beacon/UUPS 等可升级部署与所有权管理基建 可承载 `USDO`/`cUSDO` 的生产部署

  ---

### 一、合约定位
- **cUSDO 是一个可升级的 ERC-4626 金库 (Vault)份额代币** 底层资产为 `USDO`用户存入 `USDO` 获取 `cUSDO` 份额 赎回时以 `cUSDO` 换回 `USDO`
- 合约通过 ERC-4626 的份额-资产换算模型 体现底层资产变动对份额价值的影响 (若 `USDO` 在金库中的余额增长 则每份 `cUSDO` 对应的 `USDO` 数量上升)

---

### 二、采用的协议 / 标准
- **ERC-4626** (金库标准 OpenZeppelin 升级版 `ERC4626Upgradeable`)
  - 标准化存入/提取/铸造/赎回流程以及价格换算/预览/上限等接口
- **ERC-20** (由 ERC-4626 继承实现)
  - `cUSDO` 作为份额代币的基础转账、授权等标准行为
- **EIP-2612 Permit** (免 on-chain 批准的签名授权)
  - 手工实现 `permit`/`nonces`/`DOMAIN_SEPARATOR` 结合 EIP-712 域签名
- **EIP-712** (结构化签名域)
  - 由 `EIP712Upgradeable` 实现 供 Permit 使用 同时对外暴露 `eip712Domain ()`
- **UUPS (ERC-1967 槽)升级模式**
  - 通过 `UUPSUpgradeable` 提供 `upgradeTo/upgradeToAndCall` 升级权限由 `_authorizeUpgrade` 受控
- **AccessControl** (基于角色的权限控制)
  - 角色: `DEFAULT_ADMIN_ROLE`、`PAUSE_ROLE`、`UPGRADE_ROLE`
- **Pausable** (可暂停)
  - 支持暂停/恢复 并与底层 `USDO` 的暂停态联动

---

### 三、核心实现概要
- **初始化 (initialize)**
  - 设置底层资产 `USDO`、`ERC20` 名称/符号、`ERC4626` 资产、访问控制、可暂停、UUPS 与 EIP-712 域 并授予 `DEFAULT_ADMIN_ROLE` 给 `owner`
- **暂停态联动**
  - `paused ()` 返回 `USDO.paused () || cUSDO.paused ()` 即当底层或自身任一处于暂停 将整体视为暂停
- **转账前置钩子 `_beforeTokenTransfer`**
  - 若 `USDO.isBanned (from)` 则拒绝 若 `paused ()` 为真则拒绝 最后调用父类钩子
- **Permit (EIP-2612)**
  - 生成结构体哈希 计算 EIP-712 摘要 ECDSA 恢复签名者并校验为 `owner` 后 执行 `_approve`
- **升级授权**
  - `_authorizeUpgrade` 仅允许拥有 `UPGRADE_ROLE` 的账户执行升级

---

### 四、角色与权限
- **DEFAULT_ADMIN_ROLE**: 可管理其它角色 (授予/回收)
- **PAUSE_ROLE**: 可调用 `pause`/`unpause`
- **UPGRADE_ROLE**: 可通过 UUPS 接口执行实现升级

提示: 角色管理接口由 `AccessControl` 提供 (见下文写入接口)

---

### 五、公开状态变量
- **USDO**: `IUSDO public USDO;`
  - `IUSDO` 为底层资产接口 (`IERC20Metadata` + `isBanned (address)` + `paused ()`)
  - 自动生成 `USDO ()` 读取器 (external/view)

---

### 六、只读 (Read) 接口 (external/public view)

- **金库/资产相关 (ERC-4626)**
  - `asset () -> address`: 底层资产地址 (`USDO`)
  - `totalAssets () -> uint256`: 金库管理的 `USDO` 总量 (`USDO.balanceOf (address (this))`)
  - `convertToShares (uint256 assets) -> uint256`: 理想条件下 `assets` 可兑换的份额
  - `convertToAssets (uint256 shares) -> uint256`: 理想条件下 `shares` 可兑换的资产
  - `maxDeposit (address receiver) -> uint256`: `receiver` 最大可存入资产数量
  - `maxMint (address receiver) -> uint256`: `receiver` 最大可铸造份额数量
  - `maxWithdraw (address owner) -> uint256`: `owner` 最大可提取资产数量
  - `maxRedeem (address owner) -> uint256`: `owner` 最大可赎回份额数量
  - `previewDeposit (uint256 assets) -> uint256`: 存入预估份额 (不改变状态)
  - `previewMint (uint256 shares) -> uint256`: 铸造预估所需资产 (不改变状态)
  - `previewWithdraw (uint256 assets) -> uint256`: 提取预估消耗份额 (不改变状态)
  - `previewRedeem (uint256 shares) -> uint256`: 赎回预估获得资产 (不改变状态)

- **ERC-20 基础**
  - `name () -> string`、`symbol () -> string`、`decimals () -> uint8`
  - `totalSupply () -> uint256`、`balanceOf (address) -> uint256`
  - `allowance (address owner, address spender) -> uint256`

- **暂停/权限/接口**
  - `paused () -> bool`: 注意为“底层或自身”的合并暂停态
  - `hasRole (bytes32 role, address account) -> bool`
  - `getRoleAdmin (bytes32 role) -> bytes32`
  - `supportsInterface (bytes4 interfaceId) -> bool` (ERC165)

- **EIP-2612 / EIP-712**
  - `nonces (address owner) -> uint256`: 当前 Permit nonce
  - `DOMAIN_SEPARATOR () -> bytes32`
  - `eip712Domain () ->  (bytes1 fields, string name, string version, uint256 chainId, address verifyingContract, bytes32 salt, uint256[] extensions)`

- **底层资产/合规信息** (通过 `USDO`)
  - `USDO.isBanned (address) -> bool` (由底层合约实现)
  - `USDO.paused () -> bool` (由底层合约实现)

---

### 七、写入 (Write) 接口 (external/public)

- **金库流转 (ERC-4626)**
  - `deposit (uint256 assets, address receiver) -> uint256 shares`
    - 将 `assets` 数量的 `USDO` 存入金库 给 `receiver` 铸造对应 `shares`
    - 需先在 `USDO` 上 `approve (cUSDO, assets)` 或使用 `USDO` 的 `permit` 再 `deposit` (若支持)
  - `mint (uint256 shares, address receiver) -> uint256 assets`
    - 按目标 `shares` 计算所需资产 `assets` 并存入
  - `withdraw (uint256 assets, address receiver, address owner) -> uint256 shares`
    - 从 `owner` 扣除份额 向 `receiver` 提取指定 `assets` 若 `msg.sender != owner` 需要足额份额授权
  - `redeem (uint256 shares, address receiver, address owner) -> uint256 assets`
    - 从 `owner` 销毁 `shares` 向 `receiver` 提取对应 `assets`

- **ERC-20 基础**
  - `transfer (address to, uint256 amount) -> bool`
  - `approve (address spender, uint256 amount) -> bool`
  - `transferFrom (address from, address to, uint256 amount) -> bool`
  - `increaseAllowance (address spender, uint256 addedValue) -> bool`
  - `decreaseAllowance (address spender, uint256 subtractedValue) -> bool`

- **EIP-2612 Permit**
  - `permit (address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)`
    - 使用签名在链上直接设置 `allowance (owner, spender) = value`
    - 会消费 `owner` 的当前 `nonce` `deadline` 过期会回退 签名者需等于 `owner`

- **暂停控制 (ROLE)**
  - `pause ()` (仅 `PAUSE_ROLE`)
  - `unpause ()` (仅 `PAUSE_ROLE`)

- **角色管理 (AccessControl)**
  - `grantRole (bytes32 role, address account)` (调用者需拥有 `getRoleAdmin (role)`)
  - `revokeRole (bytes32 role, address account)` (调用者需拥有 `getRoleAdmin (role)`)
  - `renounceRole (bytes32 role, address account)` (仅账号本人才可放弃其自身角色)

- **合约升级 (UUPS 仅代理下可用)**
  - `upgradeTo (address newImplementation)` (仅代理上下文 + `UPGRADE_ROLE`)
  - `upgradeToAndCall (address newImplementation, bytes data)` (仅代理上下文 + `UPGRADE_ROLE`)
  - 说明: UUPS 要求通过代理调用 实现合约本体直接调用会被 `onlyProxy` 拒绝

---

### 八、事件 (Events)
- 来自 ERC-20: `Transfer (address from, address to, uint256 value)`、`Approval (address owner, address spender, uint256 value)`
- 来自 ERC-4626: `Deposit (address sender, address owner, uint256 assets, uint256 shares)`、`Withdraw (address sender, address receiver, address owner, uint256 assets, uint256 shares)`
- 来自 AccessControl: `RoleGranted`、`RoleRevoked`、`RoleAdminChanged`
- 来自 Pausable: `Paused (address account)`、`Unpaused (address account)`
- 来自 ERC1967: `Upgraded (address implementation)` (升级时触发)

---

### 八、错误 (Errors)
- 自定义错误 (cUSDO 定义): 
  - `ERC2612ExpiredDeadline (uint256 deadline, uint256 blockTimestamp)`: Permit 过期
  - `ERC2612InvalidSignature (address signer, address owner)`: 签名者非 owner
  - `cUSDOBlockedSender (address sender)`: 发送地址被底层 `USDO` 封禁
  - `cUSDOPausedTransfers ()`: 当前处于暂停态 禁止转账/铸销

---

### 九、交互 & 集成
- **存入 (获得份额)**
  -  (方式一)先在 `USDO` 上 `approve` 后 `deposit` 或 `mint`
  -  (方式二)若前端支持 `USDO` 的 `permit` 可先 `permit` 后 `deposit` 减少授权交易
- **提取/赎回**
  - 第三方代为提取/赎回需要 `owner` 对份额的授权 (`approve` `cUSDO` 份额 或使用 `permit` 直接授权份额—注意这里的 `permit` 属于 `cUSDO` 的 ERC20 授权)
- **价格/预览**
  - 使用 `preview*` 系列函数进行不改状态的换算预估 避免滑点/边界导致交易回退
- **暂停/封禁检查**
  - 转账类操作均经过 `_beforeTokenTransfer` 的封禁与暂停校验 调用方应提前检查 `paused ()` 与底层 `USDO` 的状态 或处理回退
- **升级**
  - 仅通过代理、由 `UPGRADE_ROLE` 持有者调用 `upgradeTo/upgradeToAndCall` 升级前建议遵循审计/多签流程

---

### 十、安全注意事项
- 正确配置并分散角色 (尤其是 `UPGRADE_ROLE` 与 `PAUSE_ROLE`) 避免单点风险
- 对 `permit` 的签名来源进行严格校验 确保前端 EIP-712 域与链 ID 配置一致
- ERC-4626 使用了 v4.9 的虚拟资产/份额机制以缓解初期“捐赠攻击” 但仍应关注极端情况下的价格操纵与预言机/侧路风险
- 若底层 `USDO` 发生暂停或封禁 `cUSDO` 的流转会相应受限 集成方需做好失败处理与重试逻辑

---

### 十一、常见调用示例片段

```solidity
// 读取份额与资产的即时换算
uint256 shares = cUSDO.convertToShares (1e18);
uint256 assets = cUSDO.convertToAssets (shares);

// 预览用户计划存入 100 USDO 将获得多少 cUSDO 份额
uint256 preview = cUSDO.previewDeposit (100e18);

// 通过 permit 授权 (前端获取签名后)
cUSDO.permit (owner, spender, value, deadline, v, r, s);

// 存入资产获取份额 (需先在 USDO 上授权给 cUSDO)
cUSDO.deposit (100e18, receiver)
```

---

### 十二、目录结构 & 依赖

#### 目录结构
```text
cUSDO/
  ├─ contracts/
  │   └─ tokens/
  │       └─ cUSDO.sol
  └─ @openzeppelin/
      └─ contracts-upgradeable/
          ├─ access/
          │   ├─ AccessControlUpgradeable.sol
          │   └─ IAccessControlUpgradeable.sol
          ├─ interfaces/
          │   ├─ draft-IERC1822Upgradeable.sol
          │   ├─ IERC1967Upgradeable.sol
          │   ├─ IERC4626Upgradeable.sol
          │   └─ IERC5267Upgradeable.sol
          ├─ proxy/
          │   ├─ beacon/IBeaconUpgradeable.sol
          │   ├─ ERC1967/ERC1967UpgradeUpgradeable.sol
          │   └─ utils/{Initializable.sol, UUPSUpgradeable.sol}
          ├─ security/PausableUpgradeable.sol
          ├─ token/ERC20/
          │   ├─ ERC20Upgradeable.sol
          │   ├─ extensions/{ERC4626Upgradeable.sol, IERC20MetadataUpgradeable.sol, IERC20PermitUpgradeable.sol}
          │   └─ IERC20Upgradeable.sol
          └─ utils/
              ├─ {AddressUpgradeable.sol, StorageSlotUpgradeable.sol, ContextUpgradeable.sol, StringsUpgradeable.sol}
              ├─ cryptography/{ECDSAUpgradeable.sol, EIP712Upgradeable.sol}
              ├─ introspection/{ERC165Upgradeable.sol, IERC165Upgradeable.sol}
              ├─ math/{MathUpgradeable.sol, SignedMathUpgradeable.sol}
              └─ CountersUpgradeable.sol
```

#### 依赖与用途 (均为标准开源: OpenZeppelin v4.x Upgradeable)
- `UUPSUpgradeable.sol` (标准): 提供 `upgradeTo/upgradeToAndCall` 与 `onlyProxy`、`_authorizeUpgrade` 钩子
  - 作用: 实现 UUPS 升级能力
  - 调用位置: 外部由具有 `UPGRADE_ROLE` 的管理员 (通过代理)调用 `upgradeTo*` 内部 `_authorizeUpgrade` 在 `cUSDO._authorizeUpgrade` 中受 `onlyRole (UPGRADE_ROLE)` 控制

- `ERC1967UpgradeUpgradeable.sol` (标准): EIP-1967 槽管理、升级事件
  - 作用: 被 UUPS 基类复用 存储实现地址/管理员/Beacon 槽位
  - 调用位置: 由 `UUPSUpgradeable` 升级流程间接调用

- `Initializable.sol` (标准): initializer/reinitializer 语义
  - 作用: `initialize (IUSDO,address)` 初始化模块
  - 调用位置: 部署后通过代理构造交易或初始化函数调用

- `AccessControlUpgradeable.sol` (标准): 基于角色的访问控制
  - 作用: 定义 `DEFAULT_ADMIN_ROLE`、`PAUSE_ROLE`、`UPGRADE_ROLE`
  - 调用位置: `initialize` 时 `_grantRole (DEFAULT_ADMIN_ROLE, owner)` `pause/unpause`/`_authorizeUpgrade` 使用 `onlyRole`

- `PausableUpgradeable.sol` (标准): 可暂停开关
  - 作用: `pause/unpause` 控制 `paused ()` 被覆写叠加底层 `USDO.paused ()`
  - 调用位置: `pause ()`、`unpause ()`、`_beforeTokenTransfer` 中 `paused ()` 检查

- `ERC4626Upgradeable.sol` (标准): 金库份额标准
  - 作用: 提供 `deposit/mint/withdraw/redeem`、`convertTo*`、`preview*`、`max*`
  - 调用位置: 对外直接暴露供集成方交互 内部 `_deposit/_withdraw` 完成资产/份额流转

- `EIP712Upgradeable.sol` + `ECDSAUpgradeable.sol` (标准): EIP-712 域与签名恢复
  - 作用: 为 EIP-2612 `permit` 生成域分隔符与签名校验
  - 调用位置: `permit ()` 中调用 `_hashTypedDataV4` 与 `ECDSAUpgradeable.recover`

- `IERC20PermitUpgradeable.sol` (标准接口): Permit 规范接口
  - 作用: 实现 `permit/nonces/DOMAIN_SEPARATOR`
  - 调用位置: 外部合约/前端使用 `permit` 设置授权

- `ERC20Upgradeable.sol`、`IERC20Upgradeable.sol`、`IERC20MetadataUpgradeable.sol` (标准): ERC-20 基础能力
  - 作用: `cUSDO` 作为份额代币 具备余额、转账、授权、名称符号小数
  - 调用位置: 对外标准 ERC-20 交互

- 其它工具 (`AddressUpgradeable`、`StorageSlotUpgradeable`、`ContextUpgradeable`、`StringsUpgradeable`、`MathUpgradeable`、`CountersUpgradeable`、`ERC165Upgradeable` 等) (标准)
  - 作用: 通用工具/接口支持 (nonce 计数、数学、上下文、接口探测等)
  - 调用位置: `_nonces` 使用 `CountersUpgradeable` 多处内部依赖

#### 自定义/本地接口
- `IUSDO` (非 OZ 标准): `isBanned (address)`、`paused ()`、`IERC20Metadata` 
  - 作用: 底层资产与合规联动
  - 调用位置: `paused ()` 合并底层暂停 `_beforeTokenTransfer` 检查封禁

---

### 十三、cUSDO.sol

- `contracts/tokens/cUSDO.sol`
  - 功能概述: 
    - 基于 `ERC4626Upgradeable` 的 `USDO` 包装金库 份额代币为 `cUSDO`
    - 与底层 `USDO` 的暂停态与封禁名单联动 (`paused ()` 叠加 `_beforeTokenTransfer` 检查 `isBanned (from)`)
    - 提供 `EIP-2612 permit` 以 EIP-712 域签名实现免 gas 授权 (本地实现 `nonces/DOMAIN_SEPARATOR/permit`)
    - 采用 `AccessControlUpgradeable` 定义治理与运维角色 (`DEFAULT_ADMIN_ROLE`、`PAUSE_ROLE`、`UPGRADE_ROLE`)
    - 采用 `PausableUpgradeable` 作为应急开关 
    - 采用 `UUPSUpgradeable` 作为升级机制 `_authorizeUpgrade` 受 `UPGRADE_ROLE` 限制
  - 关键方法: `initialize`、`deposit/mint/withdraw/redeem`、`permit`、`pause/unpause`、`_authorizeUpgrade`、`paused` (override)、`_beforeTokenTransfer` (override)

