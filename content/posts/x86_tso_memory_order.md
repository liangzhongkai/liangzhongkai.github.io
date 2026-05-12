+++
date = '2026-05-12T10:00:00+08:00'
draft = false
title = 'x86 TSO 下 C++ Memory Order'
tags = ['cpp']
+++

# x86 TSO 下 C++ Memory Order

> 全文按两条线展开：**C++ 内存模型**告诉我们「要证明什么成立」；**x86 的内存次序契约**（多用 TSO 来教）帮助估计「落到这门 ISA 上大概会长成什么指令」。Store Buffer、MESI 等只在后文用作直觉；真的拍板时，仍以 **SDM / ISA 手册与语言标准里的可观察语义** 为准。文末用 HFT、Web3 里的 lock-free / wait-free 片段做侧写，代码多为示意。

---

## 1. x86 的内存模型：TSO

x86 常放在 **TSO（Total Store Order）** 这一族里来讲：软件能指望的「全局先后」比 ARMv8、POWER 一类弱模型紧得多，但又**达不到**教科书式的顺序一致性（SC）。SC 要求所有核仿佛共享同一串读写次序；TSO 则仍允许 **下文第 3 节** 要展开的那种 **Store→Load** 隙缝——所以归根结底仍是弱序，只是 **「弱序里偏强的一档」**。

相对 SC，程序员在 x86 上最需要盯紧的就一件事：**同一线程里，本核较晚发起的 Load 的结果已对自身可见、也可被别的核观察到，但本核较早发起的 Store 却可能还没对其他核生效**——口语里常说 Store Buffer 或 *Store→Load* 重排。换个角度：外部观察者可能"先看到你后发的读的效果，再看到你先发的写"。另一方面，下面这些是在 **架构内存模型** 里写明、而不是某代流水线「碰巧如此」的约束：**较新 Load 不能越过较旧 Load；较新 Store 不能越过较旧 Store；较新 Store 也不能越过较旧 Load**。若要抠定义，请直接查 Intel/AMD SDM 第 8 章及你所用工具链对原子 intrinsic 的说明。

| 观测现象（同一逻辑核、不同地址，除非另有说明） | x86 是否允许 | 常见直观解释（教学用，不等同电路实现） |
|---|---|---|
| 较新 Load 早于较旧 Load | **否** | 架构保证 Load–Load 次序 |
| 较新 Store 早于较旧 Store | **否** | 架构保证 Store–Store 次序 |
| 较新 Store 早于较旧 Load | **否** | 架构保证 Load 不越过更旧的 Store |
| **较新 Load 早于较旧 Store 对外可见** | **是 ⚠️** | Store Buffer 等使写对其他核延后可见 |

正是这种「别人已经看到你后发起的读，却还没看到你先发起的写」的窗口，会在裸机上拆穿 Dekker 一类玩法；同时也是 C++ 里不同 `memory_order`、不同屏障在 **x86 上映射成本** 拉开差距的根源——你要不要把行为钉到「像 SC 那样整齐」，硬件与编译器就要不要多付一道「全钉死」的价钱。

---

## 2. 核心硬件机制

### 2.1 Store Buffer 与 Store-to-Load Forwarding

当 CPU 执行一条写指令时，数据**不会立即写入 L1 Cache**，而是先进入 **Store Buffer**（每个核私有的缓冲区）：

> **注**：Store Buffer 在架构层只保证 Store-Store 可见次序单调，具体实现并不一定是严格 FIFO——不同地址的 store 在部分微架构中可以乱序提交，只要从软件视角的可见次序仍满足 ISA 契约即可。下文"先进先出"的直觉仅用于说明跨核可见性的滞后，请勿当作微架构的精确描述。

```
执行 STORE x=1
       ↓
  Store Buffer  [x=1, ...]   ← 私有，其他核不可见
       ↓  （异步，稍后）
  申请缓存行独占权（RFO）
       ↓
  L1 Cache 中 x 所在行变为 Modified
       ↓
  x=1 对其他核可见
```

**Store-to-Load Forwarding**：同一颗核若紧接着读自己刚写过的地址，值可以直接从 Store Buffer 拿来用，而不必等那一笔写进 L1。这保证了 **单核视角的自洽**，却**管不着**别的核何时看见这次 Store——跨核可见仍要走缓存一致性那条慢一点的路线。

### 2.2 Load 次序与投机执行

