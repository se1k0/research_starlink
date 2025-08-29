### OpenEden Open Dollar  (USDO) 合约说明

---

### 关联

- **USDO 合约说明**: 详见 [USDO/README.md](../USDO/README.md)
- **cUSDO 合约说明**: 详见 [cUSDO/README.md](../cUSDO/README.md)

- **三者关系简述**: 
  - **Proxy**: 提供透明代理/Beacon/UUPS 等升级与所有权管理基建 
  - **USDO**: 可由 Proxy 承载的核心代币逻辑 基于 UUPS/AccessControl/Pausable/Permit 
  - **cUSDO**: USDO 的 ERC-4626 包装 暂停/封禁与 USDO 联动 亦可由 Proxy 承载部署

  ---

### 一、合约定位
- 该目录主要包含 OpenZeppelin 标准库的非升级版组件 用于部署/管理升级代理 (Transparent Proxy、Beacon Proxy、ERC1967)、所有权控制 (Ownable)、以及底层工具 (Address、StorageSlot、Context)
- 从目录快照看 未包含具体的 `USDO` 代币实现源码 而是提供了与“Open Dollar  (USDO)”生态相关的代理与权限基建可推测 USDO 本体合约 (代币/稳定币等)会搭配此处的代理框架进行可升级部署与管理

---

### 二、采用的协议 / 模式
- **Ownable (访问控制)**
  - `contracts/access/Ownable.sol`: 单一所有者权限模型 实现 `owner`、`onlyOwner`、`transferOwnership`、`renounceOwnership`
- **EIP-1967 (代理存储槽)**
  - `contracts/interfaces/IERC1967.sol`、`contracts/proxy/ERC1967/*`: 定义与实现 EIP-1967 代理相关存储槽与升级事件
- **UUPS (EIP-1822 接口)**
  - `contracts/interfaces/draft-IERC1822.sol`: UUPS 实现需实现 `proxiableUUID`
- **Transparent Proxy 模式**
  - `contracts/proxy/transparent/TransparentUpgradeableProxy.sol` 与 `ProxyAdmin.sol`: 透明代理与管理员合约 避免选择器冲突 仅管理员账户能执行升级/换 admin 普通用户通过代理直达实现合约
- **Beacon Proxy 模式**
  - `contracts/proxy/beacon/*`: `UpgradeableBeacon` 管理实现地址 `BeaconProxy` 通过 Beacon 获取实现地址 适合多实例共享实现的升级场景
- **底层工具**
  - `Address.sol`、`StorageSlot.sol`、`Context.sol`: 安全调用、存储槽读写、上下文工具

---

### 三、核心合约与功能

1) `access/Ownable.sol`
   - 功能: 提供单所有者访问控制
   - 关键接口: 
     - `owner () -> address`
     - `transferOwnership (address newOwner)` (onlyOwner)
     - `renounceOwnership ()` (onlyOwner)
     - 修饰符: `onlyOwner`

2) `proxy/ERC1967/*`
   - `ERC1967Proxy.sol`: 基础可升级代理 遵循 EIP-1967 存储槽规范
   - `ERC1967Upgrade.sol`: 实现实现地址升级、Beacon 升级、Admin 修改等底层逻辑与事件
   - 关键点: 
     - 存储槽: `_IMPLEMENTATION_SLOT`、`_ADMIN_SLOT`、`_BEACON_SLOT`
     - 函数: `_upgradeTo`、`_upgradeToAndCall`、`_upgradeToAndCallUUPS`、`_changeAdmin`、`_upgradeBeaconToAndCall`

3) `proxy/transparent/TransparentUpgradeableProxy.sol` 与 `ProxyAdmin.sol`
   - Transparent Proxy: 
     - 非 admin 调用 → 透明转发到实现合约 
     - admin 调用 → 只能执行代理管理函数 (`upgradeTo`、`upgradeToAndCall`、`changeAdmin`、查询 `admin`/`implementation`)
   - `ProxyAdmin`: 
     - 作为代理的 admin 提供读取实现/管理员地址、变更管理员、升级实现的管理接口 自身 `onlyOwner` 控制
   - 适用: 大多数前端/用户与实现合约交互的透明化升级代理场景

4) `proxy/beacon/*`
   - `UpgradeableBeacon` (Ownable): 保存当前实现地址 `owner` 可升级实现
   - `BeaconProxy`: 从 Beacon 取实现地址进行委托调用
   - 适用: 多代理实例共享同一 Beacon 实现统一升级

