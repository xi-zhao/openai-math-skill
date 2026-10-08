# OpenAI Math Skill

[中文](#中文) | [English](#english)

---

## 中文

### 这是什么

一个技能（工作流）文件 `SKILL.md`，适用于 Grok Bot、Cursor、Claude Code，或任何能读取 `SKILL.md` 技能文件的 agent。它帮助研究者用推理模型攻克数学、理论物理、理论计算机等证明/推导密集领域的研究问题和开放问题。

它重构的是 OpenAI 在 [openai/math](https://github.com/openai/math) 中发布 722 篇手稿时所体现的提示工程模式：依据是其中公开的提示摘录、推理摘要和审核附件，并参考了 First Proof 第二批报告和 OpenAI 此前的博客。

### 声明

- **本项目与 OpenAI 无任何关联**，未经 OpenAI 认可或背书。
- OpenAI 从未公布完整提示词。`SKILL.md` 中的模板以及流程默认值（N、审稿人数、辩论轮数、预算）都是我们根据公开摘录做的**重构或建议**，不是 OpenAI 原文，也不是其实际使用的 harness。逐字引用的原文只出现在 `SKILL.md` 的附录 A，并附出处。
- OpenAI 用的是未发布的内部模型。用公开模型时，产出率要大幅调低。

### 端到端做什么

1. **从模糊方向到候选题**：只给一个研究方向时，先做文献挖掘，交付 3–5 个**核实过出处**、确认仍未解决的候选问题，由研究者选题。
2. **只问关键问题**：每轮只问 2–3 个会改变命题的问题（术语消歧、方向、设定、完成标准、目标强度），其余用默认值补齐。
3. **签字确认题面**：写出自包含的 prove-or-refute（证明或否证）题面，经 14 项清单和歧义攻击，研究者明确签字后才开跑。
4. **best-of-N 求解**：多次彼此独立的求解运行，统一解析 `VERDICT` 行（PROVED / REFUTED / PARTIAL / NO_RESULT）。
5. **隔离审稿**：审稿者只看题面和成稿；有分歧时辩论；涉及计算时做 clean-room 复核。
6. **链式推进**：已审或已验证的结果作为前提，推向更强的目标。
7. **验证**：能形式化的走 Lean 4 + Mathlib（配合 comparator）；否则做计算或符号检查，并写明验证了什么、没验证什么。
8. **诚实的状态标签**：`PROPOSED` / `DISPUTED` / `REVIEWED(partial)` / `REVIEWED`，以及附加的 `VERIFIED(Lean | computation; scope=…)`。`REVIEWED(partial)` 只担保部分结果在其声称范围内经过审稿，不代表原题已解决。

工作流还会根据环境能力（能否开隔离会话、模型族数量、代码执行、Lean）选择 FULL / STANDARD / SOLO 模式，并设有预算与停止规则。

### 安装

把 `SKILL.md` 放进你所用 agent 的技能目录即可，例如：

- `.cursor/skills/openai-math-skill/SKILL.md`
- `~/.claude/skills/openai-math-skill/SKILL.md`

其他 agent 请按其技能/规则文件的约定放置。

### 状态

- 当前版本：**v6 快照**（改动见 [CHANGELOG.md](CHANGELOG.md)）。仍在通过真实端到端测试轮次打磨，不是最终成熟版。
- 已核实：comparator 验收通过含 REFUTED 形式的诚实 3 定理挑战，并拒绝 sorry、私有公理、`native_decide` 与陈述漂移；新模糊主题上的候选题引文全部真实。
- v2 经过了我们自己的审查和一次独立的 Codex 审查；v3 根据一次**真实的端到端试跑**修订（2026-10-08，单模型族档位：Codex；结果为 NO_RESULT/PARTIAL：4 次求解都是 PARTIAL，原命题未解决；Lean 与 comparator 只在一个子引理上跑通）。v4–v6 在后续端到端打磨轮次中修订。
- `SKILL.md` 的 Lean 一节写明了哪些命令在试跑中实际验证过、哪些没有实测。这只是数据点，不代表成功率。

### 参考

- openai/math：https://github.com/openai/math
- leanprover/comparator：https://github.com/leanprover/comparator

---

## English

### What it is

A skill (workflow) file, `SKILL.md`, for Grok Bot, Cursor, Claude Code, or any agent that reads `SKILL.md` skill files. It helps a researcher use reasoning models to attack research questions and open problems in mathematics, theoretical physics, CS theory, and other proof- or derivation-heavy fields.

It reconstructs the prompt-engineering pattern behind OpenAI's [openai/math](https://github.com/openai/math) release of 722 manuscripts, based on the public prompt excerpts, reasoning summaries, and verification artifacts there, plus the second First Proof report and earlier OpenAI blog posts.

### Disclaimer

- **This project is NOT affiliated with OpenAI** and is not endorsed by OpenAI.
- OpenAI never published its full prompts. The templates and workflow defaults in `SKILL.md` (N, number of referees, debate rounds, budgets) are **our reconstructions or suggestions** based on the public excerpts. They are not OpenAI's original text or its actual harness. Verbatim quotes appear only in Appendix A of `SKILL.md`, with sources.
- OpenAI used unreleased internal models. Expect a much lower success rate with public models.

### What it does, end to end

1. **From a vague topic to sourced candidates**: given only a research direction, it mines the literature and delivers 3–5 candidate problems with **verified sources**, checked to still be open, for the researcher to pick from.
2. **Only the key questions**: 2–3 questions per round, limited to ones that change the statement (terminology, direction, setting, resolution criteria, target strength); everything else gets a stated default.
3. **Sign-off on the statement**: a self-contained prove-or-refute statement, checked against a 14-item checklist and an ambiguity attack, with explicit researcher sign-off before any solving runs.
4. **Best-of-N solving**: several independent solver runs, with a strictly parsed `VERDICT` line (PROVED / REFUTED / PARTIAL / NO_RESULT).
5. **Isolated review**: referees see only the statement and the writeup; disagreements go to debate; computational claims get a clean-room re-check.
6. **Chaining**: reviewed or verified results become premises for the next, stronger target.
7. **Verification**: Lean 4 + Mathlib (with comparator) when formalizable; otherwise computational or symbolic checks, each stating what it verifies and what it does not.
8. **Honest status labels**: `PROPOSED` / `DISPUTED` / `REVIEWED(partial)` / `REVIEWED`, plus an optional `VERIFIED(Lean | computation; scope=…)`. `REVIEWED(partial)` means a partial result passed review within its claimed scope; it says nothing about the original problem.

The workflow also picks a FULL / STANDARD / SOLO mode from the environment's capabilities (isolated sessions, number of model families, code execution, Lean), and has budget and stopping rules.

### Install

Drop `SKILL.md` into your agent's skills folder, for example:

- `.cursor/skills/openai-math-skill/SKILL.md`
- `~/.claude/skills/openai-math-skill/SKILL.md`

For other agents, follow their own convention for skill or rule files.

### Status

- Current version: **v6 snapshot** (see [CHANGELOG.md](CHANGELOG.md)). Still being polished through real end-to-end test rounds; not the final mature version.
- Verified so far: comparator acceptance passes an honest 3-theorem challenge including the REFUTED form, and rejects sorry, a private axiom, `native_decide`, and statement drift; candidate citations on a new vague topic were all real.
- v2 went through our own audit plus an independent Codex review; v3 was revised after a **real end-to-end trial run** (2026-10-08, single-model tier: Codex; result NO_RESULT/PARTIAL: all 4 solver runs returned PARTIAL and the original problem stayed unresolved; Lean and comparator ran on one sub-lemma). v4–v6 were revised in later end-to-end polishing rounds.
- The Lean section of `SKILL.md` says which commands were actually exercised in those trials and which are untested. These are data points, not a success rate.

### References

- openai/math: https://github.com/openai/math
- leanprover/comparator: https://github.com/leanprover/comparator
