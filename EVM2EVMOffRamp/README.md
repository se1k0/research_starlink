### EVM2EVMOffRamp 合约说明

---

### 关联
- **USDO 合约说明**: 详见 [USDO/README.md](../USDO/README.md)
- **cUSDO 合约说明**: 详见 [cUSDO/README.md](../cUSDO/README.md)
- **Proxy 基建说明**: 详见 [OpenEden Open Dollar (USDO) (Proxy)/README.md](../OpenEden%20Open%20Dollar%20(USDO)%20(Proxy)/README.md)

- **关系简述**: 
  - **EVM2EVMOffRamp**: 跨链消息在目标链的验证与执行端 负责代币释放/铸造与回调路由 
  - **Proxy**: 可用于承载 OffRamp 的升级部署 (若按生产需要代理化) 
  - **USDO/cUSDO**: 可以作为 CCIP 转移中的目标资产之一 遵循本合约的池/价值/限速规则

---

### 一、合约定位
- **EVM2EVMOffRamp 是 Chainlink CCIP (跨链通信)执行端 (OffRamp)合约** 用于在目标链上按顺序批量执行来自源链的消息与代币转移
- 与 OnRamp/CommitStore 共同构成跨链执行单元: 
  - OnRamp (源链)负责打包消息并计算叶子哈希 
  - CommitStore (目标链)存放已提交的消息默克尔根 
  - OffRamp (目标链 即本合约)基于证明校验并执行消息 (合约调用与代币释放/铸造)

---

### 二、采用的协议 / 标准 / 依赖
- **CCIP 内部协议与结构体**: `Client`, `Internal`, `Pool`, `RateLimiter` 等库定义消息格式、默克尔证明、池子交互与速率限制
- **OCR (Offchain Reporting)执行框架**: 继承 `OCR2BaseNoChecks`/`OCR2Abstract` 用于 DON 执行调度与配置 (节省签名验证成本)
- **EIP-165**: 通过 `ERC165Checker` 在运行时检查池接口支持性 (`CCIP_POOL_V1`)
- **OpenZeppelin 实用库**: `EnumerableMap` 等用于可枚举映射结构
- **自定义访问控制**: `OwnerIsCreator/ConfirmedOwner*` 简化 Owner 管理 部分管理员权限 (如限速 Token 配置)走 onlyOwner

---

### 三、核心实现概要
- **消息验证 & 执行**
  - 通过 `CommitStore.verify (hashedLeaves, proofs, proofFlagBits)` 验证消息集合已被提交 (返回提交时间戳)
  - 对每条消息计算 `Internal._hash (message, i_metadataHash)` 断言与 `messageId` 一致
  - 使用位图记录消息 `sequenceNumber` 的执行状态: `UNTOUCHED/IN_PROGRESS/SUCCESS/FAILURE` 避免重入与重复执行
  - 非手动模式 (DON 执行): 只允许 `UNTOUCHED` → `SUCCESS/FAILURE` 
  - 手动模式 (`manuallyExecute`)可在超时后或上次失败后重试 允许覆盖 gas 限制但不可下降

- **顺序 & 幂等**
  - 维护 `s_senderNonce[sender]` 确保按 nonce 顺序执行 (升级切换时可回溯旧 OffRamp 的 nonce)
  - 已执行/正在执行的消息被跳过或拒绝 保证幂等性与批量稳定性

- **代币释放 & 铸造 (Pools)**
  - 通过 `ITokenAdminRegistry.getPool (localToken)` 获取目标链池地址 使用 `ERC165Checker` 校验池支持 `CCIP_POOL_V1`
  - 以“精确 gas + 返回值长度上限”调用 `IPoolV1.releaseOrMint` 并核对接收者前后余额差等于返回的 `destinationAmount`
  - 汇总本次转账中被纳入聚合限速的 token 的美元价值 调用 `AggregateRateLimiter._rateLimitValue (value)` 进行消耗