乱序核里面自然有 Load 队列、投机执行；**对写并发代码的人而言，要紧的是架构对外说了什么**，而不是某篇笔记里画的「Load Buffer 是否严格 FIFO」——换一代微架构，微观图画法可能就失真。

x86 承诺：**同一逻辑核上，较晚发出的 Load 在全局可见次序里不会挤到较早的 Load 前面**（Load–Load 保序）。所以在**只问「这条 acquire 负载要不要多发一条机器指令」**时，常见答案是一条普通 `mov` 再加编译器侧约束。**但这层答案只管 ISA 映射**：*happens-before*、*synchronizes-with* 仍由 C++ 规则和你在 release/acquire 上做的配对说了算，不能偷换成一句「硬件已经帮我排好了」。

### 2.3 MESI 与跨核失效（教学模型）

多核间的缓存一致性由 MESI 协议维护，每条缓存行处于四种状态之一：

- **M（Modified）**：本核独占，已修改，与内存不一致
- **E（Exclusive）**：本核独占，未修改，与内存一致
- **S（Shared）**：多核持有副本，均与内存一致
- **I（Invalid）**：缓存行无效，下次访问触发 Cache Miss

当某核要写一行尚处 **S（Shared）** 状态的缓存行时，须经由一致性协议通知其它持有副本的核（失效 invalidation、窥探 snoop 等——具体报文与名称因实现而异）。教材里常再叠一层 **Invalidate Queue**：先快速回 Ack，稍后再真正处理失效，用来解释弱模型上「应答」与「生效」可以不同步。**硅片未必长这副模样**，但 **ISA 许诺给软件的次序** 必须自洽。

与本文关系最大的推论是：**x86 在架构层已经较强**，因此 Linux 里 `smp_rmb()` / `smp_wmb()` 在 x86 上往往只剩 **编译器屏障**；而不少 **ARM** 场景则必须在热点路径显式写 `dmb` / `dsb` / `isb`。这里对比的是 **ISA 契约**，不要说成「x86 每条 Load 前都会把某某队列排空」那种绑定具体微结构的断言。

---

## 3. Store-Load 重排的真实原因

Store-Load 之所以能「刁钻」，核心事实只有一句：**Store Buffer 是每核私有的**，别的核天然看不见里面未落地的那几笔。

经典的 Dekker 场景（初始 `x=0, y=0`）：

```
Core 0:                    Core 1:
  STORE x = 1                STORE y = 1
  r0 = LOAD y                r1 = LOAD x
```

**执行时序**：

1. Core 0 执行 `STORE x=1`：写入 Core 0 的 Store Buffer，**尚未提交到 Cache**，此时 Invalidate 消息还没有发出
2. Core 1 执行 `STORE y=1`：写入 Core 1 的 Store Buffer，同上
3. Core 0 执行 `LOAD y`：检查自己的 Store Buffer，无 y；去 Cache 读，此时 Core 1 的 `y=1` 还困在 Core 1 的 Store Buffer 里，Cache 中 y 仍为 0 → **读到 0**
4. Core 1 执行 `LOAD x`：同理 → **读到 0**

于是 **`r0=0, r1=0` 完全合法**：从程序清单看仿佛「先写后读」，从**各自 Store Buffer 与缓存状态**看却只是两个读都太快，对方写尚未穿透到本核读路径。

值得收一句口径：**错怪的不该是「Load 故意绕开 Store Buffer」**——本核读本核新近写，转发是会发生的。真正折腾人的是 **跨核可见性**：你的 Store 在别核眼里还没「算数」，别人的读仍可能拿到旧缓存里的值。

---

## 4. C++ Memory Order 到 x86 汇编的映射

### 4.1 relaxed

```cpp
// C++
x.store(1, std::memory_order_relaxed);
int v = x.load(std::memory_order_relaxed);
```

```asm
; x86 生成
mov DWORD PTR [x], 1   ; 普通写，进 Store Buffer
mov eax, DWORD PTR [x] ; 普通读
```

`relaxed` **只保证**这一次读/写作为原子单元不会被撕裂，**不讲**与前后内存操作谁先谁后。对「对齐的天然字长」访问，x86 上通常就是普通 `mov`，与手写汇编的读写并无二致。

### 4.2 acquire

```cpp
int v = flag.load(std::memory_order_acquire);
// 之后的读写不能重排到这条 load 之前
```

```asm
mov eax, DWORD PTR [flag]   ; 普通 MOV，无额外指令
; + 编译器屏障 asm volatile("" ::: "memory")
```

