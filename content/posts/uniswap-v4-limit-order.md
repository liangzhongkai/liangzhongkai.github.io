+++
date = '2026-04-11T10:00:00+08:00'
draft = false
title = 'Uniswap V4 Limit Order'
tags = ['web3']
summary = '结合 v4-core 与官方 LimitOrder 示例，解释 Uniswap V4 如何通过 Hook、unlock 与 ERC-6909 实现链上限价单。'
+++

# Uniswap V4 Limit Order

> **目标读者**：熟悉 Uniswap V3 集中流动性基础，想理解 V4 如何通过 Hook 实现链上限价单的开发者。

---

## 一、前置概念：为什么单 tick 区间等价于限价单

理解 V4 Limit Order 之前，需要牢固掌握两个 V3 概念。

### 1.1 集中流动性与 tick

V3 把价格轴离散化为一系列 **tick**，每个 tick 对应一个具体价格：

```
price = 1.0001^tick
```

LP 提供流动性时指定 `[tickLower, tickUpper]` 区间，资金只在价格处于该区间内时参与撮合。

### 1.2 价格穿越 tick 时，资产构成会发生翻转

这是理解 Limit Order 的关键物理直觉。

假设某个区间 `[tickLower, tickUpper]` 当前**全部由 USDC 组成**，也就是当前价格在区间上方、尚未进入：

```
价格从上往下穿越该区间的过程：
  进入 tickUpper：开始消耗 USDC，换入 ETH
  穿越整个区间：USDC 100% 耗尽，换成 ETH
  离开 tickLower：该区间全部由 ETH 组成
```

反之，价格从下往上穿越，ETH 会被 100% 换成 USDC。

**把区间压缩到只有一个 `tickSpacing` 宽，就可以近似成限价单：价格一旦穿过目标区间，仓位就从单边资产翻转成另一边资产。**

### 1.3 V4 新增了什么

V3 缺少在价格穿越时自动触发逻辑的能力。V4 的两个核心变化补齐了这一点：

- **Hook 系统**：可以在 `afterSwap` 中检测价格是否跨过某个 tick，并在同一笔 swap 的回调流程里触发结算
- **Singleton PoolManager + unlock / delta 记账模型**：所有池状态集中在 `PoolManager`，`swap` / `modifyLiquidity` / `take` / `settle` / `mint` / `burn` 等动作都发生在一次 `unlock` 会话内，先记 delta，最后要求净额归零

---

## 二、架构总览

如果严格按 `v4-core` 和官方 `LimitOrder` 示例来理解，真实链路其实是这样的：

```
用户                    LimitOrder Hook                PoolManager
 │                            │                             │
 │── place(...) ─────────────▶│                             │
 │                            │── unlock(...) ────────────▶│
 │                            │   └─ unlockCallbackPlace() │
 │                            │      └─ modifyLiquidity(+L)│ 挂单
 │                            │      └─ settle(...)        │ 用户把单边资产付给池子
 │
 │   (Swapper 发起 swap，价格跨过挂单所在 tickLower)
 │
 │                            │◀──── afterSwap(...) ───────│
 │                            │      计算 crossed ticks     │
 │                            │      找到对应 epoch         │
 │                            │      modifyLiquidity(-L)    │ 移除已成交流动性
 │                            │      mint(hook, id, amt)    │ 把应得资产铸成 ERC-6909
 │
 │── withdraw(epoch, to) ────▶│                             │
 │                            │── unlock(...) ────────────▶│
 │                            │   └─ unlockCallbackWithdraw│
 │                            │      └─ burn(hook, id, amt)│ 销毁 Hook 持有的 claim token
 │                            │      └─ take(currency, to)  │ 实物转给用户
 │◀──────────── token ────────│                             │
```

整个流程依然可以概括成四步：**挂单 → 穿越成交 → 提现 → 撤单**。但要注意，核心池操作并不是在普通外部函数里直接执行，而是必须放在 `unlock` 回调里。

---

## 三、完整例子：按官方示例理解一笔限价买单

### 场景设定

为了避免把 **token 顺序、decimals、tick 数值换算** 混在一起，这里不再写容易误导的“1900 对应 tick=196020”这类具体数字，而是直接用更准确的源码语义描述：

