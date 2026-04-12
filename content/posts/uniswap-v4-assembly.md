+++
date = '2026-04-12T10:00:00+08:00'
draft = false
title = 'Uniswap V4 Assembly'
tags = ['web3']
summary = '结合 Uniswap v4-core 源码，系统拆解 Uniswap V4 在状态压缩、瞬态存储、Hook 调用与错误处理中的 Assembly 模式。'
+++

# Uniswap V4 Assembly

> **目标读者**：熟悉 Solidity 基础、了解 EVM 存储模型、希望深入掌握生产级 Assembly 写法的 DeFi 工程师与安全审计员。

---

## 1. 为什么 V4 大量使用 Assembly？

Uniswap V4 的核心合约 `PoolManager.sol` 在部署时接近以太坊的 **24,576 字节合约大小上限**（EIP-170）。同时，作为链上交易量最大的 AMM 协议之一，每一笔 swap 节省的几百 gas 乘以数亿次交易，就是数十亿美元的用户成本。

V4 使用 Assembly 的四个根本动机：

| 动机 | 具体体现 | Gas 收益量级 |
|------|----------|-------------|
| **减少 bytecode 体积** | `extsload` 代替数十个 view getter | 每个 getter ~200 bytes → 节省数 KB |
| **绕过 Solidity ABI 编码开销** | `callHook` 手动 `call` + `returndatacopy` | 节省 ~500 gas/次 |
| **EIP-1153 瞬态存储** | `tstore`/`tload` 承载交易内共享状态 | 相比持久化存储通常是数量级优化 |
| **Scratch space revert** | `CustomRevert` 在 0x00 处构造 revert data | 省去 memory 扩展 gas |

> **核心哲学**：V4 的每一段 assembly 都有明确的经济理由。理解这个理由，是读懂代码的前提。

> **版本说明**：本文基于 `Uniswap/v4-core` `main` 分支在 2026-04-12 的公开源码整理；若后续版本调整了底层实现，请以对应 commit 为准。

---

## 2. 前置知识：EVM 内存模型速查

读 V4 的 assembly 前，需要把以下概念内化为直觉：

### 2.1 内存区域划分

```
地址范围         名称             规则
─────────────────────────────────────────────────────
0x00 ~ 0x1f    Scratch space    汇编可随意使用，不受 fmp 保护
0x20 ~ 0x3f    Scratch space    同上（共 64 字节 scratch）
0x40 ~ 0x5f    Free memory ptr  mload(0x40) 读取当前 fmp
0x60 ~ 0x7f    Zero slot        永远为 0，不可写
0x80 ~         动态分配区        所有 new/abi.encode 从此开始
```

**关键约束**：`memory-safe` 标注的 assembly 块必须：
- 不在 `[fmp, fmp+size)` 外写数据（0x00-0x3f 除外）
- 若扩展 memory，必须更新 `mload(0x40)`

### 2.2 Solidity 变量的内存布局

```solidity
bytes memory data; // data 本身是一个指针（uint256）
// 内存布局：
// [data]        → 指向某地址 X
// [X]           → data.length（32 bytes）
// [X + 0x20]    → data[0]（第一个字节在高位，左对齐）
// [X + 0x20 + n]→ data[n]

uint256[] memory arr; // 类似，arr 是指针
// [arr]         → 指向地址 Y
// [Y]           → arr.length
// [Y + 0x20]    → arr[0]
// [Y + 0x20*i]  → arr[i]
```

### 2.3 关键 Opcode 速查表

| Opcode | Gas | 语义 | V4 使用场景 |
|--------|-----|------|------------|
| `TLOAD` | ~100 | 读瞬态存储 | CurrencyDelta, Lock, NonzeroDeltaCount |
| `TSTORE` | ~100 | 写瞬态存储 | 同上 |
| `SLOAD` | 冷2100/热100 | 读永久存储 | extsload 核心 |
| `SSTORE` | 冷22100/热100/2900 | 写永久存储 | 池子初始化、非瞬态余额持久化、协议费率更新 |
| `MLOAD` | 3 | 读内存数据 | 构建 Hook 调用参数、从内存中提取 swap 返回的 BalanceDelta |
| `MSTORE` | 3 | 写内存数据 | 在内存中编码 Hook 的 calldata、准备 keccak256 的输入 |
| `SAR` | 3 | 算术右移（保符号） | 提取 int128, int24 |
| `SHR` | 3 | 逻辑右移（补零） | 提取 uint 字段 |
| `SIGNEXTEND` | 5 | N 字节整数符号扩展 | tick(int24), amount(int128) |
| `RETURNDATASIZE` | 2 | 上一次 call 返回长度 | ERC20 兼容性检测 |
| `RETURNDATACOPY` | 3+mem | 复制返回数据 | callHook 结果捕获 |
| `KECCAK256` | 30 + 6/word | 计算哈希 | 将 PoolKey 计算为唯一的 PoolId |

### 2.4 有符号整数的两种提取方式

这是 V4 assembly 中最容易混淆的点，必须先搞清楚：

```
一个 256-bit word 里有两个 128-bit 字段：
[  HIGH 128 bits (amount0/int128)  ][  LOW 128 bits (amount1/int128)  ]

提取 HIGH (int128)：使用 SAR（算术右移）
  sar(128, word) → 高 128 bit 移到低位，高位用符号填充 → 正确的 int128

提取 LOW (int128)：使用 SIGNEXTEND
  signextend(15, word) → 把第 15 字节（bit 127）作为符号位向高扩展 → 正确的 int128

❌ 错误做法：用 shr(128, word) 提取高字段
   → shr 补零，负数变正数，语义错误

❌ 错误做法：用 and(word, 0xFF...FF) 提取低字段（128个1）
   → and 不做符号扩展，-1 变成 type(uint128).max，语义错误
```

---

## 3. 模式一：Slot0 多字段位解包

**文件**：`src/libraries/StateLibrary.sol`

### 3.1 背景

V4 把 pool 的核心状态压缩进单个 storage slot（`Slot0`），单次 SLOAD 读取 4 个字段：

```
bit 255        bit 231   bit 207   bit 183    bit 159          bit 0
┌──────────────┬─────────┬──────────────────┬──────────┬───────────────────────┐
│  unused(24)  │ lpFee   │ protocolFee      │  tick    │   sqrtPriceX96        │
│              │  (24)   │ (12 + 12 bits)   │  (24)    │     (160 bits)        │
└──────────────┴─────────┴──────────────────┴──────────┴───────────────────────┘
```

其中 `protocolFee` 虽然在 `StateLibrary.getSlot0()` 里整体按 `uint24` 读出，但在 `src/types/Slot0.sol` 的语义里它实际是两个方向各 12 bit 的打包值：高 12 bit 表示 `1->0`，低 12 bit 表示 `0->1`。

### 3.2 完整代码解析