**为什么在 x86 上常常是「一条 `mov`」**：架构已保证 Load–Load、Load–Store 的次序，编译器再配一道 **编译器屏障**，就能把 C++ 要求的「后面的内存访问别挪到这次 `acquire` 读之前」落实下来。至于流水线里的队列怎么排，留给性能极客；**写证明时不要拿 FIFO 故事当真公理**。

### 4.3 release

```cpp
// 之前的读写不能重排到这条 store 之后
flag.store(1, std::memory_order_release);
```

```asm
mov DWORD PTR [flag], 1   ; 普通 MOV，无额外指令
; + 编译器屏障 asm volatile("" ::: "memory")
```

**为什么在 x86 上常常是「一条 `mov`」**：Store–Store、以及「Store 不许插到更老的 Load 前」已由架构背书，`memory_order_release` 在机器码层多半仍是普通写，外加编译器屏障挡住源码级乱序。同理，**别用「Store Buffer FIFO」顶替 C++ 语言里的 release 证明**。

### 4.4 acq_rel

`acq_rel` 只用于 RMW（Read-Modify-Write）操作：

```cpp
int old = counter.fetch_add(1, std::memory_order_acq_rel);
```

```asm
lock xadd DWORD PTR [counter], eax
```

带 `LOCK` 前缀的指令在做三件事（表述按「架构效果」归纳）：

1. 把读–改–写包成**原子**一步，并在全局可见次序上立起一个**很强的定序点**；口语常说它会「等还没对外露头的写先落定」，**具体管线故事以厂商手册为准**。
2. 经一致性协议拿到该行的**排他**权限（可理解成走向 MESI 的 E/M），其它核上旧副本随之失效或被替换。
3. 结果写回后，别核再读便是新值。

开销与所有「锁总线/锁缓存行」式 RMW 同量级；要不要把它和 **带栅栏的 `seq_cst` Store**（`mfence`、带 `LOCK` 的 `xchg` 等）比快慢，要看**具体 CPU 与前后文**，没法一句话定死。

### 4.5 seq_cst

```cpp
flag.store(1, std::memory_order_seq_cst);   // Store
int v = flag.load(std::memory_order_seq_cst); // Load
```

```asm
; Store：常见两种套路，择一（视编译器与上下文）
xchg DWORD PTR [flag], eax   ; 方式一：XCHG 隐含 LOCK
; 或
mov DWORD PTR [flag], 1
mfence                        ; 方式二：MOV + MFENCE

; Load：
mov eax, DWORD PTR [flag]    ; 与 acquire 相同，无额外指令
```

按 Intel SDM 的条文：`MFENCE` 把**此前**已发的 Load/Store 与**此后**已发的 Load/Store 在**内存次序上**划开，不得在栅栏两侧乱穿。坊间「排空写缓冲、等一致性落定」的说法有助于想象画面，**真正作准的仍是手册里的可观察语义**。

因此 `seq_cst` **Store** 往往要搭 `MFENCE` 或带 `LOCK` 的写这一类「硬栅栏」；而 `seq_cst` **Load** 在 x86 上仍常与 `acquire` 一样，落到普通读加编译器约束。**别忘了语言层**：C++ 的 `seq_cst` 还要求**所有**标成 `seq_cst` 的原子共享**单一全序**——硬件碰巧很强，**不能**当作你在别的 ISA 上也可以不写 `memory_order` 的借口。

---

## 5. 编译器屏障 vs CPU 屏障

线上踩坑最常发生在这里：把「编译器听不听话」和「CPU 乱不乱序」混成一锅。

| 层次 | 机制 | 作用 | x86 指令 |
|---|---|---|---|
| **编译器层** | `asm volatile("" ::: "memory")` | 阻止编译器跨过这一点调度/合并内存访问，也别把「内存里的最新值」无限期只留在寄存器里 | 不生成任何机器指令 |
| **CPU 层** | `MFENCE` | 对此指令前后的 Load/Store 做**全方向的内存定序** | 真实指令，确有成本 |
| **CPU 层** | `LFENCE` | 约束 **Load** 及少数与指令流排序相关的情形 | **不能**单凭它解决「还要管住 Store」的同步 |
| **CPU 层** | `SFENCE` | 偏向 **WC（Write Combining）** 等写路径；对日常 **WB** 内存**不是** `MFENCE` 的平替 | 别当万能栅栏用 |

