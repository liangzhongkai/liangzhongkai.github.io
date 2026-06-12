+++
date = '2026-06-12T10:00:00+08:00'
draft = false
title = '从 Tornado Cash 理解 zkSNARKs'
tags = ['web3', 'zk', 'privacy']
description = '以 Tornado Cash 为主线：存款/取款流程、Circom 电路约束、Groth16 验证，以及 QAP 与 Trusted Setup 的数学直觉。'
+++

# 从 Tornado Cash 理解 zkSNARKs

> "我可以向你证明我知道一个秘密，而不向你透露任何关于这个秘密的信息。"
> 这听起来像悖论，却是现代密码学的日常。

---

## 一、从一个真实问题出发

2022 年，美国财政部制裁了以太坊上的 **Tornado Cash** 智能合约——理由是它让用户可以切断加密货币的来源链。

区块链本是透明账本，每笔交易公开可查。Tornado Cash 如何实现匿名？

答案是 **zkSNARKs**（零知识简洁非交互式知识论证）。下面按「业务 → 电路 → 数学」逐层拆开。

---

## 二、区块链的透明性

以太坊上的一笔典型流转：

```
Alice (0xAAA...) → Tornado 合约 : 1 ETH
Tornado 合约 → Bob (0xBBB...) : 1 ETH
```

只要知道 Alice 的身份，Bob 收到的 1 ETH 就可被追溯到 Alice——这叫**伪匿名**。

Tornado Cash 的目标是：**切断 Alice 与 Bob 之间的链上关联**，使观察者无法判断 Bob 的取款对应哪笔存款。

---

## 三、Tornado Cash 机制

Tornado 里用到两种「zk 友好」哈希，职责不同：

| 用途 | 算法 | 说明 |
|------|------|------|
| 存款凭据（commitment、nullifierHash） | **Pedersen** | 对 nullifier / secret 做承诺 |
| Merkle 树内部节点 | **MiMC** | 逐层合并子树哈希 |

普通 SHA-256 / keccak256 在 zk 电路里约束太多，代价过高；Pedersen 与 MiMC 是为此设计的替代品。

### 3.1 存款（Alice）

Alice 在本地生成两个 248-bit 随机数（须小于 BN254 标量域，电路里用 `Num2Bits(248)` 约束）：

```
nullifier  // 防双花：同一 note 只能取一次
secret     // 所有权：与 nullifier 一起构成 commitment
```

两者**从不上链**。Alice 计算：

```
commitment    = Pedersen(nullifier ‖ secret)   // 496 bit 输入
nullifierHash = Pedersen(nullifier)           // 取款时公开，此处可先算好存本地
```

然后调用 `deposit(commitment)` 并附带固定面额（如 1 ETH）。合约将 `commitment` 插入深度为 20 的 Merkle 树（最多 2^20 ≈ 104 万笔存款），链上可见的是新 `root`。

Alice 保存 note：`(nullifier, secret, leafIndex)`——类似凭条，可复制给他人代取。

### 3.2 取款（Bob / Alice 换新地址）

目标：证明「我掌握某笔合法存款的 nullifier 与 secret」，但**不暴露是哪一笔**。

流程分两步：**链下生成证明 → 链上验证**。

#### 链下：Prover 的输入

**私有输入**（绝不上链）：

| 变量 | 含义 |
|------|------|
| `nullifier`, `secret` | 存款凭据 |
| `pathElements[20]`, `pathIndices[20]` | Merkle 路径（兄弟节点哈希 + 每层左/右） |

**公开输入**（Verifier 可见）：

| 变量 | 含义 |
|------|------|
| `root` | 当前合法的 Merkle 根 |
| `nullifierHash` | `Pedersen(nullifier)`，防双花 |
| `recipient` | 收款地址 |
| `relayer`, `fee`, `refund` | 中继者与手续费（可为 0） |

Prover 将输入喂入预编译电路，运行 Groth16：

```
proof = Groth16_Prove(provingKey, privateInputs, publicInputs)
```