- **聚合限速 (Aggregate Rate Limiter)**
  - 采用 Token Bucket 模型 按“美元计价总价值”限速: 
    - 通过 `IPriceRegistry.getTokenPrice (token)` 获取 1e18 精度的美元价 
    - 使用 `USDPriceWith18Decimals._calcUSDValueFromTokenAmount` 计算价值 
    - 只对被纳入白名单的 token (dest->source 映射)统计价值并消耗桶
  - 支持 Owner 或 Admin 调整速率配置与纳管 token 集合

- **路由 & 回调**
  - 调用 `IRouter.routeMessage` 将消息分发给目标合约 `receiver` 可附带 `destTokenAmounts` 与回调 gas 限制 
  - 若 `receiver` 非合约或不支持 `IAny2EVMMessageReceiver` 且 `data` 为空且 `gasLimit` 为 0 则仅转账代币不回调

---

### 四、公开状态与配置
- **静态配置 (构造后不可变)**: `commitStore`、`onRamp`、`chainSelector/sourceChainSelector`、`prevOffRamp`、`rmnProxy`、`tokenAdminRegistry`
- **动态配置 (可变)**: `permissionLessExecutionThresholdSeconds` (手动执行开启阈值)、`maxDataBytes`、`maxNumberOfTokensPerMsg`、`router`、`priceRegistry`
- **聚合限速白名单**: `getAllRateLimitTokens () ->  (sourceTokens[], destTokens[])` 返回被纳入限速的代币集合

---

### 五、只读 (Read) 接口
- `typeAndVersion () -> string`: 版本标识 ("EVM2EVMOffRamp 1.5.0")
- `getSenderNonce (address sender) -> uint64`: 返回 sender 当前 nonce (必要时回溯旧 OffRamp)
- `getStaticConfig () -> StaticConfig`: 返回静态配置
- `getDynamicConfig () -> DynamicConfig`: 返回动态配置
- `getAllRateLimitTokens () ->  (address[] sourceTokens, address[] destTokens)`: 返回纳管的限速 token 集合
- `currentRateLimiterState () -> RateLimiter.TokenBucket` (继承自 `AggregateRateLimiter`)

---

### 六、写入 (Write) 接口
- **OCR/DON 执行入口**: 
  - `_report (bytes report)` (internal OCR 调用路径) → `_execute (ExecutionReport, [])`
  - `setOCR2Config (...)` (继承自 OCR 基类 onlyOwner)设置 OCR2 配置 (签名者、发送者、on/off-chain 配置等)
- **手动执行**: 
  - `manuallyExecute (ExecutionReport report, GasLimitOverride[] overrides)`: 超时或失败后允许人工重试 gas 仅可上调
- **聚合限速**: 
  - `setRateLimiterConfig (RateLimiter.Config)` (onlyAdminOrOwner 来自 `AggregateRateLimiter`)
  - `updateRateLimitTokens (RateLimitToken[] removes, RateLimitToken[] adds)` (onlyOwner)
- **Admin 管理** (继承自 `AggregateRateLimiter`): 
  - `setAdmin (address newAdmin)` (onlyAdminOrOwner)

---

### 七、事件 (Events)
- 核心: 
  - `ConfigSet (StaticConfig, DynamicConfig)`: 动态配置更新 
  - `ExecutionStateChanged (uint64 sequenceNumber, bytes32 messageId, MessageExecutionState state, bytes returnData)` 
  - `SkippedIncorrectNonce (uint64 nonce, address sender)`、`SkippedSenderWithPreviousRampMessageInflight (uint64 nonce, address sender)`、`SkippedAlreadyExecutedMessage (uint64 sequenceNumber)`、`AlreadyAttempted (uint64 sequenceNumber)` 
  - `TokenAggregateRateLimitAdded (address sourceToken, address destToken)`、`TokenAggregateRateLimitRemoved (address sourceToken, address destToken)`
- OCR 相关 (继承): `ConfigSet` (OCR2Abstract)、`Transmitted`
- 速率限制 (继承): `TokensConsumed (uint256 tokens)`、`ConfigChanged (Config)`、`AdminSet (address)`

---