概括一句：在 x86 上，**acquire/release 的单次读写**多半只要 **编译器屏障**；**`seq_cst` Store** 才往往要 **`MFENCE` 或带 `LOCK` 的写**。

另：**`volatile` 不是线程同步工具**。它只防编译器做一部分「意想不到的优化」，**不承担**内存顺序语义；多线程里拿它顶替 `std::atomic` 仍是 **未定义行为**。

---

## 6. Linux 内核的屏障实现

Linux 在 x86 上的屏障宏充分利用了 TSO 的特性：

```c
/* arch/x86/include/asm/barrier.h */

/* 编译器屏障：仅防止编译器重排，零 CPU 代价 */
#define barrier() asm volatile("" ::: "memory")

/*
 * x86 TSO 保证 Load-Load 和 Store-Store 有序
 * smp_rmb 和 smp_wmb 只需要告诉编译器不要重排即可
 * 不生成任何 CPU 指令
 */
#define smp_rmb()   barrier()
#define smp_wmb()   barrier()

/*
 * smp_mb()：完整内存屏障，需要真实 CPU 指令
 * 使用 LOCK 前缀的副作用而非 MFENCE，在部分 CPU 上更快
 * lock addl $0, 0(%rsp) 对栈顶执行一个无副作用的加法
 * 但 LOCK 前缀强制 Store Buffer 排空
 */
#define smp_mb()    \
    asm volatile("lock; addl $0,0(%%rsp)" ::: "memory", "cc")

/* Release store：普通 MOV + 编译器屏障 */
#define smp_store_release(p, v)             \
    do {                                    \
        barrier();                          \
        WRITE_ONCE(*p, v);                  \
    } while (0)

/* Acquire load：普通 MOV + 编译器屏障 */
#define smp_load_acquire(p)                 \
    ({                                      \
        typeof(*p) ___p = READ_ONCE(*p);    \
        barrier();                          \
        ___p;                               \
    })
```

因此你会看到：内核宁愿用 **`lock; addl $0,…` 一类的 `LOCK` 运算**来实现 `smp_mb()`，也不总是发 `MFENCE`——这是在**不牺牲语义**的前提下抠那几纳秒，**哪条更快随代际而变**。再提醒一遍：`LFENCE` / `SFENCE` 的适用面比 `MFENCE` 窄，**别随手互换**。读 `barrier.h` 务必带上**架构与内核版本**。

---

## 7. acq_rel vs seq_cst 的本质差异

一句话分清二者：**差在 C++ 给了你什么证明题**，不（完全）等于 x86 上哪条指令贵几 cycle。

- **`memory_order_acq_rel`**：钉住**这一条 RMW** 与之相邻读写的 *happens-before* 搭配——本质是 **release + acquire 二合一**（针对该原子上的读写相位）。
- **`memory_order_seq_cst`**：除了以上各类次序，还要求程序里**所有**标成 `seq_cst` 的原子，共同参与**同一条**全程序全序；各线程里自己看到的 `seq_cst` 先后也要服从这条全序。

落到 x86：**`seq_cst` Store** 往往明显肉疼；**带 `LOCK` 的 RMW** 本来就贵一圈，于是「`acq_rel` RMW」与「`seq_cst` Store」谁更伤，要就事论事，别笼统说「差不多」。

### 7.1 acq_rel 在语言里到底保证什么

把 **`memory_order_acq_rel`** 用在 **RMW** 上时，可以把它想成：这一笔原子更新在**读的一侧**携带 acquire 语义、在**写的一侧**携带 release 语义，从而与线程里前后的访问结成语言所许诺的 *synchronizes-with* / *happens-before*。**它不等于**在「SC 全序时间表」里插了一枚全局桩——那是 **`seq_cst` 多出来的那一层**。

### 7.2 真正的分水岭：IRIW（四线程），不是「单读者双写者」

下面 **IRIW（Independent Reads of Independent Writes）** 才是教科书里专门用来「切开」`seq_cst` 与非 SC 混搭的配置（初始 `x=y=0`）：

```
线程 A: x.store(1, …);
线程 B: y.store(1, …);

线程 C: r1 = x.load(…); r2 = y.load(…);
线程 D: r3 = y.load(…); r4 = x.load(…);
```

- 若四个位置用的是 **`relaxed`** 或**凑不出**那条「全体 `seq_cst` 共享的全序」的混搭，**允许**出现 **`r1=1, r2=0, r3=1, r4=0`**：两名读者对两次**互不依赖的写**「谁先谁后」各说各话——这在 **C++ 抽象机**上合法，也正是弱 ISA 上要加栅栏的经典动机。
- 若四个操作都是 **`memory_order_seq_cst`**，则上述结果 **不合法**：`seq_cst` 的全序禁止两个读者对这两次 **独立写** 看到「互相矛盾的」全局先后。