5) `proxy/Proxy.sol`
   - 提供底层 `delegatecall` 代理与 `fallback/receive` 调度的抽象基类

6) 工具库
   - `Address.sol`: 安全调用封装 `functionCall`/`functionCallWithValue`/`functionStaticCall`/`functionDelegateCall`
   - `StorageSlot.sol`: 存储槽读写封装 避免内联汇编重复
   - `Context.sol`: `_msgSender ()` / `_msgData ()` 抽象

---

### 四、主要作用
- 为 `USDO` 及其相关组件提供稳定、成熟的可升级部署框架: 
  - 可选择 Transparent 模式 (配合 `ProxyAdmin`)或 Beacon 模式 (集中式升级)
  - 确保实现合约与代理之间的存储槽不冲突 升级安全合规 (EIP-1967)
- 提供基础的权限与所有权控制 (Ownable) 方便将升级权/管理权托管到多签/治理合约
- 统一的工具库支持 减少常见安全隐患 (低级调用、槽位写错等)

---

### 五、技术要点
- **透明代理与 UUPS 的差异**
  - 本目录包含 Transparent 与 Beacon 两套模式接口 如需 UUPS 则实现合约需实现 EIP-1822 接口 (`proxiableUUID`)并在升级时调用 `_upgradeToAndCallUUPS` (UUPS 逻辑通常由实现合约的 UUPS 基类提供)
  - Transparent 模式中 用户永远与实现合约对接 admin 专用于升级操作 避免选择器冲突与权限误用
- **安全升级流程**
  - 变更实现合约前 应进行全面测试与审计
  - 建议将 `ProxyAdmin` 的 `owner`、`UpgradeableBeacon` 的 `owner` 配置为多签或治理合约 并限制单点权限
- **存储槽与兼容性**
  - 遵循 EIP-1967 统一槽位 降低代理与实现间的存储冲突风险
  - 透明代理的 `admin` 与 `implementation` 可通过标准槽位 (或 `ProxyAdmin`)查询 便于链上/脚本化运维

---

### 六、只读 (Read)接口概览

- `Ownable`: 
  - `owner () -> address`

- `TransparentUpgradeableProxy` (通过内部分发接口)
  - `admin () -> address` (仅 admin 自身调用时只读 透明模式下 ABI 由接口定义)
  - `implementation () -> address` (同上)

- `ProxyAdmin`: 
  - `getProxyImplementation (ITransparentUpgradeableProxy proxy) -> address`
  - `getProxyAdmin (ITransparentUpgradeableProxy proxy) -> address`

- `UpgradeableBeacon`: 
  - `implementation () -> address`

- EIP-1967 相关: 
  - 通过标准槽位可直接 `eth_getStorageAt` 读取 `admin`/`implementation`/`beacon` 地址

---

### 七、写入 (Write)接口概览

- `Ownable`: 
  - `transferOwnership (address newOwner)` (onlyOwner)
  - `renounceOwnership ()` (onlyOwner)

- `ProxyAdmin` (onlyOwner): 
  - `changeProxyAdmin (ITransparentUpgradeableProxy proxy, address newAdmin)`
  - `upgrade (ITransparentUpgradeableProxy proxy, address implementation)`
  - `upgradeAndCall (ITransparentUpgradeableProxy proxy, address implementation, bytes data)`

- `TransparentUpgradeableProxy` (仅 admin 地址调用 且通过内部分发机制): 
  - `changeAdmin (address)`
  - `upgradeTo (address)`
  - `upgradeToAndCall (address, bytes)`

- `UpgradeableBeacon` (onlyOwner): 
  - `upgradeTo (address newImplementation)`

---

### 八、事件 (Events)
- `IERC1967`: `Upgraded (address implementation)`、`AdminChanged (address previousAdmin, address newAdmin)`、`BeaconUpgraded (address beacon)`
- `Ownable`: `OwnershipTransferred (address previousOwner, address newOwner)`
- `UpgradeableBeacon`: `Upgraded (address implementation)`

---

### 九、集成
- `USDO` 作为稳定币/代币实现: 
  - 使用 `TransparentUpgradeableProxy + ProxyAdmin` 作为默认升级栈 将 `ProxyAdmin` 的 `owner` 设为多签/治理
  - 实现合约需遵循可升级存储布局规范 保留 `__gap` 并使用 `initialize`/`reinitializer` (如采用升级版合约基类)
  - 严格区分“实现合约逻辑权限” (如 `AccessControl`/`Ownable`)与“代理升级权限” (`ProxyAdmin`/`Beacon` 的 `owner`)
