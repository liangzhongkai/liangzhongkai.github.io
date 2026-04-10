---
date: "2026-04-10T10:00:00+08:00"
draft: false
title: "Uniswap V4 Custom Accounting 完全深度解析"
tags: ["web3"]
summary: "基于 v4-core 源码，系统拆解 Flash Accounting、BalanceDelta 与 BeforeSwapDelta 的符号方向，以及 swap、hook 与 settle/take/clear 的记账与结算逻辑。"
---
 
# Uniswap V4 Custom Accounting 完全深度解析

> 先建立正确的 `delta` 心智模型，再去看 `swap`、`modifyLiquidity`、`hook` 和结算原语，读源码时就不会被符号和调用顺序绕晕。

---

## 1. 先记住 4 个结论

如果你只想先抓住主线，先记这 4 句：

1. `delta > 0` 不是“我欠 PoolManager”，而是 **PoolManager 欠我**；`delta < 0` 才是 **我欠 PoolManager**。
2. `settle()` 会把某个账户的 `delta` **往正方向推**，`take()` / `mint()` / `clear()` 会把它 **往负方向推**。
3. `beforeSwap` 返回的不是最终会被记账的 `BalanceDelta`，它先返回 `BeforeSwapDelta`，真正给 hook 和记账对象入账的是 `afterSwap` 合并后的结果。
4. `nonzeroDeltaCount` 统计的不是“有几个地址没结清”，而是 **有多少个 `(account, currency)` transient slot 仍然非零**。

这 4 句吃透之后，Uniswap v4 的 Custom Accounting 基本就不会再读反。

---

## 2. Flash Accounting 到底在做什么

Uniswap v4 的核心变化不是“加了 hook”这么简单，而是把“做 AMM 计算”和“做真实转账”拆开了。

在 `unlock()` 打开的窗口里：

- PoolManager 允许调用者连续做多步操作
- 中间步骤大多只是修改 transient storage 里的 `delta`
- 直到最后，调用者再决定如何把这些 `delta` 结掉

一个最小流程大概是这样：

```text
PoolManager.unlock(data)
  -> Lock.unlock()
  -> msg.sender.unlockCallback(data)
       -> swap / modifyLiquidity / donate
       -> sync / settle / take / mint / burn / clear
  -> assert(nonzeroDeltaCount == 0)
  -> Lock.lock()
```

这带来两个非常重要的结果：

- **多步操作可以先净额合并，再做真实转账**
- **hook 可以在 AMM 公式之外，修改一部分输入输出语义**

但要注意，v4 并不是“随便改账”。所有改动最后都必须回到同一个硬约束：

> 在 `unlock()` 结束时，所有未清零的 `(account, currency)` delta 都必须归零，否则整笔交易回滚。

---

## 3. `BalanceDelta`、`BeforeSwapDelta`、`nonzeroDeltaCount`

这一节是整篇文章最重要的基础。

### 3.1 `BalanceDelta` 的正确语义

`BalanceDelta` 定义在 `src/types/BalanceDelta.sol`：

```solidity
type BalanceDelta is int256;
```

高 128 位是 `amount0`，低 128 位是 `amount1`。

**关键不是 packed 编码，而是正负号方向：**

| 值 | 正确含义 |
|---|---|
| `amount0 > 0` | PoolManager 欠调用方 `token0` |
| `amount0 < 0` | 调用方欠 PoolManager `token0` |
| `amount1 > 0` | PoolManager 欠调用方 `token1` |
| `amount1 < 0` | 调用方欠 PoolManager `token1` |

这和很多人第一次直觉相反。

你可以直接用 `take()` 和 `settle()` 去验证这个方向：

- `take(currency, to, amount)` 会先 `_accountDelta(currency, -amount, msg.sender)`，然后转出 token
- `settle()` 会在确认收到 token 后 `_accountDelta(currency, +paid, recipient)`

所以：

- 你要拿钱，`delta` 得先是正的，`take()` 再把它减回 0
- 你要还钱，`delta` 得先是负的，`settle()` 再把它加回 0