**别混了场景**：若只有一个线程先 `load x` 再 `load y`，即便看到 `1, 0`，多数时候只能说明 **`y` 的那笔写还没推到当前核的读路径上**，**不足以**单独论证「要不要上 `seq_cst`」。要讨论 **SC 全序**多付的那点语义，**至少**要拉来 **两名读者**，照上面 IRIW 那样「各读一遍、读序还相反」。

### 7.3 与 x86 的关系（脚注式提醒）

还有一点心理陷阱：**C++ 说「允许」某种乱序，不代表你手里的 x86 真测得出来**——TSO 更强，试件可能永远「显得守规矩」。写库、写规范级推理时，仍要按**最不该指望的那台机器**来选 `memory_order`，不能赌「反正 x86 测不到」。

---

## 8. HFT 中的 Lock-Free 实践

HFT 的系统级约束可以压成一句：**热点路径上，宁可多敲几百行无锁，也不吃一把 OS 互斥带来的抖动**。无锁在这里常常不是「优化」，而是 **QoS**。

### 8.1 SPSC Queue（单生产者单消费者无锁队列）

最典型的拆法是把 **收包线程** 和 **策略线程** 用 **SPSC** 接起来：索引用 **acquire/release** 配一对，通常是 **最省指令** 的无锁原型之一。

```cpp
template<typename T, size_t Capacity>
class SPSCQueue {
    // 缓存行对齐，防止 false sharing
    // 生产者独占 head，消费者独占 tail
    // 两者在不同缓存行，消除跨核竞争
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
    alignas(64) T buffer_[Capacity];

public:
    bool push(const T& item) {
        const size_t h = head_.load(std::memory_order_relaxed);
        const size_t next = (h + 1) % Capacity;

        // acquire：避免「先当队列没满、后头却又读到落后 tail」这一类编译器重排
        if (next == tail_.load(std::memory_order_acquire))
            return false; // 队列满

        buffer_[h] = item;

        // release：先写 payload，再公布 head；与 pop 侧 acquire 成对
        head_.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        const size_t t = tail_.load(std::memory_order_relaxed);

        // acquire：与 push 的 release 配对，保证能看到对应槽位上的数据
        if (t == head_.load(std::memory_order_acquire))
            return false; // 队列空

        item = buffer_[t];

        // release：把「已读消费槽」这一事实发布给消费者侧；与 push 的 acquire 成对
        tail_.store((t + 1) % Capacity, std::memory_order_release);
        return true;
    }
};
```

**实现上多嘴三句**：

1. **`head_` / `tail_` 分行对齐**，免得 false sharing 把两个无关更新绑在同一条缓存线上互踩。
2. **release–acquire 配的是「数据先就序、索引后露脸」**，证 publish–subscribe 时走 C++ 语义；x86 上映射成普通 `mov` 是 **ISA 赏脸**，换平台别指望。
3. 在 x86 上这条环路往往 **不额外发栅栏指令**，成本主要在 **编译器层面的 `memory` clobber**。

### 8.2 Order Book 的无锁撮合

订单簿快照每秒上百万次跳动时，**一把大锁足以把信号拖成噪声**；双缓冲单写者多读者是常见减压阀：

```cpp
// 双缓冲快照（单写者）：在 inactive 缓冲区上构图，再一次性发布版本号
// 约定：version 恒为偶数且单调 +2；gen = v/2，当前活跃槽 = gen % 2
class OrderBookSnapshot {
    struct Book {
        double bid_prices[10];
        double ask_prices[10];
        double bid_sizes[10];
        double ask_sizes[10];
        uint64_t sequence;
    };

    alignas(64) Book books_[2];
    alignas(64) std::atomic<uint32_t> version_{0};

public:
    void update(const MarketData& md) {
        const uint32_t v = version_.load(std::memory_order_relaxed);
        const uint32_t gen = v / 2;
        const uint32_t write_slot = (gen % 2) == 0 ? 1u : 0u; // 写「非当前活跃」槽

        Book& b = books_[write_slot];
        // ... 填充 b ...
        b.sequence = md.seq;

        // 与读者侧配对：先完成对 books_[write_slot] 的写，再切换到下一代 gen
        version_.store(v + 2, std::memory_order_seq_cst);
    }

    bool read(Book& out) {
        const uint32_t v1 = version_.load(std::memory_order_acquire);
        const uint32_t read_slot = (v1 / 2) % 2;
        out = books_[read_slot];
        const uint32_t v2 = version_.load(std::memory_order_acquire);
        return v1 == v2; // 不相等则说明读期间发生过版本发布，应外层重试
    }
};
```