输出约 **192 字节**（三个椭圆曲线点），大小与约束数量无关。

#### 电路在证明什么？

三组约束须**同时**成立：

**约束一：凭据合法**

```
computed_commitment = Pedersen(nullifier, secret)
computed_nullifierHash = Pedersen(nullifier)

assert computed_nullifierHash == nullifierHash   // 公开输入
```

**约束二：commitment 在 Merkle 树中**

从 `computed_commitment` 出发，沿私有路径逐层用 **MiMC** 向上合并（`pathIndices[i]` 决定左右顺序），共 20 层：

```
layer_0 = computed_commitment
// 每层: layer_{i+1} = MiMC(left, right)，顺序由 pathIndices[i] 决定
computed_root = layer_20

assert computed_root == root   // 公开输入
```

路径是私有输入，Verifier 看不到具体叶子，因此不知道对应哪笔存款。

**约束三：公开参数不可篡改**

`recipient`、`relayer`、`fee`、`refund` 在电路里被平方约束进证明（防止中继者篡改收款地址或手续费）。它们不参与 Merkle 逻辑，但必须与生成证明时一致。

完整 withdraw 电路约 **28,271** 条 R1CS 约束（公式 `1869 + 1325 × tree_depth`，depth = 20）。

#### 链上：Verifier

```solidity
withdraw(proof, root, nullifierHash, recipient, relayer, fee, refund)
```

合约依次检查：

1. **`isKnownRoot(root)`** — 根须在合约记录的**最近 30 个**历史根之中（取款不必用最新根，但也不能太旧）。
2. **`!nullifierHashes[nullifierHash]`** — 该 nullifier 未被使用过。
3. **`verifier.verifyProof(proof, [root, nullifierHash, recipient, relayer, fee, refund])`** — 椭圆曲线配对验证 Groth16 证明；开销**固定**，与 2.8 万条约束的数量无关（整笔 withdraw 约 30 万 gas）。
4. 标记 `nullifierHash` 已用，向 `recipient` 转账 `denomination - fee`，向 `relayer` 付 `fee`。

### 3.3 为什么链上无法关联？

| 信息 | 公开？ | 能否定位存款？ |
|------|--------|----------------|
| `commitment`（存款） | ✅ | ❌ Pedersen 单向，无法反推 nullifier/secret |
| `nullifierHash`（取款） | ✅ | ❌ 无法反推 nullifier，也无法与 commitment 关联 |
| `root` | ✅ | ❌ 只说明「某个合法存款存在」 |
| 路径、nullifier、secret | 🔒 | — |

链上同时出现的 `commitment` 与 `nullifierHash` 分别来自：

```
Pedersen(nullifier, secret)    // 存款
Pedersen(nullifier)            // 取款
```

在不知道 `nullifier` 的前提下，无法判断某对 `(commitment, nullifierHash)` 是否来自同一 note——这是匿名性的密码学基础。

---

## 四、零知识证明的三条性质

| 性质 | 含义 | Tornado Cash 中的体现 |
|------|------|------------------------|
| **完备性** | 陈述为真时，诚实 Prover 必能生成合法证明 | 真存过款的人一定能过验证 |
| **可靠性** | 陈述为假时，伪造证明的概率可忽略 | 没存款的人无法骗过合约 |
| **零知识性** | Verifier 除「陈述为真」外学不到更多信息 | 验证通过仍不知道对应哪笔存款 |

---

## 五、zkSNARKs 如何实现：三层压缩

28,000 条约束如何压成 192 字节？三层技术叠加：

### 5.1 约束 → 多项式（QAP）

每条 R1CS 约束形如 `(a · w) × (b · w) = (c · w)`，`w` 为公开 + 私有 witness 向量。

不逐条检查 28,271 个等式，而是把它们编码进多项式：在点 `x = 1, …, 28271` 上，若所有约束成立，则

```
A(x) · B(x) - C(x) = H(x) · Z(x)，  Z(x) = (x-1)(x-2)…(x-28271)
```