```solidity
function getSlot0(IPoolManager manager, PoolId poolId)
    internal
    view
    returns (uint160 sqrtPriceX96, int24 tick, uint24 protocolFee, uint24 lpFee)
{
    bytes32 stateSlot = _getPoolStateSlot(poolId);
    bytes32 data = manager.extsload(stateSlot);  // 单次 SLOAD

    assembly ("memory-safe") {
        // ① sqrtPriceX96：低 160 bit，无符号
        sqrtPriceX96 := and(data, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF)
        //              └── mask = 160个1，清除高 96 bit

        // ② tick：bit 160-183，有符号 int24
        tick := signextend(2, shr(160, data))
        //      └─ shr(160): 把 tick 移到低 24 bit
        //         signextend(2, x): 把 x 的第 2 字节（bit 23）作为符号位
        //                           向高 232 bit 扩展 → 正确 int24

        // ③ protocolFee：bit 184-207，无符号 uint24
        protocolFee := and(shr(184, data), 0xFFFFFF)
        //             └── shr(184) + mask 24 bit

        // ④ lpFee：bit 208-231，无符号 uint24
        lpFee := and(shr(208, data), 0xFFFFFF)
    }
}
```

### 3.3 关键推导：signextend(2, x) 的数学

`signextend(b, x)` 的语义：把 `x` 的第 `b` 字节（0-indexed，从低位数）视为符号位，向高位做符号扩展。

```
b=2 → 对应第 3 个字节（byte 2）→ 即 bit 23（24-bit 整数的符号位）

示例：tick = -1（int24 补码 = 0xFFFFFF）
  shr(160, data) 后低 24 bit = 0xFFFFFF，但高 232 bit = 0
  signextend(2, 0xFFFFFF)
  → bit 23 = 1（符号位），向高扩展
  → 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
  → 转为 int256 = -1 ✓

反例（如果用 shr 而非 signextend）：
  and(shr(160, data), 0xFFFFFF) → 0xFFFFFF = 16777215（正数！）✗
```

### 3.4 为何只用 1 次 SLOAD？

传统写法往往需要 4 个独立 getter，各自携带函数入口、ABI 编码与独立读槽逻辑。V4 把这些字段压进 1 个 slot，并通过一次 `extsload` 在 `PoolManager` 内部完成单次 `SLOAD`，从而把原本分散的多次读槽合并成一次。精确节省多少 gas 仍取决于调用路径、冷热槽与是否链上读取，但方向上就是用更少的 bytecode 和更少的读槽次数换取成本下降。

---

## 4. 模式二：BalanceDelta / BeforeSwapDelta 双 int128 打包

**文件**：`src/types/BalanceDelta.sol`，`src/types/BeforeSwapDelta.sol`

### 4.1 布局设计

```
BalanceDelta (int256):
┌────────────────────────────┬────────────────────────────┐
│    amount0 (int128)        │    amount1 (int128)        │
│    高 128 bit              │    低 128 bit              │
└────────────────────────────┴────────────────────────────┘

BeforeSwapDelta (int256)：布局相同，语义不同
┌────────────────────────────┬────────────────────────────┐
│  deltaSpecified (int128)   │ deltaUnspecified (int128)  │
│    高 128 bit              │    低 128 bit              │
└────────────────────────────┴────────────────────────────┘
```

### 4.2 构造：打包两个 int128

```solidity
function toBalanceDelta(int128 _amount0, int128 _amount1)
    pure
    returns (BalanceDelta balanceDelta)
{
    assembly ("memory-safe") {
        balanceDelta := or(
            shl(128, _amount0),                     // ① amount0 移到高 128 bit
            and(sub(shl(128, 1), 1), _amount1)       // ② amount1 保留低 128 bit
        )
    }
}
```

**拆解 `and(sub(shl(128, 1), 1), _amount1)`**：

```
shl(128, 1)       = 2^128 = 0x0000...0001 0000...0000
                              ^^^high^^^ ^^^low128^^^
sub(shl(128,1), 1)= 2^128-1 = 0x0000...0000 FFFF...FFFF
                              ^^^  0  ^^^  ^^^128个1^^^

这就是 128-bit mask。

为什么需要这个 mask？
  _amount1 是 int128，在 EVM stack 上是 int256（符号扩展）
  如果 _amount1 = -1：
    stack 值 = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    直接 shl(128, _amount0) | _amount1 → 高 128 bit 被污染！

  and(mask, _amount1) = 0x000...000 FFFF...FFFF（低 128 bit）
  安全清除高 128 bit ✓
```

### 4.3 提取：高低字段的不同操作

```solidity
// 提取高 128 bit（amount0 / deltaSpecified）
function amount0(BalanceDelta balanceDelta) internal pure returns (int128 _amount0) {
    assembly ("memory-safe") {
        _amount0 := sar(128, balanceDelta)
        // SAR = 算术右移，高位补符号位
        // balanceDelta >> 128，高 128 bit 移到低位，高位填充符号
        // 负数示例：0xFFFF...FFFF 0000...0000
        //   sar(128) → 0xFFFF...FFFF FFFF...FFFF = -1 ✓
    }
}

// 提取低 128 bit（amount1 / deltaUnspecified）
function amount1(BalanceDelta balanceDelta) internal pure returns (int128 _amount1) {
    assembly ("memory-safe") {
        _amount1 := signextend(15, balanceDelta)
        // signextend(15, x)：把第 15 字节（byte index 15 = bit 127）作为符号位
        // 即把低 128 bit 当作 int128，符号扩展到 256 bit
        // 负数示例：低 128 bit = 0xFFFF...FFFF
        //   bit 127 = 1（符号位）→ 高 128 bit 填充 1
        //   结果 = 0xFFFF...FFFF FFFF...FFFF = -1 ✓
    }
}
```

### 4.4 加法：字段独立运算

```solidity
function add(BalanceDelta a, BalanceDelta b) pure returns (BalanceDelta) {
    int256 res0;
    int256 res1;
    assembly ("memory-safe") {
        let a0 := sar(128, a)        // 提取 a.amount0（有符号）
        let a1 := signextend(15, a)  // 提取 a.amount1（有符号）
        let b0 := sar(128, b)
        let b1 := signextend(15, b)
        res0 := add(a0, b0)          // 128 bit 加法（在 256 bit 空间内）
        res1 := add(a1, b1)
    }
    // 用 toInt128 截断回 128 bit（会 revert on overflow）
    return toBalanceDelta(res0.toInt128(), res1.toInt128());
}
```

> **设计亮点**：assembly 内部在 256-bit 空间做加法（不会溢出 256 bit），最后用 Solidity 的 `toInt128()` 做溢出检查截断。分离了"无溢出计算"和"范围验证"两个关注点。

---

## 5. 模式三：EIP-1153 瞬态存储（TSTORE / TLOAD）