- 同时管理多实例: 
  - 采用 `UpgradeableBeacon + BeaconProxy` 统一升级实现 降低运营成本
- 运维监控: 
  - 定期读取 `implementation`、`admin`、`owner` 状态 审计权限变更 对升级事件进行告警

---

### 十、常见调用示例片段

```solidity
// 读取透明代理的实现地址 (通过 ProxyAdmin 工具合约)
address impl = proxyAdmin.getProxyImplementation (ITransparentUpgradeableProxy (proxy));

// 升级实现 (仅 ProxyAdmin.owner)
proxyAdmin.upgrade (ITransparentUpgradeableProxy (proxy), newImplementation);

// Beacon 升级 (仅 UpgradeableBeacon.owner)
upgradeableBeacon.upgradeTo (newImplementation);
```

### 十一、目录结构 & 依赖

#### 目录结构  
```text
OpenEden Open Dollar  (USDO)/
  └─ @openzeppelin/contracts/
      ├─ access/Ownable.sol
      ├─ interfaces/{draft-IERC1822.sol, IERC1967.sol}
      ├─ proxy/
      │   ├─ Proxy.sol
      │   ├─ ERC1967/{ERC1967Proxy.sol, ERC1967Upgrade.sol}
      │   ├─ beacon/{IBeacon.sol, UpgradeableBeacon.sol, BeaconProxy.sol}
      │   └─ transparent/{ProxyAdmin.sol, TransparentUpgradeableProxy.sol}
      └─ utils/{Address.sol, StorageSlot.sol, Context.sol}
```

#### 合约清单与说明 (均为 OpenZeppelin 标准开源)

- `access/Ownable.sol` (标准)
  - 作用: 单一所有者权限控制 (`onlyOwner`)
  - 在本项目的目的: 为 `ProxyAdmin`、`UpgradeableBeacon` 等提供安全的管理员权限
  - 调用位置: `ProxyAdmin` 与 `UpgradeableBeacon` 继承 `Ownable` 其管理函数仅 `owner` 可调

- `interfaces/draft-IERC1822.sol` (标准接口)
  - 作用: UUPS 规范接口 (`proxiableUUID ()`)
  - 在本项目的目的: 若实现合约采用 UUPS 需要满足该接口 (本目录不直接实现 UUPS 逻辑)
  - 调用位置: 配合 `ERC1967Upgrade._upgradeToAndCallUUPS` 在实现合约侧使用

- `interfaces/IERC1967.sol` (标准接口)
  - 作用: 定义 EIP-1967 事件
  - 在本项目的目的: 为代理升级事件统一规范
  - 调用位置: 由 `ERC1967Upgrade` 在升级时发出事件

- `proxy/Proxy.sol` (标准)
  - 作用: 代理抽象基类 提供 `fallback/receive` 与 `delegatecall` 机制
  - 在本项目的目的: 作为 ERC1967/透明/Beacon 代理的基础
  - 调用位置: `ERC1967Proxy`、`BeaconProxy`、`TransparentUpgradeableProxy` 继承并实现 `_implementation ()`

- `proxy/ERC1967/ERC1967Proxy.sol` (标准)
  - 作用: 可升级代理 (EIP-1967 存储槽)
  - 在本项目的目的: 作为实现合约的代理承载 配合初始化数据进行构造
  - 调用位置: 部署代理时传入逻辑合约与初始化 calldata

- `proxy/ERC1967/ERC1967Upgrade.sol` (标准)
  - 作用: 实现地址/Beacon/管理员槽管理与升级
  - 在本项目的目的: 为代理与 UUPS 升级提供底层能力
  - 调用位置: 被 `ERC1967Proxy`、透明代理内部调度、以及 UUPS 升级流程调用

- `proxy/transparent/TransparentUpgradeableProxy.sol` (标准)
  - 作用: 透明代理 (admin 与普通用户行为分离)
  - 在本项目的目的: 常用于生产中部署可升级合约 避免选择器冲突
  - 调用位置: 用户非 admin 调用转发到实现 admin 调用执行管理指令 (`upgradeTo*`、`changeAdmin`、查询)

- `proxy/transparent/ProxyAdmin.sol` (标准 Ownable)
  - 作用: 透明代理的外部管理端 (多签/治理持有)
  - 在本项目的目的: 集中执行 `upgrade/upgradeAndCall`、`changeProxyAdmin` 等操作
  - 调用位置: `ProxyAdmin.owner` 发起升级/变更 提供 `getProxyImplementation/getProxyAdmin` 读接口