### 3.2 `BeforeSwapDelta` 和 `BalanceDelta` 不是一回事

`beforeSwap` 用的不是 token0/token1 语义，而是：

- `specified`
- `unspecified`

定义在 `src/types/BeforeSwapDelta.sol`：

```solidity
type BeforeSwapDelta is int256;
```

高 128 位是 `deltaSpecified`，低 128 位是 `deltaUnspecified`。

这样设计是因为 `beforeSwap` 发生在核心 swap 之前，此时 hook 更容易围绕“用户指定的那一边”和“另一边”做逻辑，而不是自己再手动判断 token0/token1 映射。

### 3.3 `amountSpecified` 的符号也很容易看反

在 `SwapParams` 里：

```solidity
struct SwapParams {
    bool zeroForOne;
    int256 amountSpecified;
    uint160 sqrtPriceLimitX96;
}
```

源码注释写得很明确：

| `amountSpecified` | 语义 |
|---|---|
| `< 0` | exact input |
| `> 0` | exact output |

也就是说：

- `amountSpecified < 0`：你精确指定输入量
- `amountSpecified > 0`：你精确指定输出量

### 3.4 `nonzeroDeltaCount` 在数什么

`_accountDelta()` 的核心逻辑很简单：

```solidity
function _accountDelta(Currency currency, int128 delta, address target) internal {
    if (delta == 0) return;

    (int256 previous, int256 next) = currency.applyDelta(target, delta);

    if (next == 0) {
        NonzeroDeltaCount.decrement();
    } else if (previous == 0) {
        NonzeroDeltaCount.increment();
    }
}
```

这里的计数单位不是“地址”，而是每次 `_accountDelta(currency, ..., target)` 修改的那个 slot。

换句话说：

- 一个地址只欠 `token0`，计数可能是 `1`
- 同一个地址同时对 `token0` 和 `token1` 都有未清余额，计数可能是 `2`
- caller 和 hook 各自都有未清余额，计数会继续累加

因此，**“caller 和 hook 的 delta 总和对冲了”并不代表系统已经结清**。  
只要它们还分别挂在两个不同账户上，`nonzeroDeltaCount` 就不会变成 0。

### 3.5 `RETURNS_DELTA` 权限位的作用

Hooks 库里有 4 个和 delta 返回有关的 flag：

```solidity
uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3;
uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2;
uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1;
uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;
```

只有对应 flag 打开，PoolManager 才会解析并使用 hook 返回的 delta。

这些 flag 不只是 `getHookPermissions()` 里写一下，**还必须和 hook 地址低位编码一致**，否则 `validateHookPermissions()` / `isValidHookAddress()` 会失败。

---

## 4. 结算原语：`sync`、`settle`、`take`、`mint`、`burn`、`clear`

理解这些原语时，不要把它们看成“转账函数”，而要看成：

> “把某个账户在某种 currency 上的未结清 delta 往 0 推回去”的不同手段。

### 4.1 `sync(Currency currency)`

`sync()` 只是给 ERC20 结算做一个余额基准快照。

```solidity
function sync(Currency currency) external
```

对于 ERC20，正确顺序是：

```solidity
poolManager.sync(currency);
token.transfer(address(poolManager), amount);
poolManager.settle();
```

原因是 `settle()` 不接收 currency 参数，它会读取当前已经 sync 的 currency 和 sync 时记录下来的 reserves，计算本次真实付进来了多少。

对 native token，`settle()` 直接看 `msg.value`，不依赖 ERC20 那套余额差分逻辑。

### 4.2 `settle()`

签名是：

```solidity
function settle() external payable returns (uint256 paid)
function settleFor(address recipient) external payable returns (uint256 paid)
```

它的本质是：

- 识别这次实际付进来了多少
- 然后对 `recipient` 做 `_accountDelta(currency, +paid, recipient)`

所以 `settle()` 的效果是：

- 如果当前账户 `delta < 0`，它会把负债往 0 推
- 如果付多了，也会把账户推到正值

`settleFor(recipient)` 非常关键，因为它允许“我来付款，但把这笔 credit 记到别人头上”。

