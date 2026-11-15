# 手机 AI 主机形态 / Mobile AI Host

> 一台 root 过的 Android 手机，跑着完整的 Ubuntu chroot、AI Agent 运行时与本地记忆体系。
> A rooted Android phone running a full Ubuntu chroot, an AI agent runtime, and a local memory system.

本仓库记录这套自建形态的架构与判据——纯文档，不含任何可执行工具。

This repository documents the architecture and criteria of this self-built setup — documentation only, no executable tools.

---

## 什么才算「真正的 AI 手机」？/ What makes a phone a *real* AI phone?

不看营销词，看四条可验证判据：

Instead of marketing labels, here are four verifiable criteria:

| # | 判据 / Criterion | 说明 / Description |
|---|---|---|
| 1 | **系统级行动权 / System-level agency** | AI 通过 root shell 直接操作文件、进程、服务，而不是模拟点击 UI / The AI acts via root shell — files, processes, services — not UI automation |
| 2 | **可导出的持久记忆 / Exportable persistent memory** | 记忆存本地磁盘、可打包迁移，而不是厂商云端黑盒 / Memory lives on local disk, portable, not a vendor-cloud black box |
| 3 | **自我修复 / Self-repair** | AI 能定位并修复自己运行环境的故障（依赖、补丁、服务重启）/ The AI can diagnose and patch its own runtime |
| 4 | **厂商无关 / Vendor independence** | 体系不依赖任何单一厂商的存续，换硬件可整体迁移 / The stack survives any single vendor's shutdown |

本仓库记录的这台手机通过了全部四条。

The phone documented here passes all four.

---

## 架构总览 / Architecture Overview

```
┌─────────────────────────────────────────────┐
│              Android (rooted)                │
│  ┌───────────────────────────────────────┐  │
│  │         Ubuntu chroot (ARM64)         │  │
│  │                                       │  │
│  │  ┌─────────┐   ┌──────────────────┐  │  │
│  │  │ AI 运行时 │──▶│  工具面 (MCP 等)   │  │  │
│  │  │ (Agent) │   │  Tool layer (MCP) │  │  │
│  │  └────┬────┘   └──────────────────┘  │  │
│  │       │                                │  │
│  │  ┌────▼────┐   ┌──────────────────┐  │  │
│  │  │ 三层记忆  │◀──│  本地模型网关      │  │  │
│  │  │ Memory  │   │  Model gateways   │  │  │
│  │  └─────────┘   └──────────────────┘  │  │
│  │                                       │  │
│  │  编译/分析工具链: Ghidra · jadx · …     │  │
│  │  Toolchain: Ghidra · jadx · …          │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 分层职责 / Layer responsibilities

| 层 / Layer | 组件 / Components | 干什么 / What it does |
|---|---|---|
| 宿主层 / Host | rooted Android + SUKISU 内核级 root | 提供 shell 与系统调用 / Shell & syscalls |
| 系统层 / OS | Ubuntu chroot（真 chroot，非 proot）/ Ubuntu chroot (real, not proot) | 完整 Linux 用户态 / Full Linux userspace |
| 智能层 / Agent | AI 运行时 + 工具调用协议 / Agent runtime + tool-call protocol | 规划、执行、验收 / Plan, execute, verify |
| 记忆层 / Memory | 三层：运行时注入 → 文档 → 长期库 / 3-tier: runtime injection → documents → long-term store | 跨会话人格连续性 / Cross-session continuity |
| 模型层 / Models | 本地聚合网关，多源接入 / Local aggregation gateway, multi-source | 模型自由、可切换 / Model freedom |

---

## 关键设计决策 / Key design decisions

### 为什么是真 chroot 而不是 proot？/ Why real chroot, not proot?

- 性能：proot 的 ptrace 拦截有 5-10 倍 IO/进程开销；真 chroot 直接系统调用
- Performance: proot's ptrace interception adds 5-10x I/O and process overhead; a real chroot is direct syscalls
- 能跑完整工具链：JVM（Ghidra）、Node.js 22、Gradle、Rust 编译目标，在 proot 下要么跑不动要么慢到不可用
- Runs the full toolchain: JVM (Ghidra), Node.js 22, Gradle, Rust targets — impractical under proot

### 为什么记忆分层？/ Why tiered memory?

1. **运行时层 / Runtime tier** — 每轮注入的紧凑事实（上限 10 KB），保证热上下文永远新鲜 / Compact facts injected every turn (10 KB cap) keeps hot context fresh
2. **文档层 / Document tier** — 成篇设计与调研，按需检索 / Full-length designs & research, retrieved on demand
3. **长期层 / Long-term tier** — 证据与历史，语义召回 / Evidence & history, semantic recall

### 自我修复实例 / Self-repair example

网关断流，AI 自查根因：上游 SSE 流内错误码未映射到「软限流」类别，导致重试逻辑永不切换数据源。AI 定位源码行号、编写补丁、重新构建、验证通过。全程无人工介入。

The model gateway dropped streams. The AI traced the root cause: an upstream SSE error code wasn't mapped to the "soft rate limit" category, so retry logic never rotated sources. The AI located the source line, wrote a patch, rebuilt, and verified — no human in the loop.

---

## 这套形态能干什么 / What this form factor can do

- 逆向分析全流程在手机上闭环（脱壳 → 反编译 → 补丁 → 重签名 → 验证）
- Full reverse-engineering loop closed on-device (unpack → decompile → patch → re-sign → verify)
- 服务编排与保活：多进程守护、开机自启、故障自动恢复
- Service orchestration & keep-alive: multi-process supervision, boot autostart, crash recovery
- 跨会话的 AI 工作记忆：今天的会话记得上周的决策与教训
- Cross-session AI working memory: today's session remembers last week's decisions and lessons

## 代价 / The costs (honest)

- root 导致部分金融/银行 App 检测失败，需要权衡
- Some finance/banking apps fail integrity checks on rooted devices
- 维护成本：内核适配、依赖治理、内存回收都是自己的活
- Maintenance is yours: kernel compatibility, dependency governance, memory reclamation
- 全链路稀有 = 踩坑没有 StackOverflow 答案，一切自己排查
- Rarity means no StackOverflow answers — you debug everything yourself

---

## FAQ

**Q: 为什么不用电脑？/ Why not a PC?**

形态自由：算力不够就编排，屏幕太小就远程。随身携带的主机比桌上的主机更常在场。
Form-factor freedom: orchestrate when compute is short, remote in when the screen is small. The host in your pocket is more often present than the one on your desk.

**Q: 和量产「AI 手机」的区别？/ How is this different from commercial "AI phones"?**

量产机的 AI 是厂商派驻的功能模块：能力边界由 OTA 决定、记忆存厂商云、停服即失忆。这套形态里 AI 是机器的常住操作者：权限是系统级的、记忆是本地可导出的、生死不系于任何单一厂商。
Commercial AI phones ship a vendor-stationed feature: capability bounded by OTA, memory in vendor cloud, amnesia on shutdown. In this form factor the AI is the resident operator: system-level authority, local exportable memory, survival independent of any vendor.

**Q: 能复制吗？/ Can this be replicated?**

可以。路径：解锁引导加载器 → 刷 root 方案 → 部署 chroot → 装 Agent 运行时 → 搭记忆体系。每一步都是公开技术，难点在集成与维护纪律。
Yes. The path: unlock bootloader → root → deploy chroot → install agent runtime → build memory tiers. Every step is public knowledge; the hard part is integration and maintenance discipline.

---

## License

CC BY-NC-SA 4.0（文档 / documentation）