### 8.3 无锁统计计数器（多写者场景）

成交量、笔数这类 **多生产者累加**，核心往往是「别撕裂、别太拖慢」：

```cpp
// 使用 fetch_add 的无锁计数器：独立累加用 relaxed 即可（只有原子性/进度保证）
class TradeCounter {
    alignas(64) std::atomic<int64_t> volume_{0};
    alignas(64) std::atomic<int64_t> count_{0};

public:
    void record_trade(int64_t qty, double price) {
        volume_.fetch_add(qty, std::memory_order_relaxed);
        count_.fetch_add(1, std::memory_order_relaxed);
        // 两个 fetch_add 彼此无序：若报表需要「同一时刻截面」，需 seqlock/合并原子等更强协议
    }

    // 示意：两次独立 load 形不成「同一时刻截面」；真报表应对两个量做 seqlock 或打包原子
    std::pair<int64_t, int64_t> snapshot() {
        return {
            volume_.load(std::memory_order_seq_cst),
            count_.load(std::memory_order_seq_cst)
        };
    }
};
```

### 8.4 Wait-Free 的策略信号传递

alpha 一类信号经常要求 **写侧永不自旋**；读侧宁可多看半遍旧值，也不能卡在锁上：

```cpp
// Wait-free 信号槽：生产者 wait-free 写，消费者 wait-free 读
// 适用于：信号频率高、偶尔丢失可接受、绝不阻塞
template<typename T>
class WaitFreeSignalSlot {
    alignas(64) std::atomic<uint64_t> seq_{0};
    alignas(64) T data_;

public:
    // Wait-free write：始终成功，无循环，无等待
    void write(const T& signal) {
        // 奇数 seq 表示写入进行中
        seq_.fetch_add(1, std::memory_order_relaxed);
        data_ = signal; // 非原子写，被 seq 的 release 保护
        seq_.fetch_add(1, std::memory_order_release);
    }

    // Wait-free read：始终返回，可能是稍旧的值
    bool read(T& out) {
        uint64_t s1 = seq_.load(std::memory_order_acquire);
        if (s1 & 1) return false; // 写入进行中，跳过

        out = data_;

        uint64_t s2 = seq_.load(std::memory_order_acquire);
        return s1 == s2; // false 表示读取期间被写入，值可能不完整
    }
};
```

**落笔警示**：`data_` 若仍是普通成员，上面这种「一线程写、一线程读」在 **C++ 标准里是数据竞争**，除非你对 `T` 有极窄的例外（一般别赌）。这里只想展示 **seqlock 握手** 的节拍；工程上请换成 **双缓冲**、**原子载荷**，或 **只 swap 指针**，把未定义行为关进笼子。

---

## 9. Web3 中的 Wait-Free 实践

节点里 **验签吃 CPU、P2P 吃 IO**，两件事必须并排跑，还要尽量少互相拖后腿——无锁队列 / 无锁池子多是为此攒的。

### 9.1 无锁 Mempool（交易池）

Mempool 是区块链节点中竞争最激烈的数据结构：网络线程不断插入新交易，打包线程不断读取高 fee 交易，验证线程并发验证签名：

```cpp
// 基于 hazard pointer 的无锁 Mempool
// 核心：用 hazard pointer 实现安全的内存回收，避免 ABA 问题
class LockFreeMempool {
    struct TxEntry {
        Transaction tx;
        std::atomic<TxEntry*> next;
        uint64_t fee_rate;
    };

    // 按 fee_rate 分桶的无锁跳表头节点
    // 每个桶是一个无锁链表
    static constexpr int FEE_BUCKETS = 256;
    alignas(64) std::atomic<TxEntry*> buckets_[FEE_BUCKETS];

public:
    // 网络线程：insert，lock-free
    bool insert(Transaction tx) {
        int bucket = fee_bucket(tx.fee_rate);
        auto* entry = new TxEntry{std::move(tx), nullptr, tx.fee_rate};

        TxEntry* expected = buckets_[bucket].load(std::memory_order_relaxed);
        do {
            entry->next.store(expected, std::memory_order_relaxed);
            // CAS：compare_exchange_weak 在 x86 上是 LOCK CMPXCHG
            // acq_rel：成功时有完整屏障语义，其他线程读到新 head 时
            // 必然也能看到 entry 的内容（release 侧）；
            // 失败时 acquire 读到最新的 head 继续重试
        } while (!buckets_[bucket].compare_exchange_weak(
            expected, entry,
            std::memory_order_acq_rel,   // 成功
            std::memory_order_acquire    // 失败
        ));
        return true;
    }
};
```

