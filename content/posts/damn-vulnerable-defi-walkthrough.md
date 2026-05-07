+++
date = '2026-05-07T14:00:00+08:00'
draft = false
title = 'Damn Vulnerable DeFi 图解手记：18 关解题地图'
tags = ['web3']
description = '手绘流程图串起 Damn Vulnerable DeFi 各关核心漏洞与攻击路径，便于复习与对照源码。'
+++

# Damn Vulnerable DeFi 图解手记：18 关解题地图

> [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) 是学习智能合约安全的经典靶场。下面这组图把每一关的**合约关系**和**攻击编排**压缩成一张「地图」：看图能回想起「先调用谁、再借什么、最后把资产送到哪里」。

---

<style>
.dvd-wrap { margin: 2rem 0; }
.dvd-wrap figure {
  margin: 0;
  padding: 0;
  border-radius: 12px;
  overflow: hidden;
  background: #f8f9fa;
  border: 1px solid rgba(0,0,0,.06);
  box-shadow: 0 8px 32px rgba(0,0,0,.06);
}
.dvd-wrap img {
  display: block;
  width: 100%;
  height: auto;
  vertical-align: middle;
}
.dvd-wrap figcaption {
  font-size: 0.9rem;
  color: #444;
  padding: 0.85rem 1rem 1rem;
  line-height: 1.55;
  border-top: 1px solid rgba(0,0,0,.06);
  background: #fff;
}
.dvd-wrap .tagline {
  display: block;
  font-weight: 600;
  color: #111;
  margin-bottom: 0.35rem;
}
</style>

## 目录

按本文配图编号（与手绘角落题号一致）：

