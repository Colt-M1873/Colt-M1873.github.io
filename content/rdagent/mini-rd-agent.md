---
title: "Mini-RD-Agent：从零开始实现极简的RD-Agent"
date: 2026-10-15T11:00:00+08:00
draft: false
ShowToc: true
TocOpen: false
---

项目地址：[github.com/Colt-M1873/Mini-RD-Agent](https://github.com/Colt-M1873/Mini-RD-Agent)

一个从零实现、逐版本迭代的极简版 **"R&D Agent"**——用来理解
[microsoft/RD-Agent](https://github.com/microsoft/RD-Agent)（一个用 LLM Agent 自动化
数据驱动研发流程的开源框架）到底是怎么运作的，亲手把它的核心思想
一版一版地搭出来。

## 这是什么、为什么做这个

RD-Agent 的核心思想可以用一句话概括：**让 LLM 像科研人员一样，不断"提出假设 → 把假设
写成代码实现 → 真实跑起来验证 → 根据结果反思 → 提出下一个假设"，并且把每一轮的经验都
沉淀下来，让下一轮比上一轮更聪明。**

这个循环思想本身不难理解，但 RD-Agent 的真实代码库经过几年打磨，已经非常庞大、工程化
程度很高（异步并行调度、Docker/Conda 执行环境、知识库检索、多场景插件化……）。直接读
源码很容易"见树不见林"。

mini-RD-Agent 换一个角度：**从一个只有一百多行、一眼能看懂的最小实现开始，每一版只加
一个新能力**，边实现边对照真实项目的设计思路。读完/跑完全部版本，你会对 RD-Agent（以及
一般意义上"LLM Agent 怎么做自动化研发"这件事）建立起一个从底层逐步搭起来的理解，而不是
啃一遍几万行代码之后一知半解。

## 核心循环

```
          ┌──────────────────────────────────────────────┐
          ▼                                                │
   提出假设(Hypothesis)                                     │
          │                                                │
          ▼                                                │
   把假设写成代码(LLM 生成)                                    │
          │                                                │
          ▼                                                │
   真实执行、拿到结果                                           │
          │                                                │
          ▼                                                │
   产出反馈(接受/拒绝) ──────────────────────────────────────┘
          │
          ▼
   记录历史，影响下一轮的假设
```

每一版都是在给这个循环加一层能力：怎么让代码写错了能自己改、怎么安全地执行不信任的代码、
怎么同时探索多个方向、怎么让经验跨轮次复用、怎么并行跑得更快、怎么换一个业务场景复用同一套
骨架……

## 版本规划

| 版本 | 状态 | 新增能力 |
| --- | --- | --- |
| [`v0`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v0) | ✅ 已实现 | 最基本的单线历史循环：提出假设 → LLM 生成代码 → 本地子进程执行 → 规则判断是否为新 SOTA → 记录历史 |
| [`v1`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v1) | ✅ 已实现 | 引入 CoSTEER 式"代码写错了，把报错信息喂回去自己改"的任务内小循环 |
| [`v2`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v2) | ✅ 已实现 | 引入 Docker 沙箱隔离执行（禁网/资源限制/只挂必要目录），替代直接本地跑生成代码的风险做法 |
| [`v3`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v3) | ✅ 已实现 | Trace 从线性 list 升级为支持分叉的 DAG，可以从历史任意节点重新出发尝试新方向 |
| [`v4`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v4) | ✅ 已实现 | 轻量知识库（跨分支共享的"报错→修复"教训）、会话持久化续跑（`--resume`）、LLM 定性判断反馈（取代纯数值比较） |
| [`v5`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v5) | ✅ 已实现 | 并行多路循环：asyncio 并发跑多个分支任务，record/feedback 步骤加锁保证一致性 |
| [`v6`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v6) | ✅ 已实现 | Scenario 抽象化（场景变成可插拔模块）+ 接入第二个场景：基于 [Qlib](https://github.com/microsoft/qlib) 真实 A 股数据的量化因子挖掘 |
| `v7+` | 📝 计划 | 因子库累加/去重、更聪明的分支选择策略（UCB/MCTS） |

**每个版本是完全独立、可以单独拿出来跑的子文件夹**，不依赖其它版本，方便直接对比
"加了某个能力之后代码复杂度涨了多少"——而不是在一份不断膨胀的代码里叠加功能，看不清
每一步到底改了什么。

## 从哪开始看

- **只想知道最终长什么样 / 直接上手跑**：去 [`v6`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v6)（最新版本），`README.md` 里有
  完整的环境搭建和运行步骤。
- **想理解设计是怎么一步步演化出来的**：按 `v0 → v1 → v2 → ...` 顺序读，每个版本的
  `README.md` 都会清楚说明"这一版比上一版多了什么、为什么这么设计、刻意简化了什么"。
  每一版的代码量都不大（几百行量级），单独读一版的成本很低。

## 跑起来需要什么

- Python 3.10+（[`v2`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v2) 及之后的版本需要用到较新的标准库特性和依赖，建议 3.11）
- 一个 OpenAI 兼容的 LLM API Key（OpenAI 官方 / DeepSeek / 任何兼容服务都可以）
- Docker（[`v2`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v2) 开始默认用 Docker 沙箱隔离执行 LLM 生成的代码；不装也能用 `--sandbox local`
  降级运行，但生成的代码会直接在你的机器上跑，没有任何隔离，仅建议用于快速调试）
- [`v6`](https://github.com/Colt-M1873/Mini-RD-Agent/tree/main/v6) 的因子挖掘场景还需要一份 [Qlib](https://github.com/microsoft/qlib) 格式的 A 股
  历史数据，具体怎么拿见 `v6/README.md`

## 这个项目不是什么

这是一个**学习/教学性质**的从零实现，不是 RD-Agent 的精简发行版或替代品——很多真实项目
里的工程能力（更细粒度的并行调度、真正的向量检索知识库、完整的 Qlib 回测框架接入、
插件化配置系统等）被大幅简化或完全没做，每一版的 `README.md` 都会诚实列出"这一版刻意
没做的事"。如果你想要一个能真正用于研究/生产的工具，请直接使用
[microsoft/RD-Agent](https://github.com/microsoft/RD-Agent)。
