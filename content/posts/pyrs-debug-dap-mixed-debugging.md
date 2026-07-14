+++
date = '2026-07-14T23:00:00+08:00'
draft = false
title = 'DAP 代理与 Python + Rust 混合调试'
tags = ['rust']
description = '跨语言调试的本质是一个进程、两套调试接口；DAP 代理如何把 debugpy 与 LLDB 编排成单一 IDE 会话，以及 PyO3 场景下的时序与符号约束。'
+++

# DAP 代理与 Python + Rust 混合调试

PyO3 把 Rust 编译成 `.so` / `.pyd`，Python 进程 `import` 时动态加载。调试器眼里却是两套互不兼容的接口：`.py` 走 debugpy，`.rs` 走 LLDB。进程只有一个，栈帧却在两种语言间来回切换——这是混合调试的出发点，也是难点所在。

## 跨语言调试在解决什么

调试器要回答三件事：**停在哪、看见什么、下一步怎么走**。单语言时，adapter 直接绑一种运行时：解释器栈、JIT 帧、或 DWARF 符号表，三者自洽。

混合运行时打破这个假设。一次 `import my_rust_ext` 之后，调用链可能是 `main.py → ffi 边界 → lib.rs`，但：

- **符号分属两套体系。** Python 帧带 `source.path`、局部变量名；Rust 帧靠 DWARF，断点要绑到编译产物里的地址。IDE 按文件设断点，底层却得分别找 bytecode offset 和 machine code。
- **控制权跨边界转移。** 停在 Python 帧时，LLDB 的 step-over 不知道下一跳会不会进 native；停在 Rust 帧时，debugpy 拿不到 C 栈上的 `PyObject*`。用户按一次 F10，必须落到「当前停住的那一侧」。
- **模块加载有先后。** PyO3 扩展在 Python 解释器跑起来之后才 `dlopen`。若 native 调试器 attach 太晚，入口处的 Rust 初始化已经执行完，断点来不及绑。

所以混合调试不是「装两个插件」那么简单，而是要在**一个 OS 进程、一个 IDE 会话**里，协调两套 adapter 的生命周期、断点状态和步进语义。

## compound 为什么不够

VS Code 的 compound launch 能同时起 debugpy 和 CodeLLDB，但 IDE 仍把它们当成两个独立会话：两套工具栏、两份 `stopped` 事件、两份断点面板。

手工对齐有三处摩擦：

1. **时序。** LLDB 必须等 Python 进程起来再 attach；attach 前设的 native 断点常被静默丢弃。
2. **断点合并。** IDE 对同一工作区发 `setBreakpoints`，compound 不会自动按扩展名分流，两侧可能各自收到不该管的请求。
3. **活跃侧切换。** 用户不知道当前 F10 该发给谁；两侧同时 `stopped` 时，变量面板和调用栈会对不上。

compound 能跑通，但编排逻辑落在人身上。代理的目标是把这三件事写进协议层。

## DAP 为何适合做代理