| 参数 | 含义 |
|---|---|
| Pool | `currency0 = ETH`，`currency1 = USDC`，`tickSpacing = 60` |
| Alice 的意图 | 当价格下跌到目标区间时，用 USDC 买入 ETH |
| 挂单方向 | `zeroForOne = false` |
| 挂单位置 | 某个 **低于当前价格** 的 `tickLower`，区间宽度为一个 `tickSpacing` |

对这类买单来说，当前价格必须在区间上方，这样这段流动性在挂入时才会是 **纯 `token1`（USDC）**。这也是官方示例在 `unlockCallbackPlace()` 里用 `delta` 符号检查 order 是否“在正确侧” 的原因。

---

### Step 1：挂单 —— `place()`

Alice 调用的不是一个“直接往 `PoolManager` 塞钱”的函数，而是官方示例里的：

```solidity
limitOrder.place(key, tickLower, false, liquidity);
```

它的真实执行链路是：

```solidity
function place(PoolKey calldata key, int24 tickLower, bool zeroForOne, uint128 liquidity) external {
    manager.unlock(
        abi.encodeCall(
            this.unlockCallbackPlace,
            (key, tickLower, zeroForOne, int256(uint256(liquidity)), msg.sender)
        )
    );
    ...
}
```

真正的 `modifyLiquidity(+L)` 发生在 `unlockCallbackPlace()` 里：

```solidity
(BalanceDelta delta,) = manager.modifyLiquidity(
    key,
    IPoolManager.ModifyLiquidityParams({
        tickLower: tickLower,
        tickUpper: tickLower + key.tickSpacing,
        liquidityDelta: liquidityDelta,
        salt: 0
    }),
    ZERO_BYTES
);
```

然后 Hook 根据 `delta` 判断这是不是一个合法的单边限价单：

```solidity
if (delta.amount0() < 0) {
    if (delta.amount1() != 0) revert InRange();
    if (!zeroForOne) revert CrossedRange();
    key.currency0.settle(manager, owner, uint256(uint128(-delta.amount0())), false);
} else {
    if (delta.amount0() != 0) revert InRange();
    if (zeroForOne) revert CrossedRange();
    key.currency1.settle(manager, owner, uint256(uint128(-delta.amount1())), false);
}
```

这段检查有两个关键含义：

- 如果挂进去后需要同时支付两种资产，说明区间已经和现价重叠，`revert InRange()`
- 如果支付的是错误方向的单边资产，说明挂单放到了价格另一侧，`revert CrossedRange()`

对于 Alice 这种“用 USDC 买 ETH、等价格下跌”的订单，合法情况应该是：

```solidity
delta.amount0() == 0
delta.amount1() < 0
zeroForOne == false
```

也就是：Hook 需要替 Alice 向池子结算纯 `token1`（USDC）。

挂单完成后，官方示例不是记 `orders[pool][tick][user] = amount`，而是建立一套 **epoch 账本**：

```solidity
Epoch epoch = getEpoch(key, tickLower, zeroForOne);
epochInfos[epoch].liquidityTotal += liquidity;
epochInfos[epoch].liquidity[msg.sender] += liquidity;
```

---

### Step 2：价格穿越 —— `afterSwap()` 找到被 fill 的 tick

Bob 发起一笔 `zeroForOne = true` 的 swap，卖出 ETH，推动价格下跌，跨过 Alice 挂单所在的 `tickLower`：

```solidity
poolManager.swap(poolKey, SwapParams({
    zeroForOne:         true,
    amountSpecified:    -5 ether,
    sqrtPriceLimitX96:  ...
}));
```

如果那个区间此前装的是 Alice 的 USDC，那么穿越后，这段流动性就会变成 ETH。

```
tick [tickLower, tickLower + tickSpacing] 穿越前：装着 Alice 的 USDC
tick [tickLower, tickLower + tickSpacing] 穿越后：装着成交后的 ETH
```