这在 hook 或 router 代付场景里很有用。

### 4.3 `take(Currency currency, address to, uint256 amount)`

签名：

```solidity
function take(Currency currency, address to, uint256 amount) external
```

内部先做：

```solidity
_accountDelta(currency, -(amount.toInt128()), msg.sender);
```

然后再把 token 转出去。

所以 `take()` 的正确理解是：

- 适用于你当前有**正 delta**
- 它把你的 open claim 往 0 减
- 同时真的把 token 打给你

### 4.4 `mint(address to, uint256 id, uint256 amount)`

签名：

```solidity
function mint(address to, uint256 id, uint256 amount) external
```

内部逻辑是：

```solidity
_accountDelta(currency, -(amount.toInt128()), msg.sender);
_mint(to, currency.toId(), amount);
```

也就是说：

- `mint()` 不是“凭空发一个可领取的凭证”
- 它是把 **open delta** 变成 **ERC-6909 claim**

更准确地说，它在做：

> 把你当前可以取走的一部分价值，从 transient 的 open balance，转成持久化的 ERC-6909 claim。

所以它和 `take()` 很像，只不过：

- `take()` 是拿真实 token
- `mint()` 是拿 ERC-6909 claim

### 4.5 `burn(address from, uint256 id, uint256 amount)`

签名：

```solidity
function burn(address from, uint256 id, uint256 amount) external
```

内部逻辑是：

```solidity
_accountDelta(currency, amount.toInt128(), msg.sender);
_burnFrom(from, currency.toId(), amount);
```

它的作用正好和 `mint()` 相反：

- 把 ERC-6909 claim 转回 open delta
- 让调用者的 `delta` 往正方向移动

所以 `burn()` 可以有两类典型用途：

1. 你本来就有 ERC-6909 claim，现在想把它转成 open delta，再 `take()`
2. 你当前是负 delta，手里又有 ERC-6909 claim，可以用 `burn()` 把负债往 0 抵回去

### 4.6 `clear(Currency currency, uint256 amount)`

`clear()` 常被忽略，但在源码里是明确存在的。

它要求：

- 当前 `delta` 必须是正的
- `amount` 必须和当前正 delta 完全相等

然后它会把这笔 open claim 直接清掉，不做任何真实转账。

这通常只适合清除 dust。

因为一旦 `clear`，对应价值就永久留在 PoolManager 里，拿不回来了。

### 4.7 一张表记住这些原语

| 原语 | 对调用者 delta 的影响 | 典型用途 |
|---|---|---|
| `sync(currency)` | 不改 delta，只记录 ERC20 基线 | 为后续 `settle()` 做准备 |
| `settle()` | `+paid` | 还款，或把付款记到某账户 |
| `settleFor(recipient)` | 对 `recipient` 做 `+paid` | 代别人付款 |
| `take(currency, to, amount)` | `-amount` | 把正 delta 提现成真实 token |
| `mint(to, id, amount)` | `-amount` | 把正 delta 转成 ERC-6909 claim |
| `burn(from, id, amount)` | `+amount` | 把 ERC-6909 claim 转回 open delta |
| `clear(currency, amount)` | `-amount` | 放弃一笔正向 dust claim |

---

## 5. 源码视角看 `swap`

这是整套体系里最绕的一条路径，也是最值得慢慢拆开的地方。

### 5.1 `beforeSwap` 不会立刻给 hook 记账

`PoolManager.swap()` 的主干是：

```solidity
(amountToSwap, beforeSwapDelta, lpFeeOverride) = key.hooks.beforeSwap(key, params, hookData);
swapDelta = _swap(... amountToSwap ...);
(swapDelta, hookDelta) = key.hooks.afterSwap(key, params, swapDelta, hookData, beforeSwapDelta);

if (hookDelta != ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));
_accountPoolBalanceDelta(key, swapDelta, msg.sender);
```

这个顺序非常关键：