**文件**：`src/libraries/CurrencyReserves.sol`，`src/libraries/NonzeroDeltaCount.sol`，`src/libraries/Lock.sol`

### 5.1 什么是瞬态存储

EIP-1153 在 Cancun 硬分叉（2024年3月）引入两个新 opcode：

```
TSTORE(key, value)  → 写入当前交易的瞬态存储，交易结束自动清零
TLOAD(key)          → 读取当前交易的瞬态存储

关键特性：
- Gas 约 100（vs SSTORE 冷写 22,100）
- 交易结束自动清零，无需 cleanup
- 仅对当前交易可见（同 storage，不同于 memory）
- 不产生 storage refund，不进入 state trie
```

### 5.2 Slot 命名：keccak - 1 模式

V4 的所有瞬态 slot 常量都遵循 EIP-1967 的"防碰撞"命名：

```solidity
// CurrencyReserves.sol
bytes32 constant RESERVES_OF_SLOT =
    bytes32(uint256(keccak256("ReservesOf")) - 1);
//  = 0x1e0745a7db1623981f0b2a5d4232364c00787266eb75ad546f190e6cebe9bd95

bytes32 constant CURRENCY_SLOT =
    bytes32(uint256(keccak256("Currency")) - 1);
//  = 0x27e098c505d44ec3574004bca052aabf76bd35004c182099d8c575fb238593b9
```

**为什么减 1？**

`keccak256(x)` 的结果本身可能是某个 Solidity mapping 的 slot key（`keccak256(key || mappingSlot)`）。减 1 后，该值不可能是任何 Solidity mapping 自然产生的 slot，消除碰撞风险。

### 5.3 CurrencyReserves 的三个操作

```solidity
library CurrencyReserves {
    bytes32 constant RESERVES_OF_SLOT = 0x1e07...bd95;
    bytes32 constant CURRENCY_SLOT    = 0x27e0...3b9;

    // 读取当前 synced currency
    function getSyncedCurrency() internal view returns (Currency currency) {
        assembly ("memory-safe") {
            currency := tload(CURRENCY_SLOT)
        }
    }

    // 原子写入 currency 和 reserves（两次 tstore）
    function syncCurrencyAndReserves(Currency currency, uint256 value) internal {
        assembly ("memory-safe") {
            // mask address 的高 96 bit（address 只保证低 160 bit 有效）
            tstore(CURRENCY_SLOT, and(currency, 0xffffffffffffffffffffffffffffffffffffffff))
            tstore(RESERVES_OF_SLOT, value)
        }
    }

    // 重置（settle 后调用）
    function resetCurrency() internal {
        assembly ("memory-safe") {
            tstore(CURRENCY_SLOT, 0)
        }
    }
}
```

### 5.4 V4 unlock() 中的瞬态状态机

```
PoolManager.unlock(data) 调用流程：

1. Lock.unlock()
   └── tstore(IS_UNLOCKED_SLOT, true)          ← 开锁

2. result = IUnlockCallback(msg.sender).unlockCallback(data)
   │
   ├── 在 callback 内，用户可调用：
   │   ├── swap()     → _accountDelta() → CurrencyDelta.applyDelta()
   │   │                               → NonzeroDeltaCount.increment/decrement()
   │   ├── settle()   → CurrencyReserves.getSyncedCurrency() / resetCurrency()
   │   └── take()     → _accountDelta()
   │
   └── 所有中间状态存于瞬态存储，零 SSTORE

3. if (NonzeroDeltaCount.read() != 0) revert CurrencyNotSettled()
   └── 确保所有债务归零（flash accounting 安全守护）

4. Lock.lock()
   └── tstore(IS_UNLOCKED_SLOT, false)          ← 关锁

5. 交易结束 → 所有瞬态 slot 自动清零（EVM 保证）
```

---

## 6. 模式四：CurrencyDelta — 瞬态 keccak Slot 映射

**文件**：`src/libraries/CurrencyDelta.sol`

### 6.1 设计动机

Flash accounting 需要为每个 `(address, currency)` 对维护一个独立的 delta 余额。如果把它做成普通持久化 `mapping`，就会落到常规 `SLOAD` / `SSTORE` 的成本模型上。V4 的做法是把这张“临时账本”搬到瞬态存储，slot 则通过 keccak 动态计算。

### 6.2 完整代码

```solidity
library CurrencyDelta {

    // slot = keccak256(abi.encodePacked(target, currency))
    // 完全在 scratch space 内计算，不分配 memory
    function _computeSlot(address target, Currency currency)
        internal
        pure
        returns (bytes32 hashSlot)
    {
        assembly ("memory-safe") {
            // 写入 scratch space（0x00-0x3f）
            mstore(0,  and(target,   0xffffffffffffffffffffffffffffffffffffffff))
            mstore(32, and(currency, 0xffffffffffffffffffffffffffffffffffffffff))
            // keccak256(scratch[0..63])
            hashSlot := keccak256(0, 64)
        }
        // 注意：scratch space 写完即用，不需要清理
        // memory-safe 允许在 0x00-0x3f 写数据（即使不清理）
    }

    function getDelta(Currency currency, address target)
        internal
        view
        returns (int256 delta)
    {
        bytes32 hashSlot = _computeSlot(target, currency);
        assembly ("memory-safe") {
            delta := tload(hashSlot)
        }
    }

    function applyDelta(Currency currency, address target, int128 delta)
        internal
        returns (int256 previous, int256 next)
    {
        bytes32 hashSlot = _computeSlot(target, currency);
        assembly ("memory-safe") {
            previous := tload(hashSlot)    // 读旧值
        }
        next = previous + delta;           // Solidity 加法（带 overflow 检查）
        assembly ("memory-safe") {
            tstore(hashSlot, next)         // 写新值
        }
    }
}
```

### 6.3 Scratch Space 用于 keccak 的安全性

```
Scratch space 布局（assembly 写入时）：
0x00: target   (32 bytes, address mask 后右对齐)
0x20: currency (32 bytes, address mask 后右对齐)

keccak256(0, 64) 等价于 keccak256(abi.encodePacked(target_padded, currency_padded))

为什么可以这样做？
- Solidity 规范：0x00-0x3f 是 scratch space，assembly 块可自由使用
- "memory-safe" 标注：编译器知道这段 asm 遵守规范，可放心优化
- keccak 是纯计算，写完即计算，结果存在 hashSlot，无需保留 scratch 数据
```

### 6.4 与 Solidity Mapping Slot 的区别

| 特性 | Solidity Mapping | CurrencyDelta |
|------|-----------------|----------------|
| Slot 计算 | `keccak256(key ++ slot_number)` | `keccak256(target ++ currency)` |
| 存储位置 | 永久存储 | 瞬态存储 |
| Gas（写） | 冷写 22,100 | ~100 |
| 生命周期 | 永久 | 仅当前交易 |
| 碰撞风险 | mapping slot 编号隔离 | 独立的 tstore 命名空间 |