接下来 `PoolManager` 会回调 Hook 的 `afterSwap()`。官方示例不是缓存 `tickBefore` 再手写一段“从 `tickAfter` 到 `tickBefore` 遍历”的逻辑，而是维护每个池的 `tickLowerLast`，用它和当前 tick 推出 **这次到底跨过了哪些完整 tickLower**：

```solidity
function afterSwap(...) external override onlyByManager returns (bytes4, int128) {
    (int24 tickLower, int24 lower, int24 upper) = _getCrossedTicks(key.toId(), key.tickSpacing);
    if (lower > upper) return (LimitOrder.afterSwap.selector, 0);

    // 注意：被 fill 的挂单方向和 swap 方向相反
    bool zeroForOne = !params.zeroForOne;
    for (; lower <= upper; lower += key.tickSpacing) {
        _fillEpoch(key, lower, zeroForOne);
    }

    setTickLowerLast(key.toId(), tickLower);
    return (LimitOrder.afterSwap.selector, 0);
}
```

这里最关键的细节是：

- `params.zeroForOne = true` 表示 swapper 正在卖 `token0`，让价格下跌
- 但被成交的 limit order 是此前在下方挂着、等待“价格跌下来买入 `token0`” 的那一侧
- 所以官方示例明确写了 `bool zeroForOne = !params.zeroForOne`

这也是很多读者第一次看源码时最容易搞反的地方。

---

### Step 3：自动成交 —— `_fillEpoch()`

在看 fill 之前，先复习 `BalanceDelta` 的语义：

```
正值（+）= PoolManager 欠调用者，调用者可以 take() 出去
负值（-）= 调用者欠 PoolManager，调用者需要 settle() 进去
```

`modifyLiquidity` 返回的 `BalanceDelta` 表示：这次改动结束后，调用者与 `PoolManager` 之间分别在两种资产上还剩多少净额要结。

官方示例中，`afterSwap()` 找到被跨过的 tick 后，会调用：

```solidity
function _fillEpoch(PoolKey calldata key, int24 lower, bool zeroForOne) internal {
    Epoch epoch = getEpoch(key, lower, zeroForOne);
    if (!epoch.equals(EPOCH_DEFAULT)) {
        EpochInfo storage epochInfo = epochInfos[epoch];
        epochInfo.filled = true;

        (uint256 amount0, uint256 amount1) =
            _unlockCallbackFill(key, lower, -int256(uint256(epochInfo.liquidityTotal)));

        epochInfo.token0Total += amount0;
        epochInfo.token1Total += amount1;

        setEpoch(key, lower, zeroForOne, EPOCH_DEFAULT);
    }
}
```

真正的移除发生在 `_unlockCallbackFill()`：

```solidity
(BalanceDelta delta,) = manager.modifyLiquidity(
    key,
    IPoolManager.ModifyLiquidityParams({
        tickLower: tickLower,
        tickUpper: tickLower + key.tickSpacing,
        liquidityDelta: liquidityDelta,
        salt: 0
    }),
    ZERO_BYTES
);

if (delta.amount0() > 0) {
    manager.mint(address(this), key.currency0.toId(), uint128(delta.amount0()));
}
if (delta.amount1() > 0) {
    manager.mint(address(this), key.currency1.toId(), uint128(delta.amount1()));
}
```

这里有三个必须精确理解的点：

1. `modifyLiquidity(-L)` 拿回的是 **position 当前真实持有的资产**，不是“最初存进去的资产”  
   如果区间已经被完整穿越，那么原来挂进去的 USDC 现在通常已经变成 ETH。

2. 官方示例并没有假设 fill 后一定只有 `token0`  
   它同时累计 `token0Total` 和 `token1Total`，因为还要兼容手续费、边界情况和撤单时的残余结算。

3. `mint` 的真实签名是：

```solidity
manager.mint(address(this), currency.toId(), amount);
```

也就是“给谁铸”“铸哪种 id”“铸多少”，而不是 `mint(currency, to, amount)`。

此时资产并没有立刻被 `take()` 给 Alice，而是先以 ERC-6909 余额的形式记在 `PoolManager.balanceOf[address(this)][id]` 上，等待后续提现。

---

### Step 4：提现 —— `withdraw()`

官方示例里的用户提现入口叫 `withdraw(Epoch epoch, address to)`，不是 `claimOrder()`。