### 八、错误 (Errors)
- 流程类: `ZeroAddressNotAllowed`、`CommitStoreAlreadyInUse`、`EmptyReport`、`RootNotCommitted`、`InvalidMessageId`、`InvalidNewState`、`ManualExecutionNotYetEnabled`、`ManualExecutionGasLimitMismatch`、`InvalidManualExecutionGasLimit`、`InvalidTokenGasOverride`、`ReceiverError (bytes)`、`TokenHandlingError (bytes)`、`ReleaseOrMintBalanceMismatch`、`InvalidDataLength`、`UnexpectedTokenData`、`DestinationGasAmountCountMismatch`
- 限速类 (库内): `AggregateValueMaxCapacityExceeded`、`AggregateValueRateLimitReached`、`InvalidRateLimitRate` 等
- 配置/OCR 类 (继承): `InvalidConfig`、`ConfigDigestMismatch`、`UnauthorizedTransmitter`、`ForkedChain`、`WrongMessageLength` 等

---

### 九、主要作用
- **跨链消息的安全落地执行**: 通过证明验证 + 状态机位图 + 顺序控制 确保消息只执行一次 且按序可控
- **代币释放/铸造的余额校验**: 对池返回值做长度与余额差校验 防止池实现异常或回传数据攻击
- **抗 Gas/Revert 攻击**: 
  - `CallWithExactGas (_SafeReturnData)` 限制返参长度、保证最小 gas 检查 防止“返回值炸弹”与 gas 欠供 
  - 对 `routeMessage` 结果失败回滚 保持原子性
- **价格化速率限制**: 按美元价值限速 统一跨 Token 价值度量 防止大额跨链流动引发风险
- **OCR2 无签名模式**: 通过 CommitStore 的可验证性 OCR 报告不再做链上签名校验 降低成本 (NoChecks)
- **零停机升级衔接**: `prevOffRamp` + nonce 回溯 允许 V1/V2 切换期间保障消息序列连续性

---

### 十、常见调用示例片段
```solidity
// 1) 动态配置下发 (由 Owner/OCR 配置流程触发)
offRamp.setOCR2Config (signers, transmitters, f, abi.encode (
  EVM2EVMOffRamp.DynamicConfig ({
    permissionLessExecutionThresholdSeconds: 600,
    maxDataBytes: 200_000,
    maxNumberOfTokensPerMsg: 5,
    router: router,
    priceRegistry: priceRegistry
  })
), version, offchainCfg);

// 2) 手动执行 (失败或超时后 提升 gasLimit 重试)
EVM2EVMOffRamp.GasLimitOverride[] memory ovs = new EVM2EVMOffRamp.GasLimitOverride[] (report.messages.length);
ovs[0] = EVM2EVMOffRamp.GasLimitOverride ({ receiverExecutionGasLimit: 500_000, tokenGasOverrides: new uint32[] (0) });
offRamp.manuallyExecute (report, ovs);

// 3) 维护聚合限速 Token 集合
EVM2EVMOffRamp.RateLimitToken[] memory adds = new EVM2EVMOffRamp.RateLimitToken[] (1);
adds[0] = EVM2EVMOffRamp.RateLimitToken ({ sourceToken: srcToken, destToken: dstToken });
offRamp.updateRateLimitTokens (new EVM2EVMOffRamp.RateLimitToken[] (0), adds);
```

---

### 十一、目录结构 & 依赖
```text
EVM2EVMOffRamp/
  └─ src/v0.8/ccip/
       ├─ offRamp/EVM2EVMOffRamp.sol
       ├─ AggregateRateLimiter.sol
       ├─ libraries/{Client.sol, Internal.sol, Pool.sol, RateLimiter.sol, USDPriceWith18Decimals.sol}
       ├─ interfaces/{IAny2EVMMessageReceiver.sol, IAny2EVMOffRamp.sol, ICommitStore.sol, IPool.sol, IPriceRegistry.sol, IRMN.sol, IRouter.sol, ITokenAdminRegistry.sol}
       ├─ ocr/{OCR2Abstract.sol, OCR2BaseNoChecks.sol}
       └─ shared/{access/*, call/CallWithExactGas.sol, enumerable/*, interfaces/*}
```