---

## 7. 模式五：NonzeroDeltaCount — Flash Accounting 安全计数器

**文件**：`src/libraries/NonzeroDeltaCount.sol`

### 7.1 作用

这是 V4 flash accounting 的安全门：跟踪当前有多少个 `(address, currency)` 对的 delta 非零。`unlock()` 结束时必须为 0，否则有人欠债未还，交易 revert。

### 7.2 完整实现

```solidity
library NonzeroDeltaCount {
    // slot = keccak256("NonzeroDeltaCount") - 1
    bytes32 internal constant NONZERO_DELTA_COUNT_SLOT =
        0x7d4b3164c6e45b97e7d87b7125a44c5828d005af88f9d751cfd78729c5d99a0b;

    function read() internal view returns (uint256 count) {
        assembly ("memory-safe") {
            count := tload(NONZERO_DELTA_COUNT_SLOT)
        }
    }

    function increment() internal {
        assembly ("memory-safe") {
            let count := tload(NONZERO_DELTA_COUNT_SLOT)
            count := add(count, 1)
            tstore(NONZERO_DELTA_COUNT_SLOT, count)
        }
    }

    function decrement() internal {
        assembly ("memory-safe") {
            let count := tload(NONZERO_DELTA_COUNT_SLOT)
            count := sub(count, 1)
            // ⚠️ 无 underflow 保护！依赖调用方保证
            tstore(NONZERO_DELTA_COUNT_SLOT, count)
        }
    }
}
```

### 7.3 调用逻辑与 `_accountDelta` 的协议

```solidity
// PoolManager._accountDelta（唯一的调用入口）
function _accountDelta(Currency currency, int128 delta, address target) internal {
    if (delta == 0) return;  // 零 delta 不计入

    (int256 previous, int256 next) = currency.applyDelta(target, delta);

    if (next == 0) {
        NonzeroDeltaCount.decrement();    // 非零 → 零：债务消除
    } else if (previous == 0) {
        NonzeroDeltaCount.increment();    // 零 → 非零：新债产生
    }
    // previous != 0 且 next != 0：计数不变（债务变化但未消除）
}
```

**为什么 decrement 没有保护？**

只有 `_accountDelta` 调用 `decrement`，而 `decrement` 只在 `next == 0`（即 delta 从非零变零）时触发，此时 count 必然已经被 `increment` 过，因此不可能 underflow。这是**调用层保证的不变量**，而非 assembly 内部保证。

---

## 8. 模式六：extsload / exttload — 外部存储暴露

**文件**：`src/Extsload.sol`，`src/Exttload.sol`

### 8.1 设计背景

V4 的 `PoolManager` 接近 EIP-170 的合约大小上限。传统做法是为每个状态字段写一个专用 getter，但每个 getter 都会继续膨胀 bytecode。V4 的解法是暴露**通用 slot 读取接口**，把“具体读哪个 slot”的知识收敛到调用方或辅助库（例如 `StateLibrary`）里，而不是把大量只读包装函数塞进核心合约。

### 8.2 三个 extsload 重载

```solidity
// 重载 1：单 slot
function extsload(bytes32 slot) external view returns (bytes32) {
    assembly ("memory-safe") {
        mstore(0, sload(slot))   // 读 slot，写到 scratch space
        return(0, 0x20)          // 直接返回 32 bytes，绕过 Solidity ABI 编码
    }
}

// 重载 2：连续 N 个 slot
function extsload(bytes32 startSlot, uint256 nSlots)
    external
    view
    returns (bytes32[] memory)
{
    assembly ("memory-safe") {
        // 构造 ABI 编码的 bytes32[]
        mstore(0x00, 0x20)       // offset to array data
        mstore(0x20, nSlots)     // array length

        let memPtr := 0x40
        let endSlot := add(startSlot, nSlots)

        for { let i := startSlot } lt(i, endSlot) { i := add(i, 1) } {
            mstore(memPtr, sload(i))          // 逐 slot SLOAD 写入
            memPtr := add(memPtr, 0x20)
        }

        return(0x00, memPtr)     // 从 slot 0 开始直接返回所有数据
        // 注意：没有更新 free memory pointer（0x40）
        // 因为 return 后执行立即终止，不存在后续 memory 操作
    }
}

// 重载 3：离散 slot 列表（sparse extsload）
function extsload(bytes32[] calldata slots) external view returns (bytes32[] memory) {
    assembly ("memory-safe") {
        let memPtr := mload(0x40)            // 获取 fmp
        let results := memPtr
        // bytes32[] 的返回值仍然是一个动态类型，最外层先放 offset
        mstore(memPtr, 0x20)
        // 再写数组长度
        mstore(add(memPtr, 0x20), slots.length)
        // 元素区从 results + 0x40 开始
        memPtr := add(memPtr, 0x40)

        let slotsData := slots.offset        // calldata offset
        for { let i := 0 } lt(i, slots.length) { i := add(i, 1) } {
            let slot := calldataload(add(slotsData, mul(i, 0x20)))
            mstore(memPtr, sload(slot))
            memPtr := add(memPtr, 0x20)
        }
        // 更新 fmp
        mstore(0x40, memPtr)
        // 返回
        return(results, sub(memPtr, results))
    }
}
```

### 8.3 exttload（瞬态存储版）

```solidity
// Exttload.sol
function exttload(bytes32 slot) external view returns (bytes32) {
    assembly ("memory-safe") {
        mstore(0, tload(slot))    // TLOAD 替代 SLOAD
        return(0, 0x20)
    }
}

function exttload(bytes32[] calldata slots) external view returns (bytes32[] memory) {
    assembly ("memory-safe") {
        let memptr := mload(0x40)
        let start := memptr
        mstore(memptr, 0x20)
        mstore(add(memptr, 0x20), slots.length)
        memptr := add(memptr, 0x40)
        let end := add(memptr, shl(5, slots.length))
        let calldataptr := slots.offset
        for {} 1 {} {
            mstore(memptr, tload(calldataload(calldataptr)))
            memptr := add(memptr, 0x20)
            calldataptr := add(calldataptr, 0x20)
            if iszero(lt(memptr, end)) { break }
        }
        return(start, sub(end, start))
    }
}
```

和 `extsload` 不同，当前 `Exttload.sol` 只有 **两个**重载：单 slot 与离散 slot 列表，没有连续区间版。协议外部如果要观测 `CurrencyDelta`、`NonzeroDeltaCount`、`CurrencyReserves` 这类瞬态状态，需要先自行按库里的规则推导 slot，再调用 `exttload`。

---

## 9. 模式七：StateLibrary — 嵌套 Mapping Slot 推导

**文件**：`src/libraries/StateLibrary.sol`