```solidity
function withdraw(Epoch epoch, address to) external returns (uint256 amount0, uint256 amount1) {
    EpochInfo storage epochInfo = epochInfos[epoch];
    if (!epochInfo.filled) revert NotFilled();

    uint128 liquidity = epochInfo.liquidity[msg.sender];
    delete epochInfo.liquidity[msg.sender];

    uint128 liquidityTotal = epochInfo.liquidityTotal;
    amount0 = FullMath.mulDiv(epochInfo.token0Total, liquidity, liquidityTotal);
    amount1 = FullMath.mulDiv(epochInfo.token1Total, liquidity, liquidityTotal);

    epochInfo.token0Total -= amount0;
    epochInfo.token1Total -= amount1;
    epochInfo.liquidityTotal = liquidityTotal - liquidity;

    manager.unlock(
        abi.encodeCall(this.unlockCallbackWithdraw, (epochInfo.currency0, epochInfo.currency1, amount0, amount1, to))
    );
}
```

`unlockCallbackWithdraw()` 里做两步事：

```solidity
if (token0Amount > 0) {
    manager.burn(address(this), currency0.toId(), token0Amount);
    manager.take(currency0, to, token0Amount);
}
if (token1Amount > 0) {
    manager.burn(address(this), currency1.toId(), token1Amount);
    manager.take(currency1, to, token1Amount);
}
```

注意这里的真实 API：

- `burn(address from, uint256 id, uint256 amount)`
- `take(Currency currency, address to, uint256 amount)`

此外，`balanceOf` 这个 storage 来自 `v4-core/src/ERC6909.sol`，`ERC6909Claims.sol` 只是在其上补了 `_burnFrom()` 逻辑。

所以完整关系应该理解为：

```solidity
// PoolManager 继承 ERC6909Claims -> ERC6909
mapping(address owner => mapping(uint256 id => uint256 balance)) public balanceOf;
```

其中 `id` 就是 `currency.toId()` 对应的 ERC-6909 token id。

---

### Step 5：撤单 —— `kill()`

官方示例的撤单入口叫 `kill()`：

```solidity
function kill(PoolKey calldata key, int24 tickLower, bool zeroForOne, address to) external
```

它首先检查这一档订单是否已经被 fill：

```solidity
Epoch epoch = getEpoch(key, tickLower, zeroForOne);
EpochInfo storage epochInfo = epochInfos[epoch];
if (epochInfo.filled) revert Filled();
```

随后通过 `unlockCallbackKill()` 把用户对应那部分 liquidity 移除，并把拿回的实物直接 `take` 给用户：

```solidity
(BalanceDelta delta,) = manager.modifyLiquidity(
    key,
    IPoolManager.ModifyLiquidityParams({
        tickLower: tickLower,
        tickUpper: tickUpper,
        liquidityDelta: liquidityDelta,
        salt: 0
    }),
    ZERO_BYTES
);

if (delta.amount0() > 0) {
    key.currency0.take(manager, to, uint256(uint128(delta.amount0())), false);
}
if (delta.amount1() > 0) {
    key.currency1.take(manager, to, uint256(uint128(delta.amount1())), false);
}
```

这里比“简单撤回本金”更细的一点是：官方示例还专门处理了 **手续费归属**。如果不是最后一个撤单者，会先用一次 `liquidityDelta = 0` 的 `modifyLiquidity()` 把已累积 fees 提出来，留给剩余挂单者，避免有人通过“同步挂单再立刻撤单”去薅已经形成的手续费。

---

## 四、两个核心概念辨析

### 4.1 流动性"被穿越使用" vs position "被 Hook 主动移除"

这是最容易混淆的地方，必须区分清楚：

| 事件 | 触发者 | 发生时机 | 对 position 的影响 |
|---|---|---|---|
| 流动性被 swap **穿越使用** | PoolManager AMM 计算 | swap 跨过 tick 时 | position 记录仍然存在，但区间内资产构成发生翻转 |
| position 被 Hook **主动移除** | Hook 调 `modifyLiquidity(-L)` | `afterSwap` 的 fill 流程中 | position 记录减少或清零，delta 结算到 Hook |