[DAP](https://microsoft.github.io/debug-adapter-protocol/) 把 IDE（Client）和语言运行时（Adapter）拆开，JSON 消息走 `Content-Length` 帧。Client 只认 `launch` / `setBreakpoints` / `stopped` 等请求，不认 Python 还是 Rust。

这意味着可以在中间插一层 **DAP Proxy**：

```
IDE (Client)
     │
     ▼
 DAP Proxy ──┬── debugpy  (.py)
             └── LLDB     (.rs / native)
                     │
                     ▼
         同一 OS 进程 (Python + 内嵌 .so/.pyd)
```

代理注册自定义 `type`，用 `DebugAdapterServer` 起本地 TCP 服务。IDE 连上来后，代理 spawn 两个子 adapter，在三路流之间转发、改写、合并消息。对外始终是一个 `type`、一个会话。

Python + C++ 的同类实现见 [python-cpp-debugger-ext](https://github.com/bowen-xu/python-cpp-debugger-ext)。架构可复用；PyO3 的差异在符号加载和模块生命周期，不在代理框架本身。

## PyO3 场景的额外约束


| 问题   | Python 侧                         | Rust 侧                                             |
| ---- | -------------------------------- | -------------------------------------------------- |
| 断点绑定 | `source.path` 即 `.py` 文件         | 需把 `.so` 映射回 `.rs`；attach 后常要 `target modules add` |
| 启动时机 | `launch` + `stopOnEntry` 即可拿 PID | 须等 `import` 触发 `dlopen` 后再 attach，否则符号表为空          |
| 步进语义 | `stepIn` 在 Python 帧内有效           | 跨入 FFI 后须切到 LLDB；`stepOut` 回到 Python 帧时再切回         |


Rust 没有 C++ 那样成熟的「双调试器」生态，但 LLDB 对 Rust DWARF 的支持足够用。真正费工的是 **attach 窗口**：太早没有模块，太晚错过初始化断点。

## 代理要编排的协议

**断点路由。** `setBreakpoints` 按 `source.path` 扩展名分流：`.py` → debugpy，`.rs` → LLDB。一次请求若涉及两侧，代理合并响应再回 IDE，否则断点面板状态分裂。LLDB 未就绪时，native 断点标 `verified: false` 并缓存，attach 完成后批量重发——compound 最难手工对齐的正是这一步。

**启动时序。** 典型顺序：debugpy `launch` + `stopOnEntry` → 从 `process` 事件取 PID → LLDB attach → 预加载 `.so` 符号 → 刷新缓存断点 → `configurationDone` → continue。`configurationDone` 不能早于断点处理完毕，否则进程在断点生效前就跑起来。

**活跃 adapter。** 代理记录当前控制权：收到 debugpy 的 `stopped`，步进、变量、栈帧请求走 Python 侧；收到 LLDB 的 `stopped`，切到 native 侧。用户仍是一个 F5 会话，工具栏自动落到停住的一侧。

其余是边角：`breakpointLocations` 不能让 debugpy 改写 native 源的行号；`terminate` 时两侧退出顺序要协调，避免一侧已死、另一侧还在等响应。

## 优缺点

**优点**

- 单一 IDE 会话，断点、步进、变量面板语义统一。
- 不改动 debugpy / LLDB，只编排已有 adapter，维护面小。
- 时序和路由固化在代理里，换机器不用重新对齐 compound 顺序。
- DAP 是 VS Code / Cursor 通用协议，换 Client 时代理层可复用。

**缺点**

- **路由粗糙。** 靠文件扩展名分流，不认 PyO3 宏展开后的内联位置，也不处理 `.pyi` 与实现文件的映射。
- **attach 模型受限。** 当前常见实现是 launch Python + LLDB attach 同 PID，难以 attach 到已在跑的进程。
- **跨边界步进不透明。** 从 Python `stepIn` 进 Rust 时，用户感知不到「切换了调试器」，但代理必须在两侧 `stopped` 之间正确接力；实现有 bug 时表现为步进丢失或停在错误帧。
- **平台差异下沉不彻底。** Windows `.pyd`、Linux `.so`、macOS `.dylib` 的加载命令不同，符号预加载脚本要分平台维护。
- **双 adapter 开销。** 两个子进程、两路 DAP 流，启动比单语言慢；调试会话占用的句柄和端口更多。

若需求只是「偶尔在 Rust 里打个断点」，compound 或手动 `lldb -p <pid>` 也能凑合。代理的价值在**高频跨边界开发**：需要在 `.py` 和 `.rs` 之间来回步进、看两侧变量，且不想维护两套会话状态。

## 配置与边界

```json
{
  "type": "pyrs-debug",
  "request": "launch",
  "program": "入口脚本或可执行文件",
  "pythonPath": "解释器路径",
  "pythonFileExtensions": [".py"],
  "nativeFileExtensions": [".rs"],
  "lldbAttachToPythonProcess": true
}
```

依赖 Python Debugger（debugpy）和 CodeLLDB；代理只编排，不替代。

当前模式的硬边界：launch-only、扩展名路由、不做 PyO3 内部源映射。维护 Python + Rust 时，改动主要在 `nativeFileExtensions` 默认值和符号预加载脚本；代理架构本身与 Python + C++ 共用。