### 9.1 Solidity Mapping 的 Slot 计算规则

```
mapping(K => V) 存在 storage slot S 处：
  V 的 slot = keccak256(abi.encodePacked(key, S))
            = keccak256(key ++ uint256(S))  // 32 bytes 拼接

嵌套 mapping：
  pools[poolId] → slot P = keccak256(poolId ++ 6)
    （6 = pools 在 PoolManager 中的 slot index）

  pools[poolId].ticks[tick] → slot T:
    pools 内 ticks 在 Pool.State 的 offset = 4
    ticks mapping slot = P + 4
    T = keccak256(tick ++ (P + 4))
```

### 9.2 StateLibrary 中的 Slot 推导链

```solidity
// PoolManager 的 storage layout:
// slot 0: ...
// slot 6: mapping(PoolId => Pool.State) pools;
bytes32 public constant POOLS_SLOT = bytes32(uint256(6));

// pools[poolId] 的 slot
function _getPoolStateSlot(PoolId poolId) internal pure returns (bytes32) {
    return keccak256(abi.encodePacked(PoolId.unwrap(poolId), POOLS_SLOT));
}

// pools[poolId].ticks[tick] 的 slot
function _getTickInfoSlot(PoolId poolId, int24 tick) internal pure returns (bytes32) {
    bytes32 stateSlot    = _getPoolStateSlot(poolId);
    bytes32 ticksMapping = bytes32(uint256(stateSlot) + TICKS_OFFSET); // offset = 4
    return keccak256(abi.encodePacked(int256(tick), ticksMapping));
    //                                 ^^^^^^^^^^^^
    //                                 tick 需要 pad 到 32 bytes（int256）
}

// 使用：extsload 批量读 3 个连续 slot（TickInfo 结构）
function getTickInfo(IPoolManager manager, PoolId poolId, int24 tick)
    internal
    view
    returns (
        uint128 liquidityGross,
        int128 liquidityNet,
        uint256 feeGrowthOutside0X128,
        uint256 feeGrowthOutside1X128
    )
{
    bytes32 slot = _getTickInfoSlot(poolId, tick);
    bytes32[] memory data = manager.extsload(slot, 3);  // 批量读 3 slot

    assembly ("memory-safe") {
        // data 是已经 ABI decode 完成的 bytes32[] memory，布局是：
        // [data+0]  → length
        // [data+32] → slot[0]（liquidityGross + liquidityNet）
        // [data+64] → slot[1]（feeGrowthOutside0X128）
        // [data+96] → slot[2]（feeGrowthOutside1X128）

        let firstWord := mload(add(data, 32))

        liquidityNet   := sar(128, firstWord)
        //  高 128 bit，int128，用 SAR 算术右移

        liquidityGross := and(firstWord, 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF)
        //  低 128 bit，uint128，AND mask

        feeGrowthOutside0X128 := mload(add(data, 64))
        feeGrowthOutside1X128 := mload(add(data, 96))
    }
}
```

### 9.3 Pool.State 内的字段 Offset 表

```
Pool.State struct storage layout：
  slot + 0:  slot0（sqrtPriceX96 | tick | protocolFee | lpFee）
  slot + 1:  feeGrowthGlobal0X128
  slot + 2:  feeGrowthGlobal1X128
  slot + 3:  liquidity（uint128，只用低 128 bit）
  slot + 4:  mapping(int24 => TickInfo) ticks
  slot + 5:  mapping(int16 => uint256) tickBitmap
  slot + 6:  mapping(bytes32 => Position.State) positions

对应 StateLibrary 常量：
  FEE_GROWTH_GLOBAL0_OFFSET = 1
  LIQUIDITY_OFFSET          = 3
  TICKS_OFFSET              = 4
  TICK_BITMAP_OFFSET        = 5
  POSITIONS_OFFSET          = 6
```

---

## 10. 模式八：CustomRevert — Scratch Space Revert

**文件**：`src/libraries/CustomRevert.sol`

### 10.1 传统 revert vs Assembly revert

```solidity
// 传统方式（编译器生成）：
revert TickSpacingTooLarge(tickSpacing);
// 编译器通常会：
// 1. 获取/推进 free memory pointer
// 2. 在连续内存中写 selector + args
// 3. revert(ptr, size)

// V4 方式：
TickSpacingTooLarge.selector.revertWith(tickSpacing);
// 对单参数重载，源码直接把 selector 写到 0x00，把参数写到 0x04：
//   mstore(0x00, selector)
//   mstore(0x04, arg)
//   revert(0x00, 0x24)
// 这样 returndata 恰好是连续的 4 + 32 bytes
```

### 10.2 各重载解析

```solidity
library CustomRevert {
    // 无参数：仅 4 字节 selector
    function revertWith(bytes4 selector) internal pure {
        assembly ("memory-safe") {
            mstore(0, selector)      // selector 写在 word 的高 4 字节（左对齐）
            revert(0, 0x04)          // 返回 4 字节
        }
    }

    // address 参数：selector + address（右对齐 32 bytes）
    function revertWith(bytes4 selector, address addr) internal pure {
        assembly ("memory-safe") {
            mstore(0x00, selector)
            mstore(0x04, and(addr, 0xffffffffffffffffffffffffffffffffffffffff))
            revert(0x00, 0x24)
        }
    }

    // int24 参数：selector + signextend 后的 int24
    function revertWith(bytes4 selector, int24 value) internal pure {
        assembly ("memory-safe") {
            mstore(0x00, selector)
            mstore(0x04, signextend(2, value))   // int24 → int256（ABI 要求有符号扩展）
            revert(0x00, 0x24)
        }
    }

    // 两个参数时，当前源码改用 fmp，避免和 scratch space 重叠写入
    function revertWith(bytes4 selector, address value1, address value2) internal pure {
        assembly ("memory-safe") {
            let fmp := mload(0x40)
            mstore(fmp, selector)
            mstore(add(fmp, 0x04), and(value1, 0xffffffffffffffffffffffffffffffffffffffff))
            mstore(add(fmp, 0x24), and(value2, 0xffffffffffffffffffffffffffffffffffffffff))
            revert(fmp, 0x44)
        }
    }
}
```

### 10.3 为什么把参数写到 `0x04`

当前 `main` 分支的单参数重载不再使用旧版本里常见的偏移技巧，而是直接把参数写到 `0x04`，让 revert data 在内存中天然连续。

```
memory[0x00..0x03]  = selector
memory[0x04..0x23]  = ABI word #1（32 bytes 参数）

所以：
  revert(0x00, 0x24)
  = [selector:4B][arg:32B]
```

这也是当前版本最值得记住的规则：

- 无参数：`mstore(0, sel); revert(0, 0x04)`
- 单参数：`mstore(0, sel); mstore(0x04, arg); revert(0, 0x24)`
- 双参数：通常改用 `fmp`，写成 `selector | arg1 | arg2` 的连续 68 bytes 再 `revert(fmp, 0x44)`