**AMM 在 swap 过程中“用到”这段流动性，不等于这段 position 自动从池子里消失。官方示例必须在 `afterSwap` 里显式调用 `modifyLiquidity(-L)`，才能把成交后的资产真正结算出来。**

如果 Hook 忘记在穿越后把这段 position 移掉，它就会继续留在池里。下一次价格反向穿越时，这段仓位可能又被反向交易，Hook 内部账本和池中真实资产就会失配。

### 4.2 ERC-6909 claim token vs Hook 的内部账本

| | ERC-6909 `balanceOf` | Hook 的 `epochInfo.liquidity` |
|---|---|---|
| 位置 | `PoolManager` 继承的 `ERC6909` storage | Hook 合约 storage |
| 记录粒度 | **Hook 整体**持有多少某 token | **每个用户**在某个 epoch 占多少 liquidity |
| 标准 | ERC-6909（可转让、可查询） | 自定义 mapping |
| 解决的问题 | Hook ↔ PoolManager 的跨 tx 结算 | Hook ↔ 用户的份额分账 |

```
PoolManager 视角（只认 Hook 合约地址）：
  balanceOf[hookAddress][ETH_id] = 1_000_000 gwei   // 整体

Hook 内部视角（细分到每个用户）：
  epochInfos[epoch_A].liquidity[Alice] = 600_000
  epochInfos[epoch_A].liquidity[Bob]   = 400_000
```

ERC-6909 不知道 Hook 内部如何分配，`epochInfo.liquidity` 是 Hook 自己维护的第二层会计账本。两者共同构成完整的结算体系：**ERC-6909 负责 Hook 和 `PoolManager` 之间的资产债权，epoch 账本负责 Hook 和用户之间的份额分配。**

---

## 五、为什么官方示例必须在 `afterSwap` 同步 fill

如果你采用的是和官方示例一样的设计：**先把真实 liquidity 挂进池子，等价格穿越后再移除**，那么 `afterSwap` 里就不能只处理一部分 crossed ticks。

原因链条：

```
1. afterSwap 被调用时，本次 crossed 的 tick 已经发生了真实状态变化
2. 被跨过的单 tick position 现在装的是“穿越后的资产”
3. 如果 Hook 只 fill 一部分，剩余 position 还继续躺在池里
4. 下一次价格反向经过时，这部分仓位可能再次被 swap 使用
5. Hook 自己的结算账本和池中真实资产会逐渐分叉
```

因此，官方示例选择的就是最直接的方案：**在 `afterSwap` 中把这次跨过的 epoch 全部同步处理完**。

---

## 六、完整流程状态机

```
Alice 挂单
  │  place() -> unlockCallbackPlace()
  │  把单边资产结算进 PoolManager
  ▼
流动性锁定：position.liquidity = L，epochInfo 记录用户份额
  │
  │  Bob swap：价格跨过该 tickLower
  ▼
tick 区间资产翻转：挂单资产变成成交后的另一侧资产
  │
  │  afterSwap 回调
  ▼
Hook 找到 crossed ticks，并对对应 epoch 调 modifyLiquidity(-L)
  └─ 得到正向 delta
  └─ 把正 delta 铸成 Hook 持有的 ERC-6909 余额
  │
  ▼
epochInfo[epoch].filled = true
epochInfo[epoch].token0Total / token1Total 更新
  │
  │  Alice 任意时间 withdraw(epoch, to)
  ▼
unlockCallbackWithdraw()
  └─ burn(address(this), id, amount)
  └─ take(currency, Alice, amount)
  │
  ▼
Alice 收到按 epoch 份额分配的成交资产
```

---

## 七、源码对应关系速查

`v4-core`:

```text
src/
├── PoolManager.sol
│   ├── unlock()                     ← 所有池操作的入口会话
│   ├── swap()                       ← 触发 beforeSwap / afterSwap
│   ├── modifyLiquidity()            ← 挂单 / 移除流动性
│   ├── take(currency, to, amount)   ← 把实物转给接收者
│   ├── settle() / settleFor()       ← 偿还负 delta
│   ├── mint(to, id, amount)         ← 铸造 ERC-6909 claim token
│   └── burn(from, id, amount)       ← 销毁 ERC-6909 claim token
│
├── ERC6909.sol
│   └── balanceOf[owner][id]         ← ERC-6909 余额定义在这里
│
└── ERC6909Claims.sol
    └── _burnFrom(...)               ← 在 ERC6909 上补充 burnFrom 逻辑
```

