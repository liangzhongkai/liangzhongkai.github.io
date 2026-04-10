+++
date = '2026-04-10T12:00:00+08:00'
draft = false
title = 'Uniswap V4 Dynamic Fees'
tags = ['web3']
+++

# Uniswap V4 Dynamic Fees

> 本文基于 [v4-core](https://github.com/Uniswap/v4-core) 源码，逐层拆解动态费率的实现机制。

---

## 一、静态 vs 动态：V3 与 V4 的根本差异

在 V3 中，手续费在池子创建时固定写死（0.05%、0.3%、1% 等），不可更改。V4 引入了两层改变：

- **静态费率**仍然支持，在 `initialize()` 时通过 `fee` 参数指定，写入 `PoolKey`。
- **动态费率**：在 `PoolKey.fee` 中传入特殊值 `0x800000`（即 `LPFeeLibrary.DYNAMIC_FEE_FLAG`），告诉 PoolManager：这个池子的费率由 Hook 实时决定。

---

## 二、关键数据结构

```solidity
struct PoolKey {
    Currency currency0;
    Currency currency1;
    uint24 fee;          // 最高位置1 = 动态费率模式 (0x800000)
    int24 tickSpacing;
    IHooks hooks;
}
```

`LPFeeLibrary` 中定义了两个关键 flag：

```solidity
// LPFeeLibrary.sol
uint24 public constant DYNAMIC_FEE_FLAG  = 0x800000;  // bit 23：标记该池为动态费率池
uint24 public constant OVERRIDE_FEE_FLAG = 0x400000;  // bit 22：beforeSwap 返回时用于覆盖本次费率
uint24 public constant REMOVE_OVERRIDE_MASK = 0xBFFFFF; // 剥去 OVERRIDE_FEE_FLAG 的掩码
uint24 public constant MAX_LP_FEE = 1_000_000;         // 最大费率 100%
```

`uint24` 的位布局：

```
bit 23  (0x800000): DYNAMIC_FEE_FLAG  — PoolKey.fee 设此位 = 动态费率池
bit 22  (0x400000): OVERRIDE_FEE_FLAG — beforeSwap 返回值设此位 = 覆盖本次费率
bit 0-21(0x3FFFFF): 实际费率值 (max 1_000_000 = 100%)
```

---

## 三、两种更新机制

动态费率有两条独立的生效路径，定位完全不同。

### 路径 A：`updateDynamicLPFee()` — 持久化更新

Hook 在任意时刻（`afterInitialize`、`afterSwap`、外部 keeper 触发等）主动调用 PoolManager 的该函数，将新费率**永久写入** `slot0.lpFee`，对所有后续 swap 生效，直到下次被再次调用覆盖。

```solidity
// Hook 合约中调用
poolManager.updateDynamicLPFee(poolKey, newFee);

// PoolManager 内部
function updateDynamicLPFee(PoolKey memory key, uint24 newDynamicLPFee) external {
    // 1. 只有动态费率池才允许调用
    // 2. 只有该池的 hook 合约本身（msg.sender == address(key.hooks)）才有权限
    if (!key.fee.isDynamicFee() || msg.sender != address(key.hooks)) {
        revert NotDynamicFee();
    }
    key.toId().updateLPFee(newDynamicLPFee);  // 写入 slot0.lpFee
}
```

> **要点**：`updateDynamicLPFee` 可以被反复调用，每次调用都覆盖 `slot0.lpFee`。它是"批量生效直到下次更新"的机制，Hook 可以根据波动率、时间段等条件随时调整。

---

### 路径 B：`beforeSwap` 返回值 override — 单次覆盖

这是真正"每笔 swap 独立定价"的机制。`beforeSwap` 的返回值中第三个参数 `uint24 lpFeeOverride` 通过 `OVERRIDE_FEE_FLAG`（bit 22）标识是否启用本次覆盖，**不写入 storage，不影响其他 swap**。

#### 完整调用栈

```
用户调用
  └─ PoolManager.swap(key, params, hookData)
       │
       ├─ Hooks.beforeSwap(key, params, hookData)         [Hooks.sol L248]
       │    ├─ callHook → IHooks.beforeSwap(...)          Hook 合约执行，返回 96 bytes
       │    └─ if isDynamicFee → lpFeeOverride = result.parseFee()   [L263]
       │                         ↑ 静态费率池：此值直接丢弃
       │
       └─ PoolManager._swap(pool, id, SwapParams{lpFeeOverride}, ...)
            └─ Pool.swap(params)                          [Pool.sol L279]
                 ├─ params.lpFeeOverride.isOverride() ?   [L303]
                 │    yes → removeOverrideFlagAndValidate()   本次用 hook 的 fee
                 │    no  → slot0Start.lpFee()               本次用 storage 里的 fee
                 └─ swapFee 参与全程 tick 循环计价
```

---

## 四、源码逐层解析

### Layer 1：`PoolManager.swap()` — 调起 hook，传递 override

```solidity
// PoolManager.sol L187-217
function swap(PoolKey memory key, SwapParams memory params, bytes calldata hookData)
    external onlyWhenUnlocked noDelegateCall
    returns (BalanceDelta swapDelta)
{
    PoolId id = key.toId();
    Pool.State storage pool = _getPool(id);
    pool.checkPoolInitialized();

    BeforeSwapDelta beforeSwapDelta;
    {
        int256 amountToSwap;
        uint24 lpFeeOverride;
        // ① 调用 Hooks 库的 beforeSwap，拿到三元组返回值
        (amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);

        // ② 将 lpFeeOverride 透传给 Pool 层
        swapDelta = _swap(pool, id, Pool.SwapParams({
            tickSpacing:       key.tickSpacing,
            zeroForOne:        params.zeroForOne,
            amountSpecified:   amountToSwap,
            sqrtPriceLimitX96: params.sqrtPriceLimitX96,
            lpFeeOverride:     lpFeeOverride   // 核心透传
        }), params.zeroForOne ? key.currency0 : key.currency1);
    }

    // ③ afterSwap hook（可处理 hookDelta）
    BalanceDelta hookDelta;
    (swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);
    if (hookDelta != BalanceDeltaLibrary.ZERO_DELTA)
        _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));

    _accountPoolBalanceDelta(key, swapDelta, msg.sender);
}
```

---

### Layer 2：`Hooks.beforeSwap()` — 解析返回值，提取 fee

```solidity
// Hooks.sol L248-282
function beforeSwap(IHooks self, PoolKey memory key, SwapParams memory params, bytes calldata hookData)
    internal
    returns (int256 amountToSwap, BeforeSwapDelta hookReturn, uint24 lpFeeOverride)
{
    amountToSwap = params.amountSpecified;
    if (msg.sender == address(self)) return (amountToSwap, BeforeSwapDeltaLibrary.ZERO_DELTA, lpFeeOverride);

    if (self.hasPermission(BEFORE_SWAP_FLAG)) {
        bytes memory result = callHook(self,
            abi.encodeCall(IHooks.beforeSwap, (msg.sender, key, params, hookData)));

        // 返回值必须是 96 bytes：bytes4 selector + BeforeSwapDelta(32) + uint24 fee(32)
        if (result.length != 96) InvalidHookResponse.selector.revertWith();

        // ★ 关键 guard：只有 isDynamicFee 的池子才解析 fee override
        // 静态费率池的 beforeSwap 就算返回了 fee，也在这里被丢弃
        if (key.fee.isDynamicFee()) lpFeeOverride = result.parseFee();

        if (self.hasPermission(BEFORE_SWAP_RETURNS_DELTA_FLAG)) {
            hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());
            int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();
            if (hookDeltaSpecified != 0) {
                bool exactInput = amountToSwap < 0;
                amountToSwap += hookDeltaSpecified;
                if (exactInput ? amountToSwap > 0 : amountToSwap < 0)
                    HookDeltaExceedsSwapAmount.selector.revertWith();
            }
        }
    }
}
```

---

### Layer 3：`Pool.swap()` — 决策用哪个 fee

```solidity
// Pool.sol L279-307
function swap(State storage self, SwapParams memory params)
    internal
    returns (BalanceDelta swapDelta, uint256 amountToProtocol, uint24 swapFee, SwapResult memory result)
{
    Slot0 slot0Start = self.slot0;
    bool zeroForOne = params.zeroForOne;

    uint256 protocolFee = zeroForOne
        ? slot0Start.protocolFee().getZeroForOneFee()
        : slot0Start.protocolFee().getOneForZeroFee();

    // ...（初始化 amountSpecifiedRemaining、sqrtPriceX96、tick、liquidity）

    // ★ 费率决策的核心分支
    {
        uint24 lpFee = params.lpFeeOverride.isOverride()
            ? params.lpFeeOverride.removeOverrideFlagAndValidate()  // 剥去 0x400000，取实际费率
            : slot0Start.lpFee();                                   // 读 slot0 storage

        swapFee = protocolFee == 0
            ? lpFee
            : uint16(protocolFee).calculateSwapFee(lpFee);  // 叠加协议费
    }

    // swapFee 参与后续整个 tick 循环计价...
}
```

---

### Layer 4：`LPFeeLibrary` — 位操作语义

```solidity
// LPFeeLibrary.sol — 完整核心实现

function isDynamicFee(uint24 self) internal pure returns (bool) {
    return self == DYNAMIC_FEE_FLAG;   // PoolKey.fee 是否为动态费率池标识
}

function isOverride(uint24 self) internal pure returns (bool) {
    return self & OVERRIDE_FEE_FLAG != 0;  // beforeSwap 返回值是否携带 override 标志
}

function removeOverrideFlag(uint24 self) internal pure returns (uint24) {
    return self & REMOVE_OVERRIDE_MASK;    // 0xBFFFFF：清除 bit 22，保留实际费率
}

function removeOverrideFlagAndValidate(uint24 self) internal pure returns (uint24 fee) {
    fee = self.removeOverrideFlag();
    fee.validate();    // 校验 fee <= MAX_LP_FEE (1_000_000)，否则 revert
}

function getInitialLPFee(uint24 self) internal pure returns (uint24) {
    // 动态费率池初始 fee 为 0
    // 若想要非 0 初始费率，在 afterInitialize hook 中调用 updateDynamicLPFee
    if (self.isDynamicFee()) return 0;
    self.validate();
    return self;
}
```

---

## 五、两条路径的定位对比

| 维度 | 路径 A：`updateDynamicLPFee` | 路径 B：`beforeSwap` override |
|---|---|---|
| 写入 storage | ✅ 写 `slot0.lpFee` | ❌ 不写 storage |
| 生效范围 | 所有后续 swap，直到再次更新 | 仅本次 swap |
| 调用权限 | Hook 合约本身，任意时机 | 每次 swap 自动触发 |
| 适合场景 | 定时更新、keeper 驱动、区间费率 | per-swap 精确定价、用户级差异化 |
| gas 成本 | 一次 SSTORE | 无额外 storage 写入 |

两者通常组合使用：路径 A 维护一个"当前基准费率"，路径 B 在特定条件下（如白名单用户、高波动瞬间）临时覆盖单笔。

---

## 六、典型应用场景

| 策略 | 实现方式 | 说明 |
|---|---|---|
| 波动率感知费率 | `beforeSwap` 查链上预言机，高波动时提高 fee | 每笔独立定价 |
| 时间段折扣 | keeper 定时调用 `updateDynamicLPFee` | 交易时段优惠 |
| 用户白名单折扣 | `beforeSwap` 检查 `sender` 地址 | 协议集成商优惠 |
| 均值回归模型 | 根据当前 tick 偏离 TWAP 动态调整 | 做市商保护 |
| MEV 内化 | Hook 接受外部竞价，高出价者获低手续费 | 搜索者博弈 |

---

## 七、关键约束总结

1. **只有 `DYNAMIC_FEE_FLAG` 的池**才能调用 `updateDynamicLPFee()`，静态费率池调用会 revert。
2. **只有该池的 Hook 合约本身**（`msg.sender == address(key.hooks)`）才有权限调用 `updateDynamicLPFee`。
3. `beforeSwap` 的 fee override **只对动态费率池生效**（`Hooks.sol L263` 的 `isDynamicFee()` guard）。
4. Fee 值上限 `MAX_LP_FEE = 1_000_000`（100%），由 `validate()` 强制校验。
5. 动态费率池的**初始 fee 为 0**，若需非 0 初始值，须在 `afterInitialize` hook 中调用 `updateDynamicLPFee`。

---

## 参考

- [v4-core `PoolManager.sol`](https://github.com/Uniswap/v4-core/blob/main/src/PoolManager.sol)
- [v4-core `Pool.sol`](https://github.com/Uniswap/v4-core/blob/main/src/libraries/Pool.sol)
- [v4-core `Hooks.sol`](https://github.com/Uniswap/v4-core/blob/main/src/libraries/Hooks.sol)
- [v4-core `LPFeeLibrary.sol`](https://github.com/Uniswap/v4-core/blob/main/src/libraries/LPFeeLibrary.sol)