### 10.4 实际使用

```solidity
// 替换前（高 gas）：
if (key.tickSpacing > MAX_TICK_SPACING) {
    revert TickSpacingTooLarge(key.tickSpacing);
}

// 替换后（V4 方式）：
if (key.tickSpacing > MAX_TICK_SPACING) {
    TickSpacingTooLarge.selector.revertWith(key.tickSpacing);
}
```

---

## 11. 模式九：Currency.transfer — 低级 ERC20 调用

**文件**：`src/types/Currency.sol`

### 11.1 为什么手动构造 calldata

标准的 `IERC20.transfer(to, amount)` 调用会触发 Solidity 的 ABI 编码 + external call 机制，分配 memory 并执行多步处理。V4 手动构造 calldata，在 fmp 处写入，然后直接 `call`，更精确控制每一步。

### 11.2 完整实现

```solidity
function transfer(Currency currency, address to, uint256 amount) internal {
    bool success;

    if (currency.isAddressZero()) {
        // Native ETH 转账：直接 call with value
        assembly ("memory-safe") {
            success := call(gas(), to, amount, 0, 0, 0, 0)
        }
        if (!success) {
            CustomRevert.bubbleUpAndRevertWith(to, bytes4(0), NativeTransferFailed.selector);
        }
        return;
    }

    assembly ("memory-safe") {
        let fmp := mload(0x40)  // 获取 free memory pointer

        // ① 写入 transfer(address,uint256) 的 4-byte selector
        // selector = 0xa9059cbb，bytes4 左对齐，其余 28B 为 0
        mstore(fmp, 0xa9059cbb00000000000000000000000000000000000000000000000000000000)

        // ② 写入 `to` 参数（右对齐 address，mask 防脏高位）
        mstore(add(fmp, 4), and(to, 0xffffffffffffffffffffffffffffffffffffffff))

        // ③ 写入 `amount` 参数（uint256，全 32 bytes 有效，无需 mask）
        mstore(add(fmp, 36), amount)

        // ④ 发送 call，捕获返回值
        // ⚠️ 关键：这段写法依赖 Yul 生成的求值顺序
        success := and(
            or(
                // 分支 A：返回了 ≥32 字节，且第一个 word = 1（标准 ERC20 bool true）
                and(
                    eq(mload(0), 1),
                    gt(returndatasize(), 31)
                ),
                // 分支 B：无返回值（USDT 等非标准 ERC20）
                iszero(returndatasize())
            ),
            // 写在 and() 的第二个参数位置，编译后会先执行
            call(
                gas(),      // 转发全部 gas
                currency,   // token 合约地址
                0,          // msg.value = 0
                fmp,        // calldata 起点
                68,         // calldata 长度 = 4 + 32 + 32
                0,          // returndata 写到 scratch space 0x00
                32          // 最多读 32 bytes 返回值
            )
        )

        // ⑤ 清理 memory（维护 memory-safe 约定）
        mstore(fmp,              0)
        mstore(add(fmp, 0x20),   0)
        mstore(add(fmp, 0x40),   0)
    }

    if (!success) {
        CustomRevert.bubbleUpAndRevertWith(
            Currency.unwrap(currency),
            IERC20Minimal.transfer.selector,
            ERC20TransferFailed.selector
        );
    }
}
```

### 11.3 call 顺序的 EVM 副作用陷阱

```
这段代码成立的关键不是“短路求值”，而是编译器把
`and(is_valid_return, call(...))`
生成为一段先执行 `call(...)`、再检查 `is_valid_return` 的栈式代码。

因此执行顺序可以理解为：
  1. 先执行 `call(...)`
  2. 返回值最多 32 bytes 被写到 scratch space `0x00..0x1f`
  3. 此时 `returndatasize()` 也已经变成这次 token 调用的返回长度
  4. 再判断：
       - 返回 32+ bytes 且首 word 为 1
       - 或者根本没有返回值

如果把 `call(...)` 与返回值检查表达式交换位置，后面的 `mload(0)` /
`returndatasize()` 就可能在错误时点被读取，从而误判兼容性。
```

### 11.4 兼容非标准 ERC20

```
ERC20 返回值规范（历史遗留问题）：

标准 ERC20（OpenZeppelin 等）：
  transfer() → returns (bool)
  返回 32 bytes：0x...0000...0001（true）或 0x...0000...0000（false）
  → 分支 A：eq(mload(0), 1) && gt(returndatasize(), 31) ✓

USDT, BNB 等非标准 ERC20：
  transfer() → 没有 return 语句（ABI 说有 bool，但实际不返回）
  → returndatasize() = 0
  → 分支 B：iszero(returndatasize()) ✓

恶意/错误实现：
  返回了数据但第一个 word ≠ 1 → 两个分支都不通过 → success = false → revert ✓
```

---

## 12. 模式十：Hooks.callHook — returndata 动态捕获

**文件**：`src/libraries/Hooks.sol`

### 12.1 完整实现

```solidity
function callHook(IHooks self, bytes memory data)
    internal
    returns (bytes memory result)
{
    bool success;
    assembly ("memory-safe") {
        // 发送 call，不预分配 returndata 空间
        success := call(
            gas(),             // 转发全部 gas（EIP-150 保留 1/64）
            self,              // hook 合约地址
            0,                 // msg.value = 0
            add(data, 0x20),   // calldata = data[32:]（跳过 bytes 的 length prefix）
            mload(data),       // calldata length = data.length
            0,                 // returndata offset = 0（不直接写 memory）
            0                  // returndata length = 0（后用 returndatacopy 捕获）
        )
    }

    if (!success) {
        CustomRevert.bubbleUpAndRevertWith(
            address(self),
            bytes4(data),          // 从 calldata 头部取 selector（bytes4 左对齐）
            HookCallFailed.selector
        );
    }

    // 动态捕获 returndata 到 memory
    assembly ("memory-safe") {
        // ① result 指向当前 fmp
        result := mload(0x40)

        // ② 推进 fmp = result + ceil32(returndatasize + 32)
        //    ceil32(x) = and(x + 31, ~31) = and(x + 0x1f, not(0x1f))
        //    +32 是为 length prefix 预留空间
        //    所以 ceil32(returndatasize + 32) = and(returndatasize + 0x3f, not(0x1f))
        mstore(0x40, add(result, and(add(returndatasize(), 0x3f), not(0x1f))))

        // ③ 写入 bytes 的 length prefix
        mstore(result, returndatasize())

        // ④ 复制 returndata 到 result[32:]
        returndatacopy(add(result, 0x20), 0, returndatasize())
    }
}
```

### 12.2 Memory 分配图解