1. `beforeSwap` 先返回 `BeforeSwapDelta`
2. 这个返回值会影响 `amountToSwap`
3. 核心 AMM 只处理“剩余的 amount”
4. `afterSwap` 再把前后两个阶段的信息合并成最终 `hookDelta`
5. 最后才分别给 hook 和 caller 入账

所以，**`beforeSwap` 返回 delta != “hook 立刻被记账”**。

### 5.2 `beforeSwap` 到底改了什么

Hooks 库里的 `beforeSwap()` 会先拿到：

```solidity
amountToSwap = params.amountSpecified;
```

如果 hook 开了 `BEFORE_SWAP_RETURNS_DELTA_FLAG`，它会取出：

```solidity
int128 hookDeltaSpecified = hookReturn.getSpecifiedDelta();
amountToSwap += hookDeltaSpecified;
```

然后还要检查一件很重要的事：

- 原来是 exact input，不能被你改成 exact output
- 原来是 exact output，不能被你改成 exact input

如果翻过符号边界，会直接触发 `HookDeltaExceedsSwapAmount()`。

所以 `beforeSwap` 的本质不是“直接改余额”，而是：

> 先从本次 swap 里切走一部分 specified side，让核心 AMM 只处理剩下的那部分。

### 5.3 `afterSwap` 才把 hook 的改动落成最终 `hookDelta`

`afterSwap()` 会把三部分信息合并起来：

1. `beforeSwap` 给的 `specified delta`
2. `beforeSwap` 给的 `unspecified delta`
3. `afterSwap` 返回的 `unspecified delta`

然后把它们映射回 token0/token1 语义，得到最终 `hookDelta`。

最后：

```solidity
swapDelta = swapDelta - hookDelta;
```

这意味着：

- `hookDelta` 记到 hook 地址上
- `swapDelta` 会被改写成“caller 在 hook 介入后真正应该看到的最终结果”

### 5.4 为什么 `afterSwap` 只返回 `int128`

`IHooks.afterSwap()` 的返回值不是 `BalanceDelta`，而是：

```solidity
returns (bytes4, int128)
```

这个 `int128` 只代表 **unspecified currency** 的额外调整。

原因是 specified side 已经在 `beforeSwapDelta` 里表达过了，`afterSwap` 只补那部分需要在核心 swap 之后才能知道的变化。

### 5.5 `swapDelta` 应该怎么解读

`PoolManager.swap()` 返回的 `swapDelta` 是**caller 视角的最终 delta**。

所以一个 router 写结算逻辑时，正确判断方式是：

```solidity
if (delta.amount0() < 0) {
    // caller 欠 PoolManager token0，应该去付钱
}
if (delta.amount0() > 0) {
    // PoolManager 欠 caller token0，应该 take 或 mint
}
```

这个方向千万不要写反。

### 5.6 一个简化版结算框架

```solidity
BalanceDelta delta = poolManager.swap(key, params, hookData);

if (delta.amount0() < 0) {
    poolManager.sync(key.currency0);
    token0.transfer(address(poolManager), uint256(uint128(-delta.amount0())));
    poolManager.settle();
} else if (delta.amount0() > 0) {
    poolManager.take(key.currency0, recipient, uint256(uint128(delta.amount0())));
}

if (delta.amount1() < 0) {
    poolManager.sync(key.currency1);
    token1.transfer(address(poolManager), uint256(uint128(-delta.amount1())));
    poolManager.settle();
} else if (delta.amount1() > 0) {
    poolManager.take(key.currency1, recipient, uint256(uint128(delta.amount1())));
}
```

上面这段不是完整生产代码，但足够说明最关键的一点：

> **负数去付款，正数去取款。**

---

## 6. 源码视角看 `modifyLiquidity` 和 `donate`

### 6.1 `modifyLiquidity` 返回两个值，但真正记账的是处理后的 `callerDelta`

`PoolManager.modifyLiquidity()` 的核心顺序是：