大量离散约束化为**一个代数整除关系**。

### 5.2 随机点求值 + Trusted Setup

要验证上述多项式恒等，只需在秘密随机点 τ 上检查：

```
A(τ)·B(τ) - C(τ) = H(τ)·Z(τ)
```

由 **Schwartz–Zippel 引理**，有限域上随机点匹配则恒等成立（否则只是极低概率巧合）。

Prover 不知道 τ，但 Trusted Setup 预先公开 `[τ⁰]G, [τ¹]G, …`，销毁 τ 后 Prover 仍可用这些点线性组合出 `[A(τ)]G` 等。

### 5.3 配对验证乘法

Prover 提交 `[A(τ)]G₁, [B(τ)]G₂, [C(τ)]G₁, …`。Verifier 需验证隐藏标量之间的乘法关系，直接提取 `A(τ)` 是离散对数难题。

**双线性配对**允许在不解密的情况下验证乘积：

```
e([A(τ)]G₁, [B(τ)]G₂) = e(G₁, G₂)^(A(τ)·B(τ))
```

Groth16 将 Verifier 工作归结为 **3 个配对等式**，电路再大，验证代价也恒定。

---

## 六、Trusted Setup 的代价

一次性生成：

- **Proving Key（pk）**：Prover 生成 proof 用
- **Verification Key（vk）**：链上 Verifier 用

Setup 中的毒废料 τ 若泄露，攻击者可伪造「从未存款却取款」的证明。

**MPC Ceremony** 缓解此风险：N 个参与者依次混入随机数并销毁，最终 τ = r₁·r₂·…·rN，无人掌握完整 τ。**只要有一人诚实销毁，系统即安全。** Tornado Cash 使用了 Zcash / iden3 的 Powers of Tau 成果（200+ 参与者）。

---

## 七、更广的应用

Tornado 是 zkSNARK 最直观的入门案例，同类技术已在：

**zkRollup（如 zkSync Era）** — 链下打包数千笔交易，主链只验证一个 proof，吞吐量大幅提升。结构与 Tornado 类似：Sequencer 是 Prover，L1 合约是 Verifier，交易细节是私有输入，新状态根是公开输入。

**选择性披露身份** — 私有输入为完整证件，公开输入为「年龄 > 18」等命题；证明命题为真，不泄露生日、证件号等。

**可验证 AI 推理** — 证明「模型 M 对输入 x 输出 y」，不公开模型权重。

---

## 八、zkSNARKs vs zkSTARKs

| 特性 | zkSNARKs (Groth16) | zkSTARKs |
|------|---------------------|----------|
| 证明大小 | ~192 B | ~100 KB |
| 验证速度 | 极快（固定配对） | 较快 |
| Trusted Setup | 需要 | **不需要** |
| 量子安全 | 否（椭圆曲线） | **是**（哈希） |
| 代表项目 | Tornado Cash, zkSync | StarkNet, Polygon Miden |

STARK 用哈希 + FRI 替代配对，省去 Setup，代价是更大的证明体积。

---

## 九、总结

```
透明链上存取款可被关联
    → 证明「存款合法」但不指明是哪笔
    → 零知识证明（ZKP）/ zkSNARKs
    → Pedersen 承诺 + MiMC Merkle 路径 + nullifier 防双花
    → QAP 多项式化 → 随机点 → 椭圆曲线隐藏 → 配对验证
    → Verifier O(1) 确认全部约束，对私有输入一无所知
```

零知识证明的核心：

> **证明一件事为真，与解释它为什么为真，可以完全分离。**

在不信任的环境里，这让我们能用数学精确控制「透露什么、保留什么」。

---

*参考资料：Groth16 (2016)；[tornadocash/tornado-core](https://github.com/tornadocash/tornado-core)；Zcash Protocol Specification；Vitalik Buterin ZKP 系列；[Why and How zk-SNARK Works](https://github.com/maksym-petruk/how-to-build-a-snark) (Maksym Petkus, 2019)*
