+++
date = '2026-06-05T10:00:00+08:00'
draft = false
title = 'Zero Knowledge Puzzles 学习总结'
tags = ['web3', 'zk', 'circom']
description = 'RareSkills zero-knowledge-puzzles 练习笔记：Circom 语法、R1CS 约束模式、snarkjs 工具链与 Whitelist/Vote/Claim 全链路场景。'
+++

# Zero Knowledge Puzzles 学习总结

[zero-knowledge-puzzles](https://github.com/liangzhongkai/zero-knowledge-puzzles) 练习笔记（fork 自 RareSkills）。栈：Circom 2.x、circomlib、circom_tester、snarkjs、Hardhat、Groth16。

---

## 一、项目在练什么

把「某陈述为真」写成 BN254 标量域上的**二次约束（R1CS）**，再跑 witness → 证明 →（可选）链上验证。

四条线，顺序基本对应 README：

1. 信号与约束语法（`===`、`<==`、`<--`）
2. 电路内可实现的运算（比较、位约束、Poseidon/MiMC）
3. 模板与循环组合（Sudoku、Sujiko）
4. snarkjs 出 proof + Solidity verifier（Compile）

---

## 二、题目怎么排

**主线**（README 顺序）：Addition → Equality/NotEqual → Poseidon → ForLoop → Power → Range → Salt → QuadraticEquation → Compile → Sudoku → Sujiko。

**补充题**（多为 `pragma 2.1.8`；不少是纯约束、无 `signal output`，也有带 output 的如 MultiAND/OR、IntSqrtOut）：Summation、MultiAND/OR/NoOut、HasAtLeastOne、IntSqrt/IntDiv 及 Out 变体、FourBitBinary、AllBinary、IsSorted、IsTribonacci、IncreasingDistance 等。主线跑通后再做，专门练位宽和「只约束、不输出」。

| 块 | 题 | 练到的点 |
|----|-----|----------|
| 语法 | Addition, MultiplyNoOut, BinaryXY | 乘加约束、二进制 `x(x-1)=0` |
| 布尔 | Equality, NotEqual, MultiAND, MultiOR | `IsEqual`、积链、或门公式 |
| 结构 | ForLoop, Power, Summation | 编译期循环、非二次运算绕行 |
| 比较 | Range, IsSorted, IntSqrt, IntDiv | `LessThan(n)`、位宽、除法余数 |
| 哈希 | Poseidon, Salt | 库模板、salt 抬枚举成本 |
| 综合 | Sudoku, Sujiko, Compile | 子电路、公开输入、链上 verify |

[Sujiko/Frontend](https://github.com/liangzhongkai/zero-knowledge-puzzles/tree/main/Sujiko/Frontend)：同一套电路接到浏览器，属于集成而非新语法。

---

## 三、写电路前先固定的三条规则

**1. 电路验的是关系，不是执行顺序**  
Prover 提供 witness；约束只判断赋值是否同时成立。错解 → `calculateWitness` 失败或约束不满足。

**2. 三种赋值语义不同**

| 写法 | 含义 |
|------|------|
| `===` | 纯约束；两侧为 signal 或其线性组合（常量可出现在一侧） |
| `<==` | 定义 witness 并生成约束 |
| `<--` | 仅 witness 赋值，**默认不约束正确性** |

IntDivOut、IntSqrtOut 的考点：`<--` 算商/根，再用乘法和 `LessThan` 钉死，否则 prover 可乱填。

**3. 约束必须二次**  
`/``%`、依赖 witness 的分支、`a**b`（`b` 为 signal）不能直接进 R1CS。整数除用 `\` 只在 `<--` 路径；指数用预计算表 + `IsEqual` 选择（Power）。

---

## 四、五种重复出现的写法

### 1. 直接算术

`in[0] === in[1] + in[2]`，`c <== a * b`。入门题只练语法。

### 2. 布尔进域

- 限制 0/1：`in[i] * (in[i] - 1) === 0`
- AND：`acc[i+1] <== acc[i] * in[i]`
- OR：`acc[i+1] <== acc[i] + in[i] - acc[i]*in[i]`
- NOT：`c <== 1 - eq.out`（`IsEqual` 接好输入后取反，见 NotEqual）
- 多条件与：`out <== a.out * b.out`（Range）

未做二进制约束时，`out` 可在整个域上取值，乘法约束仍可能成立——MultiAND/OR 必做第 1 步。

### 3. 比较

`LessThan(n)` / `GreaterEqThan(n)`：`n` 要盖住操作数上界（Range 32，IntSqrt 32，IntSqrtOut / IntDiv 252）。比较的是**整数**，不是模运算下的序；IntSqrt 用

`b² ≤ a` 且 `a < (b+1)²`

两次 `LessThan`，`lower.out * upper.out === 1`。

### 4. 先算后证（非二次）

| 目标 | 做法 |
|------|------|
| 整除 | `q <-- n \ d`，`r <-- n - q*d`，`n === q*d + r`，`r < d` |
| `a^b`，b 小整数 | `powers[i]` 循环乘，`IsEqual(b,i)` 掩码求和 |
| 开方 | `out <-- floorSqrt(a)`，再接上一节不等式 |

### 5. 拆子电路再乘起来

- Sudoku：`CheckFourCells` 验 4 元排列；行、列、盒各 4 组；`checks` 积链得 `out`；`question` 与 `solution` 用 `assert` 对齐已知格与取值范围。
- Sujiko：四角和用 `===`；1–9 各出现一次 = 9 组 `IsEqual`，每组 9 个 `out` 之和 `=== 1`。

排列（4 元）：对每个 `d∈{1..4}`，四个格等于 `d` 的次数之和 `=== 1`。比排序省约束。

---

## 五、四道题串起来看

**Power**  
`b` 是 witness，不能分支。对 `b∈[0,10]` 各建 `IsEqual(b,i)*a^i`，累加。离散小域上的 if/lookup 都可改成：相等掩码 × 预计算项。

**Salt**  
`MiMCSponge(2, 220, 1)`：`a`、`b` 为输入，`salt` 作 `k`（key）。公开 hash 后枚举 `(a,b)` 不够，必须猜 salt。和 Poseidon 的差别主要在约束成本和项目惯例，不是「谁更安全」一句话能说完。

**Sudoku**  
证的是「存在 solution 满足规则且与 question 一致」，不是电路内搜索。约束数随规模涨——后面优化才有意义。

**Compile**  
`circom --r1cs --wasm` → witness → `groth16 setup/prove` → `verifier.sol`；Hardhat `verifyProof`。`public [a]` 决定链上可见输入；本地 `pot12` 仅学习用。

---

## 六、工具链（三条路线）

> 下面三条是**工程流水线**，与第一节的四条**学习线**不同。

<img src="/images/zk/three-zk-routes.png" alt="Circom + SnarkJS 三条 ZK 路线：JS 测试电路、JS 生成 witness 并验证、全链路链上验证" loading="lazy" width="1200" height="675" />

**JS 测试电路**：`circom` 编译 → `circom_tester` → `loadConstraints` / `calculateWitness`，日常 `yarn test ./test/X.js` 走这条，快。普通 puzzle 的 witness 在测试里手写 input 即可。

**JS 生成 witness 并验证**（Whitelist / Vote / Claim 场景）：`circomlibjs` / `merkle.js` / `proofInputs.js` → `input.json` → `generate_witness.js` → `witness.wtns` → `snarkjs groth16 prove` → `snarkjs groth16 verify`（`vk.json`）。

**全链路**：`snarkjs zkey export solidityverifier` 导出 `Groth16Verifier`，业务合约调 `verifyProof(proofA, proofB, proofC, pubSignals)`。

中间共用：`circom` → R1CS → `groth16 setup`（PTAU）→ `zkey contribute` → `0001.zkey` + `vk.json`。场景脚本细节见 `scripts/compile-scenario.sh`（本地还会跑 pot14 ceremony，setup 用下载的 `powersOfTau28_hez_final_12.ptau`）。全量 test 编译慢，可单题或 Docker。

---

## 七、易错

1. `/` 模除，`\` 整数除；后者只能配合 `<--`。
2. `LessThan` 位宽不足 → 比较错，难查。
3. 只有 `<--` 没有后续 `===` / `<==` → under-constrained。
4. 逻辑输出未限制在 {0,1}。
5. JS 大整数用字符串；域上 -1 写作 `p-1`。

---

## 八、做完之后补什么

**理论**：有限域与曲线；R1CS/QAP；Groth16 vs PLONK/STARK；soundness / ZK 定义。

**电路**：约束计数与 `--O2`；何时不用 SHA256；lookup；递归/聚合。

**应用**：Merkle inclusion、commitment + nullifier、range proof（Range 题的直接延伸）、public input 与 gas。

**工程**：生产 PTAU/ceremony；`--sym` 调试；Noir/Halo2 等对照读一份同类电路。

**安全**：under-constraint、SRS 信任、public signal 顺序与重复花费类漏洞。

---

## 九、案例场景

| 案例 | 目录 | 公开输入 | 生产要点 |
|------|------|----------|----------|
| 白名单 | `Whitelist/` | `root` | Merkle 成员，不暴露叶子 |
| 投票 | `Vote/` | `root, pollId, nullifier, voteCommitment` | 选民集 + nullifier 防双投 |
| 领取 | `Claim/` | `root, distributionId, nullifier, amount` | 叶子绑定金额 + 一次性领取 |

共享：`lib/MerkleProof.circom`（Poseidon Merkle）、`lib/merkle.js` / `lib/proofInputs.js`（链下建树与 witness）。

链上：`contracts/ZKWhitelist.sol`、`ZKVote.sol`、`ZKClaim.sol` + `IZKVerifier`；`yarn test ./test/ZKContracts.js` 用 mock verifier 测业务逻辑。

全链路：`node scripts/gen-input.js <scenario>` → `*/script.sh` → 各场景目录生成 `<Scenario>/contracts/verifier.sol`，按需复制到根 `contracts/`（如 `WhitelistVerifier.sol`）。

**注意**：`script.sh` 里本地 PTAU 仅学习用；主网用公开 ceremony 的 ptau/zkey。