- `proxy/beacon/IBeacon.sol`、`proxy/beacon/UpgradeableBeacon.sol`、`proxy/beacon/BeaconProxy.sol` (标准)
  - 作用: Beacon 升级模式 (多代理共享实现)
  - 在本项目的目的: 当需要多实例统一升级时使用
  - 调用位置: 升级通过 `UpgradeableBeacon.upgradeTo` 代理通过 Beacon 获取实现地址

- `utils/Address.sol`、`utils/StorageSlot.sol`、`utils/Context.sol` (标准)
  - 作用: 低级调用封装、标准槽位读写、上下文
  - 在本项目的目的: 被上述模块普遍依赖 减少安全风险与重复代码

---

### 十二、自研合约与脚本

- 说明: 
  - 本目录仅包含 OpenZeppelin 标准开源合约 (代理/所有权/工具等) 用于为 USDO 生态提供可升级与权限基建 
  - 本项目自研的核心业务合约位于 `cUSDO/contracts/tokens/cUSDO.sol` 不在本目录内 
  - 若后续在本目录新增自研 USDO 实现 (如稳定币核心逻辑) 建议基于此处的 Transparent/Beacon 代理模式与 `Ownable`/多签治理进行部署与升级管理

---

### 十三、主网部署结构

- **典型结构 (与你仓库中的基建一致)**
  - 代理: `TransparentUpgradeableProxy` (代理地址 = Token 地址)
  - 管理: `ProxyAdmin` (`Ownable` 多签/治理持有)
  - 实现: ERC20 逻辑合约 (包含 `initialize`、`name/symbol/decimals`、铸造/权限等)

- **Etherscan 识别方式**
  - Contract 标签显示“Proxy / Read as Proxy / Write as Proxy”
  - 页面提供 `Implementation` 与 `Admin` 链接
  - 事件: `Upgraded (implementation)`、`AdminChanged (previousAdmin,newAdmin)`
  - 代理部署交易中 构造参数含: `logic`、`admin`、`init data (abi.encodeWithSelector (initialize,...))`

- **快速自查 5 点**
  1) 是否标记为 Proxy 且有 “Read/Write as Proxy”
  2) `Implementation` 地址源码与 ABI (是否包含 `initialize` 与 ERC20 逻辑)
  3) `Admin` 地址 (是否为多签/治理)及 `AdminChanged/Upgraded` 事件
  4) 代理构造交易的初始化 calldata 是否为 `initialize (...)` 编码
  5) 初始发行: `Transfer (address (0) -> treasury)` 是否存在 `totalSupply` 与持币地址分布是否合理

---

### 十四、为何代理地址即 Token 地址

- 用户/钱包调用的 ERC20 接口 (`totalSupply`、`balanceOf`、`transfer` 等)都由代理转发到实现合约执行 
- 因此浏览器/钱包会把“代理地址”识别为 Token 合约地址 交易与事件也记在代理地址下

---

### 十五、复刻部署步骤 (Transparent 模式)

1) 部署实现合约 (USDOImplementation)
   - 实现 `initialize (name,symbol,...)` (必要状态一次性初始化) 
   - 可选: 实现权限/铸造/黑白名单/风控等逻辑

2) 部署 `ProxyAdmin` (`Ownable`)
   - 将 `owner` 设为多签/治理账户

3) 部署 `TransparentUpgradeableProxy`
   - `_logic` = 实现地址 `admin_` = `ProxyAdmin` 地址 
   - `_data` = `abi.encodeWithSelector (USDOImplementation.initialize.selector, ...)`

4) 在 Etherscan 验证实现与代理
   - 标注 Proxy 关联 Implementation 
   - 确认 `Read/Write as Proxy` 可用

5) 发行与分发
   - 若在 `initialize` 中铸造: 检查 `Transfer (0x00 -> treasury)` 
   - 若延后铸造: 由 `onlyOwner/onlyRole (MINTER_ROLE)` 进行

6) 运营与升级
   - 如需升级: `ProxyAdmin.upgrade/upgradeAndCall` 切换实现 
   - 监听 `Upgraded/AdminChanged` 事件 并做好变更审计与告警

- 变体说明: 
  - UUPS: 仍用 EIP-1967 槽位 升级入口在实现合约 (`upgradeTo*`) `_authorizeUpgrade` 控权 
  - Beacon: `UpgradeableBeacon + BeaconProxy` 多实例共享实现 由 Beacon 统一升级

---