1. [Unstoppable](#1-unstoppable) — 会计不一致
2. [Naive Receiver](#2-naive-receiver) — 费用与转发器伪造调用者
3. [Truster](#3-truster) — 零额闪电贷 + 任意 `call`
4. [Side Entrance](#4-side-entrance) — 份额与余额脱钩
5. [Puppet](#5-puppet) — Uniswap V1 现货预言机
6. [Compromised](#6-compromised) — 链外报价与信任边界
7. [The Rewarder](#7-the-rewarder) — Merkle 领取状态更新
8. [Selfie](#8-selfie) — 闪电贷伪装治理权重
9. [Puppet V2](#9-puppet-v2) — Uniswap V2 储备价预言机
10. [Free Rider](#10-free-rider) — 市集合约支付逻辑
11. [Backdoor](#11-backdoor) — Safe 初始化 `delegatecall` 与模块
12. [Climber](#12-climber) — Timelock 执行与校验顺序
13. [Puppet V3](#13-puppet-v3) — Uniswap V3 TWAP 延迟
14. [Wallet Mining](#14-wallet-mining) — `CREATE2` 撞地址与授权初始化
15. [ABI Smuggling](#15-abi-smuggling) — 授权解析与执行解析不一致
16. [Shards](#16-shards) — 精度与清算路径
17. [Curvy Puppet](#17-curvy-puppet) — Curve `virtual_price` 与清算
18. [Withdrawal](#18-withdrawal) — L1/L2 Merkle 提款编排

---

## 1. Unstoppable

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/01-unstoppable.png" alt="Unstoppable 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：直接 `transfer` 让「账上代币余额」与「份额会计」错位</span>金库用内部累计与 `ERC20.balanceOf` 做不变量校验；玩家绕过 `deposit` 向合约转代币后，余额变大但份额未增，后续 `flashLoan` 自检失败 — 监控侧 `checkFlashLoan` 也随之报错，构成“不可阻挡”的服务打断。</figcaption>
</figure>
</div>

---

## 2. Naive Receiver

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/02-naive-receiver.png" alt="Naive Receiver 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：对受害者重复「零本金的闪电贷手续费」+ 中继伪造特权 `sender`</span>第一步：以接收者为借款方多次借 0、付固定手续费，把其 WETH 「蹭」进池子。第二步：借助 `BasicForwarder` 在 calldata 尾部拼接权威地址，让池子误判 `msg.sender`，一次提走池内全部 WETH。</figcaption>
</figure>
</div>

---

## 3. Truster

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/03-truster.png" alt="Truster 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：闪电贷金额为 0 仍可触发对任意目标的 `call`</span>池子在放款回调前用低级 `call` 执行攻击者给的 `data`，以池子身份对代币 `approve` 给攻击合约；回调结束后再由攻击方 `transferFrom` 清空池子。</figcaption>
</figure>
</div>

---

## 4. Side Entrance

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/04-side-entrance.png" alt="Side Entrance 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：在同一笔交易里用闪电贷操纵「存款瞬间」的可计量余额</span>经典打法是闪电贷借入池子资产、完成存款取得份额，再通过转账等方式让池子的公允计量偏离预期，从而在 `withdraw` 时多取价值，最后归还闪电贷本息。</figcaption>
</figure>
</div>

---

## 5. Puppet

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/05-puppet.png" alt="Puppet 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：Uniswap V1 现货池作预言机 — 大单砸盘即可压低 DVT 标价</span>攻击合约将大量 DVT 换成 ETH，使池子读取到的「ETH 侧变贵 / DVT 侧贬值」；随后用极少 ETH 抵押借空 `PuppetPool`，把 DVT 送到 `recovery`。</figcaption>
</figure>
</div>

---

## 6. Compromised

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/06-compromised.png" alt="Compromised 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：链外依赖被攻破后，预言机报价与资产定价同步失守</span>本题典型路径是还原被泄露的链下身份，操控依赖该报价的交换，从而以畸低价格拿走高价值资产（具体细节以官方仓库与测试为准）。</figcaption>
</figure>
</div>

---

## 7. The Rewarder

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/07-the-rewarder.png" alt="The Rewarder 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：同一批次连续 `claim` 时，只把「最后一次」记入已领取状态</span>对 DVT、WETH 的 Merkle 证明若可在同一块中重复用尽，攻击者可多轮提取直至接近奖池扣减「合法用户应得」后的剩余。</figcaption>
</figure>
</div>

---

## 8. Selfie

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/08-selfie.png" alt="Selfie 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：治理权重按「当前余额」快照，可被闪电贷瞬间放大</span>从 `SelfiePool` 借出几乎全部治理代币，在回调内向 `SimpleGovernance` `queueAction` 排入对 `emergencyExit` 的调用；还款后等待延时到期再执行，池内 DVV 流入 `recovery`。</figcaption>
</figure>
</div>

---

## 9. Puppet V2

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/09-puppet-v2.png" alt="Puppet V2 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：抵押计算读取 Uniswap V2 `getReserves` 的即期价</span>先把 DVT 通过 Router 换成 WETH 再 `unwrap` 成 ETH，必要时再 `deposit` 回 WETH；储备被大幅扭曲后，用少量 WETH 在 `PuppetV2Pool` 借出全部 DVT。</figcaption>
</figure>
</div>

---

## 10. Free Rider

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/10-free-rider.png" alt="Free Rider 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：市集合约对批量购买的付款与退款逻辑存在可乘之机</span>攻击合约从 Pair 闪电贷 WETH，解包成 ETH 后调用 `buyMany`：在低流动性条件下既取走全部 NFT，又触发异常资金流；随后将 NFT 交给 `FreeRiderRecoveryManager` 领赏，再打包换回 WETH 还贷。</figcaption>
</figure>
</div>

---

## 11. Backdoor

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/11-backdoor.png" alt="Backdoor 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：`Safe.setup` 允许的回调 `delegatecall` 可写入模块列表</span>工厂 `createProxyWithCallback` 初始化时代理对 `SafeSetupHelper` 进行 `delegatecall`，再在代理上下文 `enableModule` 挂上 `BackdoorModule`；注册表按事件向新钱包打币后，模块可 `execTransactionFromModule` 把资产转到 `recovery`。</figcaption>
</figure>
</div>

---

## 12. Climber

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/12-climber.png" alt="Climber 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：Timelock 在一次 `execute` 中先执行外部调用、再校验「已安排」</span>攻击编排一次批量执行：自授予 `PROPOSER`、把 `delay` 改为 0、在执行的尾部「补登记」同一批操作，使校验被绕过；随后以 Timelock 拥有者身份 `upgradeToAndCall` 换实现并在新逻辑里转走 10,000,000 DVT。</figcaption>
</figure>
</div>

---

## 13. Puppet V3

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/13-puppet-v3.png" alt="Puppet V3 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：V3 池的 TWAP 需要时间在观测窗口内「酿成」低价读数</span>先把 ETH 换 WETH，再把玩家 DVT 通过池子砸成 WETH；等待不少于约 70 秒的窗口推进后，TWAP 认为 DVT 近乎归零，即可用微量 WETH 抵押借走百万 DVT。</figcaption>
</figure>
</div>

---

## 14. Wallet Mining

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/14-wallet-mining.png" alt="Wallet Mining 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：`CREATE2` 代理地址与 `initializer + saltNonce` 枚举碰撞</span>配合 `AuthorizerUpgradeable` 初始化阶段的存储布局问题，对 `USER_DEPOSIT_ADDRESS` 放行；再由 `WalletDeployer.drop` 部署与离线签名路径匹配的 Safe，执行转账并取回部署激励。</figcaption>
</figure>
</div>

---

## 15. ABI Smuggling

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/15-abi-smuggling.png" alt="ABI Smuggling 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：授权器在固定偏移读取「看起来像 withdraw 的 selector」</span>而在执行路径里用另一套偏移解析真实 calldata，把被禁止的 `sweepFunds` 与 `IERC20` 参数藏在包体后部 — 一次 `execute` 同时骗过检查并完成特权调用。</figcaption>
</figure>
</div>

---

## 16. Shards

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/16-shards.png" alt="Shards 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：成交/赎回分支上的支付取整误差可被循环放大</span>`paymentToken.transfer` 相关的 `cancel` 与 `_closeOffer` / `fill` 在舍入上不对称，可构造循环套取「免费的」DVT，最终体现为 `missingTokens` 与奖池被吸干。</figcaption>
</figure>
</div>

---

## 17. Curvy Puppet {#17-curvy-puppet}

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/17-curvy-puppet.png" alt="Curvy Puppet 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：`virtual_price` 驱动 LP 抵押估值，可用大额流动性操作瞬间扭曲</span>自财政库取出 WETH/LP，经 Aave 闪电贷放大本金，加减 Curve 流动性并做不平衡撤出，压低抵押端标价后对多名借款人 `liquidate`，再把 DVT 归还财政并偿还闪电贷。</figcaption>
</figure>
</div>

---

## 18. Withdrawal

<div class="dvd-wrap">
<figure>
<img src="/images/dvd/18-withdrawal.png" alt="Withdrawal 解题思路图解" loading="lazy" width="1200" height="675" />
<figcaption><span class="tagline">核心：L1 `finalizeWithdrawal` + Merkle 校验后的消息转发链</span>运营商在 7 天挑战期后按叶子 `keccak256(encode(...))` 证明提款；四笔流水里第三笔数额异常巨大时，需要先把代币路由到 `recovery` 再回 bridges 以满足题目约束 — 整体是跨层消息与会计对齐题。</figcaption>
</figure>
</div>

---

## 小结

这 18 张图共同的「读图方式」可以概括为三句：

- **钱从哪来**：闪电贷、池子余额、还是治理/预言机的瞬时权重？
- **哪条不变量被打破**：会计、权限、时间锁、还是解码 calldata 的方式？
- **最后一跳去哪**：把代币推到题目给定的 `recovery` / `treasury`，还是把某条跨链消息收尾？

如果你在对照官方 `test/` 与 `src/` 时更新了某关的标准解法，可以把新注释直接补在手绘图上 — 这份文档里的文字只起「索引」作用，**以仓库源码为准**。