**上例**只画了「头插 + `compare_exchange_weak` 次序」这一刀；注释里喊的 **hazard pointer / epoch** 管的是 **何时真能 `delete`**。缺了回收协议，`ABA`、悬空指针会把这段代码改成 **CVE 预备役**——此处从略不表示不重要。

**`compare_exchange_weak` 怎么选序**：成功一侧要发布新 head，用 **`acq_rel`**；失败一侧没有可见写，用 **`acquire` 拉回最新 expected** 即可。x86 上整条路径仍旧 **`LOCK CMPXCHG`**，成败由微码一肩扛，但 **C++ 侧**仍要严肃写好 success/failure order。

### 9.2 无锁 UTXO 缓存

比特币节点的 UTXO 缓存需要支持高并发读（交易验证）和低频写（区块确认）：

```cpp
// 写时复制（Copy-on-Write）的无锁 UTXO 视图
// 读者 wait-free，写者 lock-free
class UTXOCacheView {
    struct Snapshot {
        std::unordered_map<OutPoint, UTXO> utxos;
        uint32_t block_height;
        std::atomic<uint32_t> ref_count{1};
    };

    // 当前活跃快照
    alignas(64) std::atomic<Snapshot*> current_;

public:
    // 验证线程调用：wait-free 读
    // 获取当前快照引用，期间快照不会被释放
    Snapshot* acquire_snapshot() {
        Snapshot* snap = current_.load(std::memory_order_acquire);
        // acquire 保证：读到 snap 指针后，对 snap 内容的访问
        // 不会被重排到 load 之前，即内容对我们是可见的
        snap->ref_count.fetch_add(1, std::memory_order_relaxed);
        return snap;
    }

    void release_snapshot(Snapshot* snap) {
        if (snap->ref_count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            // acq_rel：确保看到所有 fetch_add 之后才决定释放
            // fetch_sub 返回 1 说明我是最后一个持有者
            delete snap;
        }
    }

    // 区块确认线程调用：lock-free 写
    // 创建新快照，原子替换，旧快照在最后一个读者释放时销毁
    void apply_block(const Block& block) {
        Snapshot* old = current_.load(std::memory_order_acquire);
        Snapshot* fresh = new Snapshot(*old); // 深拷贝

        // 在新快照上应用区块变更
        for (const auto& tx : block.txs) {
            for (const auto& input : tx.inputs)
                fresh->utxos.erase(input.outpoint);
            for (size_t i = 0; i < tx.outputs.size(); ++i)
                fresh->utxos[{tx.txid, i}] = tx.outputs[i];
        }
        fresh->block_height = block.height;

        // seq_cst store：确保所有验证线程在看到新快照指针时
        // 也能看到新快照的完整内容
        // 旧快照的 ref_count 已被 acquire_snapshot 增加的读者持有
        // 最后一个读者 release 时会 delete old
        if (!current_.compare_exchange_strong(
            old, fresh,
            std::memory_order_seq_cst,
            std::memory_order_acquire
        )) {
            delete fresh; // 另一个写者先提交了，重试
        } else {
            release_snapshot(old);
        }
    }
};
```

### 9.3 无锁签名验证队列

区块链节点需要批量验证交易签名（ECDSA/BLS），这是 CPU 最密集的操作。使用 MPSC（多生产者单消费者）无锁队列将网络接收和签名验证解耦：