`v4-periphery` 官方示例（`example-contracts` 分支）：

```text
LimitOrder.sol
├── place()                          ← 外部挂单入口
├── unlockCallbackPlace()            ← 真正执行 modifyLiquidity(+L) + settle
├── afterSwap()                      ← 计算 crossed ticks
├── _fillEpoch()                     ← 标记 filled，累计成交资产
├── _unlockCallbackFill()            ← 真正执行 modifyLiquidity(-L) + mint
├── withdraw()                       ← 外部提现入口
├── unlockCallbackWithdraw()         ← burn + take
├── kill()                           ← 外部撤单入口
├── unlockCallbackKill()             ← 撤单并处理手续费归属
├── tickLowerLasts                   ← 上次 tickLower 快照
├── epochs                           ← (key, tickLower, zeroForOne) -> epoch
└── epochInfos[epoch]                ← filled / token0Total / token1Total / liquidity
```

### 7.1 官方源码链接索引

- `v4-core / PoolManager.sol`：
  [https://github.com/Uniswap/v4-core/blob/main/src/PoolManager.sol](https://github.com/Uniswap/v4-core/blob/main/src/PoolManager.sol)
- `v4-core / ERC6909.sol`：
  [https://github.com/Uniswap/v4-core/blob/main/src/ERC6909.sol](https://github.com/Uniswap/v4-core/blob/main/src/ERC6909.sol)
- `v4-core / ERC6909Claims.sol`：
  [https://github.com/Uniswap/v4-core/blob/main/src/ERC6909Claims.sol](https://github.com/Uniswap/v4-core/blob/main/src/ERC6909Claims.sol)
- `v4-core / IHooks.sol`：
  [https://github.com/Uniswap/v4-core/blob/main/src/interfaces/IHooks.sol](https://github.com/Uniswap/v4-core/blob/main/src/interfaces/IHooks.sol)
- `v4-periphery / LimitOrder.sol`（官方示例分支）：
  [https://github.com/Uniswap/v4-periphery/blob/example-contracts/contracts/hooks/examples/LimitOrder.sol](https://github.com/Uniswap/v4-periphery/blob/example-contracts/contracts/hooks/examples/LimitOrder.sol)
- `v4-periphery / README.md`：
  [https://github.com/Uniswap/v4-periphery/blob/main/README.md](https://github.com/Uniswap/v4-periphery/blob/main/README.md)

如果你准备边读边对照源码，推荐阅读顺序是：`PoolManager.sol` -> `IHooks.sol` -> `LimitOrder.sol` -> `ERC6909.sol` / `ERC6909Claims.sol`。

---

## 八、总结

结合 `v4-core` 和官方 `LimitOrder` 示例，正确的认知链条应该是：

```
单 tickSpacing 宽度的集中流动性可以近似表达限价单
  → 挂单时必须处于当前价格一侧，保证它是单边资产
  → Hook 不能直接调用池操作，必须通过 PoolManager.unlock() 进入回调
  → swap 跨过 tick 后，afterSwap() 找到 crossed ticks
  → 对应 epoch 的 liquidity 要立刻 modifyLiquidity(-L) 移出
  → 拿到的正 delta 通过 mint(to, id, amount) 记成 Hook 持有的 ERC-6909 余额
  → Hook 再用 epochInfos[epoch].liquidity[user] 做第二层分账
  → 用户后续 withdraw() 时，Hook 在 unlockCallbackWithdraw() 中 burn + take
  → 若未成交，用户可通过 kill() 撤单
```

一句话概括：**V4 Limit Order 不是“价格到了就直接发币给用户”，而是 Hook 借助 `afterSwap`、`unlock` 和 ERC-6909，把“穿越后移仓”和“用户后续提现”拆成了两步。**