```solidity
(principalDelta, feesAccrued) = pool.modifyLiquidity(...);
callerDelta = principalDelta + feesAccrued;
(callerDelta, hookDelta) = key.hooks.afterModifyLiquidity(key, params, callerDelta, feesAccrued, hookData);

if (hookDelta != ZERO_DELTA) _accountPoolBalanceDelta(key, hookDelta, address(key.hooks));
_accountPoolBalanceDelta(key, callerDelta, msg.sender);
```

这里最容易误读的点有两个：

1. `feesAccrued` 只是单独返回给你参考，不会作为一笔独立 delta 再单独入账
2. 如果 after hook 返回了 `hookDelta`，Hooks 库里已经先做了：

```solidity
callerDelta = callerDelta - hookDelta;
```

也就是说，PoolManager 最后记到 caller 身上的，是**已经扣过 hook 调整后的最终 callerDelta**。

### 6.2 如何理解 add/remove liquidity 的 delta

直觉上很好记：

- **加流动性**：你通常要付钱进去，所以 caller 很容易出现负 delta
- **减流动性 / 收手续费**：你通常会拿钱出来，所以 caller 很容易出现正 delta

当然，精确数值取决于 position 当前状态、手续费和 hook 逻辑，但“负数像负债，正数像可领取余额”的总方向不变。

### 6.3 `donate` 是最干净的一条路径

`donate()` 的流程很短：

```solidity
key.hooks.beforeDonate(...);
delta = pool.donate(amount0, amount1);
_accountPoolBalanceDelta(key, delta, msg.sender);
key.hooks.afterDonate(...);
```

这里没有 donate 对应的 returns-delta 权限位。

所以 donate hook 能做的是：

- 前后做逻辑
- 记录状态
- 更新外部信息

但它**不能像 swap / modifyLiquidity 那样直接改本次 donor 的结算数额**。

---

## 7. Hook 为什么像一个“独立账户”

一旦 hook 返回了最终 `hookDelta`，PoolManager 会做：

```solidity
_accountPoolBalanceDelta(key, hookDelta, address(key.hooks));
```

从这一刻开始，hook 在 accounting 体系里和普通用户没区别。

它有自己的：

- `account`
- `currency`
- transient delta slot

也要在 `unlock()` 结束前自己把账结平。

### 7.1 最容易误会的一点：caller 和 hook 互相“对冲”不等于结清

假设：

- caller 在 `token0` 上是 `-100`
- hook 在 `token0` 上是 `+100`

从“系统总和”看像是对冲了，但从 PoolManager 的账本看，它们仍然是两个独立 slot：

```text
(caller, token0) = -100
(hook,   token0) = +100
```

这两个 slot 都非零，`nonzeroDeltaCount` 仍然不是 0。

所以必须继续处理：

- caller 去 `settle()` 把 `-100` 补回来
- hook 去 `take()` / `mint()` / `clear()` 把 `+100` 消掉

或者：

- 另一个人代 hook 付款，用 `settleFor(address(hook))`
- hook 用自己已有的 ERC-6909 claim 做 `burn()`

### 7.2 hook 清账的几种常见方式

**方式 A：hook 自己付款**

hook 或 router 先把 token 打给 PoolManager，再调用：

```solidity
poolManager.settleFor(address(hook));
```

这适合 hook 当前是负 delta 的情况。

**方式 B：hook 手里已经有 ERC-6909 claim**

如果 hook 持有对应 currency 的 ERC-6909 claim，可以：

```solidity
poolManager.burn(address(hook), currency.toId(), amount);
```

这样会把 hook 的 delta 往正方向拉回去，用来抵它的负债。

**方式 C：hook 当前是正 delta**

这时 hook 可以：

- `take()` 领走真实 token
- `mint()` 转成 ERC-6909 claim
- `clear()` 放弃 dust

### 7.3 一个更准确的生命周期图

```text
unlockCallback 开始
  -> swap / modifyLiquidity
  -> 生成 callerDelta
  -> 如 hook 返回 delta，再生成 hookDelta
  -> 两个账户分别被记账
  -> caller 处理自己的 delta
  -> hook 也必须处理自己的 delta
unlock() 结束前
  -> 所有 (account, currency) slot 必须回到 0
否则
  -> CurrencyNotSettled()
```