```cpp
// MPSC 无锁队列：多个网络线程生产，单个验证线程消费
// 示意：Vyukov 式 MPSC 轮廓（非完整 Michael–Scott）；注释与下文文字一致
template<typename T>
class MPSCQueue {
    struct Node {
        T data;
        std::atomic<Node*> next{nullptr};
    };

    alignas(64) std::atomic<Node*> head_; // 消费者独占
    alignas(64) std::atomic<Node*> tail_; // 生产者竞争

public:
    MPSCQueue() {
        Node* dummy = new Node{};
        head_.store(dummy, std::memory_order_relaxed);
        tail_.store(dummy, std::memory_order_relaxed);
    }

    // 生产者：lock-free，多线程安全
    void push(T item) {
        Node* node = new Node{std::move(item)};

        // 原子地将 node 追加到链表尾部
        // exchange 获取旧 tail，然后将旧 tail 的 next 指向 node
        Node* prev = tail_.exchange(node, std::memory_order_acq_rel);
        // release 保证 node 的内容在 next 指针可见前已写入完成
        prev->next.store(node, std::memory_order_release);
    }

    // 消费者：wait-free（单消费者无竞争）
    bool pop(T& out) {
        Node* head = head_.load(std::memory_order_relaxed);
        // acquire：读到 next 后，对 next->data 的访问不会上移
        Node* next = head->next.load(std::memory_order_acquire);

        if (!next) return false;

        out = std::move(next->data);
        head_.store(next, std::memory_order_relaxed);
        delete head; // 释放哑节点
        return true;
    }
};
```

与 **Michael–Scott** 经典实现并不是同一段伪代码；更接近 **Vyukov** 那条「`tail.exchange` + 链 `next`」路线。再次强调：**`pop` 里裸 `delete`** 要结合 **epoch / hazard pointer**，否则别的生产者还可能透过 `prev->next` 摸到已释放内存——正文略去的是 **正确性**，不是 **可选优化**。

---

## 10. 选择 Memory Order 的决策框架

把前文收束成一棵**实用但非穷举**的决策树（真上线仍要对算法单独证明）：

```
需要原子操作吗？
├── 否 → 普通变量，注意编译器不要优化掉（volatile 或 WRITE_ONCE）
└── 是
    ├── 只关心操作本身原子，不关心顺序？
    │   └── memory_order_relaxed
    │       典型场景：计数器的最终统计、引用计数的 fetch_add
    │
    ├── 是 RMW 操作（fetch_add, CAS, exchange）且需要顺序保证？
    │   └── memory_order_acq_rel
    │       典型场景：CAS 循环、无锁栈 push/pop
    │
    ├── 是 Load，且后续操作依赖此 Load 的结果？
    │   └── memory_order_acquire
    │       典型场景：读取锁标志后访问被保护数据
    │
    ├── 是 Store，且希望之前的写在此 Store 后对其他核可见？
    │   └── memory_order_release
    │       典型场景：写完数据后设置标志通知消费者
    │
    └── 论证需要「所有 `memory_order_seq_cst` 原子共享单一全序」（如禁止四线程 IRIW）？
        └── memory_order_seq_cst
            典型：多标志与多读者同时存在、要用 SC 全序说清因果
            注意：在 x86 上 **Store** 端常比 `release` 更贵（`MFENCE`/带 `LOCK` 的写）；**Load** 端仍多为普通读
```

### x86 上的实际代价对比

| Memory Order | Store 代价 | Load 代价 | 数量级（Skylake 类，仅供体感） |
|---|---|---|---|
| relaxed | ~4 cycles（L1 hit 量级） | ~4 cycles | < 1ns 量级 |
| acquire | ~4 cycles | ~4 cycles | < 1ns 量级 |
| release | ~4 cycles | ~4 cycles | < 1ns 量级 |
| acq_rel (RMW) | ~15–20 cycles（`LOCK`） | （同一条 `LOCK`） | ~数 ns |
| seq_cst Store | 常高于普通 Store（`MFENCE` / 带 `LOCK` 的写） | ~4 cycles | 因实现与周围指令而异 |

在 x86 上，**多数时候** acquire/release **不会**比 relaxed 多出一条 **CPU 栅栏**（编译器屏障照样要）。**ARM** 上同一套 C++ 却往往要长出 **`dmb`**。所以 HFT 爱在 x86 上抠热点没错，但 **写库**仍要按「最蠢的那颗核」来选 `memory_order`，别指望「反正我会在 x86 上跑」。

---

## 参考资料

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 3A, Chapter 8: Memory Ordering
- Paul E. McKenney, *Memory Barriers: a Hardware View for Software Hackers*
- Herb Sutter, *atomic<> Weapons: The C++ Memory Model and Modern Hardware*
- Linux Kernel Documentation: `Documentation/memory-barriers.txt`
- Maurice Herlihy & Nir Shavit, *The Art of Multiprocessor Programming*
- Preshing on Programming: https://preshing.com/20120913/acquire-and-release-semantics/
