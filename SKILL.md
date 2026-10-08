---
name: "OpenAI Math Skill"
description: "Use when a researcher gives a research topic, a rough question, or a precise open problem in mathematics, theoretical physics, CS theory, or another proof/derivation-heavy field and wants reasoning models to attack it the openai/math way: verified candidate problems from a vague topic, self-contained prove-or-refute statements, best-of-N solving, isolated review and debate, chaining, Lean or computational verification, and honest status labels. 中文触发：用 AI 做数学/理论物理/理论计算机研究、开放问题、猜想、证明或否证、写题面、best-of-N、AI 审稿、Lean 形式化、只有一个研究方向想找题。"
---

# OpenAI 式研究工作流（openai/math 模式）v4

> **定位**：本技能把 openai/math（2026-10 发布，722 篇手稿）公开的提示摘录、推理摘要和审核附件，加上 First Proof 第二批报告和 OpenAI 前作博客，整理成**与工具无关、可降级**的工作流。
> **声明**：
> - §T 的模板以及流程默认值（N、审稿人数、辩论轮数、预算、超时）都是本技能的**重构或建议**，不是 OpenAI 原文或其实际 harness；逐字原文只出现在附录 A。
> - OpenAI 用的是未发布的内部模型。公开模型的产出率要**大幅调低**：First Proof 预跑中，公开模型在 30 分钟超时下，十题**一题也没解出**（附录 A-9）。
> - v3 根据一次真实端到端试跑（2026-10-08，STANDARD 模式，单模型族 Codex；结果 4/4 PARTIAL，原命题无结果；Lean + comparator 只在一个子引理上跑通）修订，摘要见附录 B。
> - v4 根据成熟度测试修订（附录 C）：无上下文 agent 只拿到本技能和题目，端到端跑校准题（真命题）与否证题（假命题），以及模糊方向的选题入口；另有多定理 Challenge、干净验收目录和注入 `sorry` / 私有公理 / `native_decide` 的对抗检查。
> - 凡写"**试跑已验证**"或"**实测**"的命令都实际执行过；写"**未实测**"的只是建议。

## 0. 快速开始（先读这一节）
1. **判定输入**（§3 I-3）：只有方向 → 步骤 1 找候选；粗略问题 → 1 个形式化 + 1–2 个变体；精确陈述 → 直接步骤 2。
2. **环境自检**（§1）：确定 FULL / STANDARD / SOLO 模式，按 §1.1 实现隔离并做一次冒烟检查。
3. **定预算**（§2）：研究者没给就用分阶段默认值，**花费上限必须问一句**。研究者不在线、事先写了授权时，按 §3 I-8（无人值守）执行。
4. **步骤 1**：交付 3–5 个**核实过出处**的候选 → 【停点 H1：研究者选题】。
5. **步骤 2**：按清单 (f) 写 `input.vN.md`（**只放题面**，模板 (a)；必须含检索政策），备好 N 个提示变体（可选 (b)），用模板 (i) 做歧义攻击 → 【停点 H2：签字】。
6. **步骤 3**：best-of-N 求解。每次发送 = (a0) 求解前言 + `input.vN.md` + 〔(b)〕 + (a1) 输出格式；解析 VERDICT 行。
7. **步骤 4**：隔离审稿（模板 (d)，审稿者只拿题面 + 成稿，给两行判词），只有作者**未声明**的问题才辩论（g2），涉及计算就做 clean-room 复核（h）。
8. **步骤 5–7**：链式推进（c）→ 验证（6A Lean 走 e1/e2 并经【H3】；6B 做计算或符号检查）→ 意义筛选与成稿。
9. 全程按 §2 的停止规则（每条规则触发后做什么见 §2 表），只在【H4 里程碑】时写正式状态报告（I-5）；平台要求的简短进度消息可以发，但不得提问、不得增加停点。所有产物放进 `research/<id>/`（§9）。