---

## 8. 四种典型模式

这里不再追求“花哨示例越多越好”，而是只保留最值得掌握、且和源码行为一致的四种模式。

### 8.1 Dynamic Fee：最轻量的 hook

这是最适合入门的模式，因为它不碰 delta。

特征：

- 用 `beforeSwap`
- pool 必须是动态费率池，`fee == LPFeeLibrary.DYNAMIC_FEE_FLAG`
- hook 可以在 `beforeSwap` 里更新或返回 override fee
- 不需要 `beforeSwapReturnDelta`

它的价值在于：

- 动态调整 LP fee
- 不引入额外结算复杂度
- 不会给 hook 自己制造未清 delta

如果你是第一次写 v4 hook，建议先从这类模式起步。

最小示例：

```solidity
contract DynamicFeeHook is BaseHook {
    function getHookPermissions() public pure override returns (Hooks.Permissions memory p) {
        p.beforeSwap = true;
    }

    function beforeSwap(
        address,
        PoolKey calldata key,
        SwapParams calldata params,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        uint24 fee = _computeFee(key, params);

        // 只有动态费率池才会使用 override 值
        uint24 feeOverride = fee | LPFeeLibrary.OVERRIDE_FEE_FLAG;

        return (IHooks.beforeSwap.selector, BeforeSwapDeltaLibrary.ZERO_DELTA, feeOverride);
    }
}
```

这个例子的重点是：

- `beforeSwap` 可以参与定价参数，但不碰资金流
- 返回 `ZERO_DELTA`，因此不会给 hook 自己留下未清账目
- 适合先熟悉 hook 权限位、调用时机和 fee 更新路径

### 8.2 Liquidity Rebate：after hook 给 LP 返利

这类模式常用：

- `afterAddLiquidity`
- `afterAddLiquidityReturnDelta`

思路是：

- LP 正常加流动性后，会得到一个 callerDelta
- hook 返回一个对 LP 更友好的 `hookDelta`
- Hooks 库把最终 callerDelta 调整成 `callerDelta - hookDelta`

但真正难的不是“返多少”，而是：

> hook 自己那边的 delta 准备怎么清掉？

这也是为什么很多 rebate 设计最后都需要搭配：

- `settleFor(address(hook))`
- 或 hook 自持 ERC-6909 claim，再 `burn()`

最小示例：

```solidity
contract RebateHook is BaseHook {
    using BalanceDeltaLibrary for BalanceDelta;

    function getHookPermissions() public pure override returns (Hooks.Permissions memory p) {
        p.afterAddLiquidity = true;
        p.afterAddLiquidityReturnDelta = true;
    }

    function afterAddLiquidity(
        address,
        PoolKey calldata,
        ModifyLiquidityParams calldata,
        BalanceDelta delta,
        BalanceDelta,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BalanceDelta) {
        // caller 加流动性时通常会产生负 delta；这里返还 10%
        int128 rebate0 = delta.amount0() < 0 ? delta.amount0() / 10 : 0;
        int128 rebate1 = delta.amount1() < 0 ? delta.amount1() / 10 : 0;

        BalanceDelta hookDelta = toBalanceDelta(rebate0, rebate1);
        return (IHooks.afterAddLiquidity.selector, hookDelta);
    }
}
```

这个例子表达的是：

- hook 返回负的 `hookDelta`，表示 hook 把一部分价值让给 LP
- Hooks 库会把 caller 的最终 `callerDelta` 改成 `callerDelta - hookDelta`
- 但 hook 自己也因此得到一笔独立 delta，后续必须额外设计清账路径

### 8.3 Custom Curves：让 AMM 只做剩余部分

这类模式通常用：

- `beforeSwap`
- `beforeSwapReturnDelta`

它的正确理解不是“hook 一开，内置 AMM 就彻底失效”，而是：

- hook 可以先拿走一部分 specified side
- 核心 AMM 再处理剩余 amount
- 如果 hook 恰好把 `amountToSwap` 调成 0，那这次 swap 才等于完全由 hook 接管