```
调用前：                    调用后：
fmp = 0x80                  fmp = 0x80 + ceil32(retsize + 32)

memory:                     memory:
  ...                         ...
  0x80: ?                     0x80..0x9f  = retsize
  0xa0: ?                     0xa0..      = returndata[0..]
  ...                         ...
                              0xa0+retsize 之后按 32 字节对齐

result 指针 = 0x80
result.length = mload(0x80)
result data   = memory[0xa0..0xa0+retsize)
```

### 12.3 `and(returndatasize() + 0x3f, not(0x1f))` 的数学

```
not(0x1f) = 0xFFFF...FFE0（二进制：...1110 0000）

and(x, not(0x1f)) = 把 x 的低 5 位清零 = 向下对齐到 32 的倍数

我们要：new_fmp = result + 32(length) + ceil32(retsize)
                = result + ceil32(retsize + 32)   // 合并

ceil32(y) = and(y + 31, not(31)) = and(y + 0x1f, not(0x1f))
ceil32(retsize + 32) = and(retsize + 32 + 31, not(31))
                     = and(retsize + 63, not(31))
                     = and(retsize + 0x3f, not(0x1f))

所以：
new_fmp = result + and(returndatasize() + 0x3f, not(0x1f))

验证（retsize = 64 = 0x40）：
  and(0x40 + 0x3f, not(0x1f)) = and(0x7f, 0xffe0) = 0x60 = 96 = 32+64 ✓
  (需要 32B length + 64B data = 96B，对齐到 32 倍数)

验证（retsize = 65）：
  and(65 + 63, not(31)) = and(128, ~31) = 128
  而 32(length) + ceil32(65) = 32 + 96 = 128 ✓
```

---

## 13. 模式十一：ParseBytes — bytes memory 字段提取

**文件**：`src/libraries/ParseBytes.sol`

### 13.1 实现

```solidity
library ParseBytes {
    // 从 bytes memory 提取前 4 字节作为 selector
    function parseSelector(bytes memory returnData)
        internal
        pure
        returns (bytes4 selector)
    {
        assembly ("memory-safe") {
            selector := mload(add(returnData, 0x20))
            // returnData 指向 length prefix
            // +0x20 跳过 length，指向数据首 32 bytes
            // bytes4 赋值：EVM 取 word 的高 4 bytes（left-aligned）
            // = returnData[0..3] ✓
        }
    }

    // 从 hook 返回数据提取 lpFee（第三个 ABI word 的低 24 bit）
    function parseFee(bytes memory returnData)
        internal
        pure
        returns (uint24 lpFee)
    {
        assembly ("memory-safe") {
            lpFee := mload(add(returnData, 0x60))
            // returnData[64..95] = 第三个 ABI word
            // uint24 赋值：EVM 取 word 的低 3 bytes（right-aligned）
            // = ABI 编码的 uint24 值 ✓
        }
    }
}
```

### 13.2 EVM 类型对齐方向

```
EVM ABI 规范的对齐规则：

bytes / string / bytes4 / bytes32 等 bytesN 类型：
  → 左对齐（left-aligned），高位有效，低位补零
  → mload 后赋值给 bytes4 取 word 的高 4 字节

uint / int / address / bool 等数值类型：
  → 右对齐（right-aligned），低位有效，高位补零
  → mload 后赋值给 uint24 取 word 的低 3 字节

示例：
  memory word = 0x1234560000000000000000000000000000000000000000000000000000000000
  作为 bytes4 赋值 → 0x12345600（取高 4 字节）
  作为 uint24  赋值 → 0x000000（取低 3 字节）

  memory word = 0x0000000000000000000000000000000000000000000000000000000000123456
  作为 bytes4 赋值 → 0x00000000（取高 4 字节，丢失数据！）
  作为 uint24  赋值 → 0x123456（取低 3 字节，正确）

关键点：parseFee 解析的 lpFee 是 uint24，
hook 按 ABI 编码返回右对齐的值，mload 后直接赋给 uint24 自动截取低 3 字节。✓
```

---

## 14. 模式十二：UnsafeMath — 有意绕过 Overflow 检查

**文件**：`src/libraries/UnsafeMath.sol`

### 14.1 Solidity 0.8+ 的 Checked Arithmetic

```solidity
// Solidity 0.8+：
uint256 a = x * y;
// 编译器生成：
// 1. 检查 x * y 是否溢出（额外 ~5-10 gas）
// 2. 若溢出则 revert
// 3. 否则存储结果

// 在数学上已证明不会溢出时，这个检查是纯粹的 gas 浪费
```

### 14.2 UnsafeMath 实现

```solidity
library UnsafeMath {
    // 简单乘除（无 overflow/div-by-zero 检查）
    // 前置条件：调用方保证 x*y ≤ 2^256-1 且 d ≠ 0
    function simpleMulDiv(uint256 x, uint256 y, uint256 d)
        internal
        pure
        returns (uint256 z)
    {
        assembly ("memory-safe") {
            z := div(mul(x, y), d)
            // mul：无检查乘法（溢出时截断，不 revert）
            // div：无检查除法（d=0 时返回 0，不 revert）
        }
    }

    // 向上取整除法：ceil(x/d)
    function divRoundingUp(uint256 x, uint256 d)
        internal
        pure
        returns (uint256 z)
    {
        assembly ("memory-safe") {
            z := add(div(x, d), gt(mod(x, d), 0))
            // div(x, d) = 商
            // gt(mod(x, d), 0) = 1 if 有余数，0 otherwise
            // 若有余数则商 +1 = 向上取整
        }
    }
}
```

### 14.3 V4 的调用场景与安全证明

```solidity
// Pool.sol 中的使用（feeGrowthGlobal 累积）：
if (amount0 > 0) {
    state.feeGrowthGlobal0X128 +=
        UnsafeMath.simpleMulDiv(amount0, FixedPoint128.Q128, liquidity);
}

// 安全性证明：
// - amount0 ≤ type(int128).max = 2^127 - 1
// - Q128 = 2^128
// - 乘积 ≤ (2^127 - 1) * 2^128 < 2^255 < 2^256 - 1
// → 不会溢出 uint256 ✓
// - liquidity > 0（调用前已检查 NoLiquidityToReceiveFees）
// → 不会 div by zero ✓
```

---

## 15. 五大设计原则总结

### 原则一：Scratch Space 是免费的工作区

```
EVM 规范保证 memory[0x00..0x3f] 是 scratch space，
Solidity 也明确标注可被 assembly 自由使用（即使 memory-safe 约束下）。

V4 复用 scratch space 的三个场景：
  1. keccak 输入（CurrencyDelta._computeSlot）
  2. revert data 构造（CustomRevert）
  3. call 返回值读取（Currency.transfer 的 mload(0)）

原则：优先把短生命周期数据放进 scratch space，尽量避免不必要的 memory 扩展。
```

### 原则二：符号语义绝不混淆