所以 Custom Curves 的本质更像：

> hook 先在 swap 前切走一段订单流，剩下的再交给内置 AMM。

这类模式最容易犯的错有两个：

1. 把 exact input / exact output 的符号判断写反
2. 只改了 caller 的感知结果，没有准备 hook 自己的清账路径

最小示例：

```solidity
contract CustomCurveHook is BaseHook {
    function getHookPermissions() public pure override returns (Hooks.Permissions memory p) {
        p.beforeSwap = true;
        p.beforeSwapReturnDelta = true;
    }

    function beforeSwap(
        address,
        PoolKey calldata,
        SwapParams calldata params,
        bytes calldata
    ) external override onlyPoolManager returns (bytes4, BeforeSwapDelta, uint24) {
        bool exactInput = params.amountSpecified < 0;

        // 例子：hook 吃掉一半 specified side，让 AMM 只处理剩余一半
        // exactInput 时 amountSpecified 为负，因此这里要返回正值来“减小绝对值”
        int128 specifiedSlice = exactInput
            ? int128(uint256(-params.amountSpecified) / 2)
            : -int128(uint256(params.amountSpecified) / 2);

        int128 unspecifiedSlice = _quoteOtherSide(params, specifiedSlice);

        BeforeSwapDelta delta = toBeforeSwapDelta(specifiedSlice, unspecifiedSlice);
        return (IHooks.beforeSwap.selector, delta, 0);
    }
}
```

这里最值得读者注意的是：

- exact input 在源码里是 `amountSpecified < 0`
- `beforeSwapDelta` 改的是“本次要送进核心 AMM 的量”，不是直接给 hook 记账
- 真正落成 `hookDelta` 并入账，要等 `afterSwap` 合并阶段完成

### 8.4 Limit Order / Receipt 模式：把 open delta 和持久化 claim 分离

很多 limit order、延迟结算、可转让 claim 设计，本质都建立在这一点上：

- open delta 是交易内、短期、必须在本次 unlock 里结掉的
- ERC-6909 claim 是跨交易、持久化、可后续再处理的

所以一个非常常见的设计套路是：

1. 本次交易内先把应得价值变成 open delta
2. 不立刻 `take()`，而是 `mint()` 成 ERC-6909 claim
3. 以后再 `burn()` 回 open delta，或者在新的 unlock 中组合更多操作

这类模式的重点不是某个具体 demo，而是理解：

> `mint` / `burn` 在 v4 里承担的是“open claim 和持久化 claim 之间的桥”。

最小示例：

```solidity
contract ReceiptRouter is IUnlockCallback {
    function unlockCallback(bytes calldata data) external returns (bytes memory) {
        (PoolKey memory key, SwapParams memory params, address user) =
            abi.decode(data, (PoolKey, SwapParams, address));

        BalanceDelta delta = poolManager.swap(key, params, "");

        // 把本次交易中用户应得的 token1 先转成 ERC-6909 claim
        if (delta.amount1() > 0) {
            poolManager.mint(user, uint256(uint160(Currency.unwrap(key.currency1))), uint128(delta.amount1()));
        }

        // 如果用户欠 token0，则本次交易内补齐
        if (delta.amount0() < 0) {
            poolManager.sync(key.currency0);
            IERC20(Currency.unwrap(key.currency0)).transferFrom(
                user,
                address(poolManager),
                uint256(uint128(-delta.amount0()))
            );
            poolManager.settle();
        }

        return "";
    }
}
```

以及后续某次交易里，用户可以再把 claim 转回 open delta：

```solidity
poolManager.burn(user, uint256(uint160(Currency.unwrap(currency1))), amount);
poolManager.take(currency1, user, amount);
```

这组例子想说明的是：

- `mint()` 不是白送凭证，而是在消耗当前正 delta
- `burn()` 会把 claim 再转回 open delta
- `mint/burn` 让“交易内结算”和“跨交易持有权”可以拆开设计

---

## 9. 最容易写错的 10 个点

### 1. 把 `BalanceDelta` 正负号整体看反