```
有符号整数字段（int128, int24）：
  提取高位 → SAR（算术右移，高位补符号）
  提取低位 → SIGNEXTEND（以特定字节为符号位扩展）

无符号整数字段（uint128, uint24）：
  提取高位 → SHR + AND（逻辑右移后 mask）
  提取低位 → AND mask

混淆规则 = silent bug。
V4 代码库：BalanceDelta 的 amount0 用 sar，amount1 用 signextend，
           Slot0 的 tick 用 signextend，fee 用 shr+and。
```

### 原则三：EIP-1153 适合承载交易内共享的临时状态

```
V4 中最适合放进 transient storage 的，是那些
"在当前交易里跨调用共享、交易结束后立即失效"的状态：

  Lock 状态         SSTORE(bool)        → tstore(bool)
  NonzeroDeltaCount SSTORE(uint)        → tstore(uint)
  CurrencyReserves  SSTORE(addr + uint) → 2 x tstore
  CurrencyDelta     SSTORE mapping      → tstore keccak slot

它不是对 stack / memory / scratch space 的替代，而是补上了
"需要像 storage 一样跨调用可见，但又不该持久化" 这一空档。

从 gas 角度看，收益通常是非常显著的，但具体节省幅度仍取决于冷热槽、调用路径与交易上下文，更适合表述为量级优化，而不是固定常数。
```

### 原则四：Bytecode 体积驱动 API 设计

```
每个额外函数 = 额外 bytecode：
  - 函数签名：~4 bytes
  - 函数体：50-200 bytes（依复杂度）
  - jump table entry：~5 bytes

V4 的 bytecode 压缩策略：
  extsload     代替 N 个 view getter  → 节省 N×100 bytes
  CustomRevert 代替 N 个 revert 语句  → 节省 N×30 bytes
  assembly 直接 return              → 省去 ABI 编码层
```

### 原则五：有意识的信任边界委托

```
以下操作故意不保护：
  NonzeroDeltaCount.decrement() → 无 underflow 检查
  UnsafeMath.simpleMulDiv()     → 无 overflow/zero 检查
  extsload()                    → 暴露任意 slot

设计原则：
  - 不变量由调用层保证，不在底层重复验证（节省 gas）
  - 在注释中明确记录：⚠️ "Potential to underflow"
  - 在审计文档中明确信任边界

审计责任：验证所有调用路径是否满足底层函数的前置条件。
```

---

## 16. 安全审计 Checklist

基于本文分析，以下是 V4 assembly 的关键审计点：

### ✅ 必查项

```
□ 1. 所有 signed int 提取是否使用 SAR/SIGNEXTEND 而非 SHR/AND？
     → BalanceDelta.amount0/1，Slot0.tick，TickInfo.liquidityNet

□ 2. CurrencyDelta._computeSlot 的参数顺序是否正确？
     → mstore(0, target)，mstore(32, currency)
     → 顺序颠倒会产生不同 slot，导致账本错误

□ 3. NonzeroDeltaCount.decrement 的所有调用路径是否保证 count > 0？
     → 只在 _accountDelta 中 next==0 时调用，验证 previous != 0

□ 4. Currency.transfer 中 call 与返回值检查的相对位置是否保持源码假设？
     → 错误改写求值顺序会破坏 `returndatasize()` / `mload(0)` 的时序

□ 5. extsload 暴露的 slot 是否可被攻击者利用？
     → 检查是否存在依赖 storage 值"不被外部知晓"的假设

□ 6. UnsafeMath 的调用处是否都有数学上的不溢出证明？
     → 检查所有 simpleMulDiv 调用的输入范围

□ 7. 所有瞬态 slot 常量是否都使用 keccak256("name") - 1 模式？
     → 防止与 Solidity 正常 storage slot 碰撞

□ 8. keccak256 输入中 address 类型是否经过 mask？
     → and(addr, 0xffffffffffffffffffffffffffffffffffffffff)
     → 脏高位可导致不同 hash 但相同地址

□ 9. callHook 的 returndata 长度检查是否足够严格？
     → result.length >= 32 以包含 selector，否则 InvalidHookResponse

□ 10. memory-safe 标注的 assembly 是否真的不越界访问？
      → 检查所有 mstore 的地址是否在 [scratch] ∪ [fmp, fmp+allocated)
```

### ⚠️ 高风险陷阱

```
① returndatasize() 的时序依赖：
   在任何 call/staticcall 之后，returndatasize 会被更新。
   多个 call 之间的 or/and 逻辑必须小心求值顺序。

② bytes4 vs uint32 的对齐方向：
   bytes4 左对齐（高位），uint32 右对齐（低位）。
   错误赋值方向 = 静默取到错误字节。

③ slot 碰撞（尤其是 mapping slot）：
   keccak256(key, slot) 的两个参数各自的 ABI padding 必须一致。
   V4 的 CurrencyDelta 用 64 字节 packed encoding，而非 abi.encode。

④ 瞬态存储的跨事务假设：
   tstore 数据在交易结束后消失。
   任何依赖"下一笔交易能读到 tstore 数据"的代码都是严重 bug。
```

---

## 17. 参考资料

| 资源 | 链接 |
|------|------|
| Uniswap V4 Core 源码 | [github.com/Uniswap/v4-core](https://github.com/Uniswap/v4-core) |
| Uniswap V4 Whitepaper | [uniswap.org/whitepaper-v4.pdf](https://uniswap.org/whitepaper-v4.pdf) |
| EIP-1153 Transient Storage | [eips.ethereum.org/EIPS/eip-1153](https://eips.ethereum.org/EIPS/eip-1153) |
| EIP-7751 Wrapped Errors | [eips.ethereum.org/EIPS/eip-7751](https://eips.ethereum.org/EIPS/eip-7751) |
| EVM Codes (Opcode 参考) | [evm.codes](https://www.evm.codes) |
| Solidity Assembly 文档 | [docs.soliditylang.org/en/latest/assembly.html](https://docs.soliditylang.org/en/latest/assembly.html) |
| StateLibrary 文档 | [docs.uniswap.org/contracts/v4/reference/core/libraries/StateLibrary](https://docs.uniswap.org/contracts/v4/reference/core/libraries/StateLibrary) |
| BalanceDelta 指南 | [docs.uniswap.org/contracts/v4/reference/core/types/balancedelta-guide](https://docs.uniswap.org/contracts/v4/reference/core/types/balancedelta-guide) |
| Cyfrin V4 Swap Deep Dive | [cyfrin.io/blog/uniswap-v4-swap-deep-dive](https://www.cyfrin.io/blog/uniswap-v4-swap-deep-dive-into-execution-and-accounting) |
| OpenZeppelin V4 Audit | [openzeppelin.com/news/uniswap-v4-core-audit](https://www.openzeppelin.com/news/uniswap-v4-core-audit) |

---

*如有勘误或补充，欢迎在评论区讨论。*