最常见，也最致命。

记忆方法：

- `delta > 0`：PoolManager 欠我
- `delta < 0`：我欠 PoolManager

### 2. 把 `amountSpecified` 的 exactIn / exactOut 看反

源码是：

- `< 0` = exact input
- `> 0` = exact output

### 3. 把 `settle()` 写成 `settle(currency)`

源码没有这个签名。

正确流程是：

```solidity
poolManager.sync(currency);
token.transfer(address(poolManager), amount);
poolManager.settle();
```

或者直接用 native token 的 `msg.value`。

### 4. 以为 `mint()` 是“无中生有发凭证”

不是。

`mint()` 是把 open delta 变成 ERC-6909 claim，它本质上和 `take()` 一样，都是在消耗你当前的正向 claim。

### 5. 以为 `burn()` 只能“销毁凭证，不改账”

也不是。

`burn()` 会把 ERC-6909 claim 转回 open delta，并给调用者做 `+amount`。

### 6. 以为 `beforeSwap` 返回后 hook 已经立刻被记账

真正记账发生在 `afterSwap` 合并出最终 `hookDelta` 之后。

### 7. 以为 caller 和 hook 在总量上对冲了，就等于系统结平

错误。

`nonzeroDeltaCount` 看的是 slot，不是总和。

### 8. 忽略 `settleFor(recipient)`

很多人只盯着 `settle()`，结果发现“我能付款，但这笔 credit 记不到想记的地址上”。

这时通常就该想到 `settleFor`。

### 9. 忘记 `sync()` 只对 ERC20 路径重要

native token 的 `settle()` 看的是 `msg.value`。  
ERC20 才需要 `sync -> transfer -> settle`。

### 10. 只设计了 caller 的结算路径，忘了 hook 自己也会留下 delta

每次 hook 返回非零 delta，都该追问一句：

> hook 账户最后怎么归零？

### 审计时可以直接逐项核对

```text
□ `delta > 0 / < 0` 的语义是否全部写对？
□ `amountSpecified` 的正负号是否全部写对？
□ 是否把 `settle()` / `settleFor()` 的签名和时序写对？
□ 是否正确区分了 open delta 和 ERC-6909 claim？
□ hook 返回 delta 时，权限位和 hook 地址编码是否一致？
□ `beforeSwap` / `afterSwap` 的合并路径是否按源码理解？
□ `modifyLiquidity` 是否把 feesAccrued 当成“返回给你看的信息量”而不是二次入账量？
□ 是否考虑了 hook 自己的清账路径？
□ 是否知道 `nonzeroDeltaCount` 统计的是 `(account, currency)` slot？
□ 所有路径结束时，是否确保所有未清 slot 回到 0？
```

---

## 10. 一页纸心智模型

如果把整篇文章压缩成一张图，大概就是下面这样：

```text
1. unlock() 打开一个临时账本
2. swap / modifyLiquidity / donate 往账本里写 delta
3. hook 可以在权限允许的前提下修改部分 delta
4. delta 始终按“账户 + 币种”分开记
5. 负 delta 需要付钱，用 settle / settleFor / burn 去补
6. 正 delta 代表可领取，用 take / mint / clear 去消
7. unlock() 结束前，所有 slot 必须归零
```

再压缩成一句话就是：

> **Uniswap v4 的 Custom Accounting 不是“hook 任意改余额”，而是“在 `unlock()` 这段时间里，PoolManager 允许你和 hook 一起修改一组临时 delta，然后要求你们在窗口结束前把每一个 `(account, currency)` 账目都结平”。**

---

## 参考源码

- `src/PoolManager.sol`
- `src/libraries/Hooks.sol`
- `src/libraries/CurrencyDelta.sol`
- `src/libraries/CurrencyReserves.sol`
- `src/libraries/LPFeeLibrary.sol`
- `src/types/BalanceDelta.sol`
- `src/types/BeforeSwapDelta.sol`
- `src/types/PoolOperation.sol`
- `src/interfaces/IPoolManager.sol`
- `src/interfaces/IHooks.sol`
