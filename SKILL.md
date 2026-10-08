---
name: "OpenAI Math Skill"
description: "Use when a researcher gives a research topic, a rough question, or a precise open problem in mathematics, theoretical physics, CS theory, or another proof/derivation-heavy field and wants reasoning models to attack it the openai/math way: verified candidate problems from a vague topic, self-contained prove-or-refute statements, best-of-N solving, isolated review and debate, chaining, Lean or computational verification, and honest status labels. 中文触发：用 AI 做数学/理论物理/理论计算机研究、开放问题、猜想、证明或否证、写题面、best-of-N、AI 审稿、Lean 形式化、只有一个研究方向想找题。"
---

# OpenAI 式研究工作流（openai/math 模式）v3

> **定位**：本技能把 openai/math（2026-10 发布，722 篇手稿）公开的提示摘录、推理摘要和审核附件，加上 First Proof 第二批报告和 OpenAI 前作博客，整理成**与工具无关、可降级**的工作流。
> **声明**：
> - §T 的模板以及流程默认值（N、审稿人数、辩论轮数、预算、超时）都是本技能的**重构或建议**，不是 OpenAI 原文或其实际 harness；逐字原文只出现在附录 A。
> - OpenAI 用的是未发布的内部模型。公开模型的产出率要**大幅调低**：First Proof 预跑中，公开模型在 30 分钟超时下，十题**一题也没解出**（附录 A-9）。
> - v3 根据一次真实端到端试跑（2026-10-08，STANDARD 模式，单模型族 Codex；结果 4/4 PARTIAL，原命题无结果；Lean + comparator 只在一个子引理上跑通）修订，摘要见附录 B。凡写"**试跑已验证**"的命令都在那次试跑中实际执行过；写"**未实测**"的只是建议。

## 0. 快速开始（先读这一节）
1. **判定输入**（§3 I-3）：只有方向 → 步骤 1 找候选；粗略问题 → 1 个形式化 + 1–2 个变体；精确陈述 → 直接步骤 2。
2. **环境自检**（§1）：确定 FULL / STANDARD / SOLO 模式，按 §1.1 实现隔离并做一次冒烟检查。
3. **定预算**（§2）：研究者没给就用分阶段默认值，**花费上限必须问一句**。
4. **步骤 1**：交付 3–5 个**核实过出处**的候选 → 【停点 H1：研究者选题】。
5. **步骤 2**：按清单 (f) 写 `input.vN.md`（**只放题面**，模板 (a)；必须含检索政策），备好 N 个提示变体（可选 (b)），用模板 (i) 做歧义攻击 → 【停点 H2：签字】。
6. **步骤 3**：best-of-N 求解。每次发送 = (a0) 求解前言 + `input.vN.md` + 〔(b)〕 + (a1) 输出格式；解析 VERDICT 行。
7. **步骤 4**：隔离审稿（模板 (d)，审稿者只拿题面 + 成稿，给两行判词），只有作者**未声明**的问题才辩论（g2），涉及计算就做 clean-room 复核（h）。
8. **步骤 5–7**：链式推进（c）→ 验证（6A Lean 走 e1/e2 并经【H3】；6B 做计算或符号检查）→ 意义筛选与成稿。
9. 全程按 §2 的停止规则（每条规则触发后做什么见 §2 表），只在【H4 里程碑】时汇报。所有产物放进 `research/<id>/`（§9）。

## 何时使用 / 不用
- **用**：结论能写成精确陈述（证明 / 否证 / 给出界 / 推导），且研究者愿意投入审稿和验证。
- **不用**：纯经验问题；只想要综述；无法写成可判定陈述、研究者又不愿意收窄的问题。
- 理论物理推导题：把 "Prove or refute" 改成 "Derive or refute"，`DERIVED` 等同于 PROVED。

## 核心原则
1. **杠杆在题面**。First Proof 上 OpenAI 员工用的求解提示只有 6 句；openai/math 的题面可长达数千字符，几乎都在消歧和堵捷径。
2. **双向开放**：prove **or** refute，两边都写清完成标准。
3. **防偷换**：特例、弱化、单一方法失败都**不算**解决，要写进题面。
4. **信息隔离**：按 §1.1 的可执行定义隔离；审稿者只看题面和成稿；计算复核从规格出发重写，先冻结再比对。
5. **诚实分级**：标签按 §7 打（`PROPOSED / DISPUTED / REVIEWED(partial) / REVIEWED` + 可选 `VERIFIED(…)`）；每项验证都写明"验证了什么、没验证什么"。
6. **公布分母**：完整记录题单、尝试次数和失败。

---

## 1. 环境自检与运行模式（开工前做一次，写进 `intake.md`）
先探测 6 项能力：① 能否开**隔离会话**（定义见 §1.1；同一上下文"换个角色"**不算**）；② 可用的**模型族**数量；③ 能否**执行代码**（含 CAS，如 SymPy/mpmath）；④ 有无 **Lean 4 + Mathlib**；⑤ 有无 **comparator**（Linux + landrun + lean4export，见 6A）；⑥ 能否**联网核对文献**。

| 模式 | 条件 | best-of-N | 审稿 | 标签上限 |
|---|---|---|---|---|
| FULL | 能隔离，且有 ≥2 个模型族 | 各次在独立会话里跑，可混用模型 | 2 名审稿者，至少 1 名来自另一模型族 | VERIFIED |
| STANDARD | 能隔离，只有 1 个模型族 | 独立会话 | 2 个独立会话（同族）；标签注明 "AI 2× same family" | VERIFIED |
| SOLO | 不能隔离（只有你自己） | 在同一上下文里串行尝试，并记录"非独立" | **不能自审**：生成"审稿包"（`input.vN.md` + 成稿 + 模板 (d)）交研究者到别处跑 | **PROPOSED** |

- 无代码执行：跳过 (h) 和数值检查，记为 `NOT_RUN`（不得写成通过）。无联网：步骤 1/7 的文献核查交给研究者，引用标"未核"，**不得进入 H2**。
- 无 Lean（或装不起）：交付未执行的 Lean 陈述草稿（Lean 记为 `STATEMENT_ONLY`）+ 6B 检查；Lean 永远不写成通过。
- **数据外发**：受理时问"题目和未发表的思路能否发给第三方模型 API？"，优先用零数据保留接口（First Proof 预跑即用 Zero Data Retention）。
- **思考档位**：取该模型**最高的单 agent 档位**。会自动委派子 agent 的档位（会破坏隔离、打乱 token 统计）只有在记录档位名、并保证子 agent 同样满足 §1.1 时才可用。档位名逐字记进 `runs.csv` 的 `thinking_level`。

### 1.1 隔离会话的可执行定义
一个会话算"隔离"，必须**同时**满足：
1. **无历史**：全新会话；不带对话历史、记忆、用户级配置或技能；看不到该工具保存的以往会话日志（例如 CLI 的 sessions/log 目录）。
2. **文件可见范围受限**：只能读下表"可读"一栏列出的内容；其他运行的输出、审稿、`intake.md`/`decisions.md`、审计稿或笔记（常含解题思路）、研究包以外的方向提示一律不可见。**不带工具的单次 API 调用**天然满足本条。
3. **联网按题面**：求解会话能否联网检索，由题面 PERMITTED INPUTS 里的 **SEARCH POLICY** 决定（清单 (f) 第 12 项，必填）。拦不住时（CLI 本身要联网调模型），事后检查事件日志里的检索动作，记入 `runs.csv` 的 `web_used`。
4. **单 agent**：一个会话不得把任务委派给不满足 1–3 的子 agent。

| 角色 | 可读 | 不可读 |
|---|---|---|
| 求解 (a)/(c) | 组装好的提示（经 stdin 传入）、空的 scratch 目录；SEARCH POLICY 允许的网络资源 | 其他运行、审稿、intake/decisions、审计稿、会话日志 |
| 歧义攻击 (i) | `input.vN.md` | 任何成稿、求解前言、研究包 |
| 审稿 (d) | `input.vN.md` + 成稿（+ FOCUS）；所引一手文献 | 作者推理/事件日志、代码、其他稿件、其他审稿 |
| 提炼 (g1) | `input.vN.md` + 一份或一组同质成稿 | 审稿意见 |
| 辩论 (g2) | `input.vN.md` + 成稿 + 审稿要点 | 其他稿件 |
| 计算复核 (h) | `input.vN.md` + `spec.md` 指定行 | 提交代码、旧结果、审稿 |
| Lean (e1) | 待形式化陈述的摘录 + 版本；可选只读 Lean 项目 | 成稿其余部分、审稿 |
| Lean (e2) | 开发目录（可写）+ Lean 工具链（只读） | 验收目录、Mathlib 本地缓存、其他产物 |

**最低实现**（在共享机器上用 CLI 时）：容器，或 bubblewrap 把整个家目录和研究项目根目录挂成空 tmpfs，只放回模型认证目录和 CLI 二进制，提示从外部经 stdin 送入。试跑已验证的形式（Codex CLI；各目录换成你自己的）：
```bash
bwrap --ro-bind / / --dev /dev --proc /proc \
  --tmpfs /tmp --tmpfs "$PROJECT_ROOT" --tmpfs "$HOME" \
  --ro-bind "$CLI_DIR" "$CLI_DIR" --bind "$HOME/.codex" "$HOME/.codex" \
  --tmpfs "$HOME/.codex/sessions" --tmpfs "$HOME/.codex/log" \
  --bind "$SCRATCH" /tmp/scratch --chdir /tmp/scratch --setenv HOME "$HOME" \
  "$CLI_DIR/codex" exec --ephemeral --skip-git-repo-check --ignore-user-config \
    -m "$MODEL" -c model_reasoning_effort='"<最高单 agent 档位>"' \
    -s workspace-write -C /tmp/scratch --json -o /tmp/scratch/last_message.md - \
    < prompt.txt > events.jsonl 2> stderr.log
```
(e2) 用同一包装，再 `--ro-bind` Lean 工具链目录、`--bind` 开发目录（并把工作目录设为开发目录）。开跑前做 1 次**冒烟调用**（计入预算），确认会话里看不到项目目录和会话日志。做不到第 2 条（文件系统藏不住），又不能保证可读范围内没有上述文件时，按 SOLO 处理。

## 2. 预算与停止规则
**计数口径**：一次"模型调用" = 一个模型会话或一次独立调用（一次 `codex exec`、一个子 agent 运行、一次 API 请求链），不论内部用了多少工具步骤。冒烟测试、(i)、(g1)、(d)、(g2)、(e1)、(e2)、(h) **全部计入**。

**默认预算**（研究者未指定时写进 `intake.md`，可覆盖；均为本技能建议）：
| 阶段 | 内容 | 默认上限 |
|---|---|---|
| 准备 | 冒烟 + (i) 歧义攻击 | 3 次 |
| 求解 | best-of-N | N=4（SOLO N=2） |
| 审稿 | (g1)、(d)、(g2)、复审 | 8 次 |
| 验证 | (e1)、(e2)、(h)、比对脚本 | 5 次 |
| 合计 | | ≤20 次，墙钟 ≤24 h |
- 每份 PROVED/REFUTED 2 份审稿；辩论 ≤2 轮。PROVED 与 REFUTED 同时出现（两份都要送审，各自还可能辩论）时，审稿阶段 8 次通常不够：先按 H4 汇报，经研究者批准再追加。
- **单次超时**（建议）：求解 60 min，其余 30 min；超时记 `TIMEOUT`。依据：试跑中 `max` 档单次求解 20–29 min；First Proof 预跑用 30 min 超时，对这类题偏紧。
- **花费上限必须问**，没有回答就只按调用次数上限执行。长时调用放后台或异步执行，避免撞上工具超时。

**量级参考**：
- First Proof 报告 Table 4：单次 xhigh 的 ChatGPT 5.5 Pro 调用每题约 US$8–16（10 题合计 $117，墙钟 5.8 h），多 agent 编排系统每题 $36–951；openai/math README 称每个结果平均用 "three hours of ChatGPT Pro thinking compute"。
- 本技能试跑（一个数据点，附录 B）：STANDARD 模式、一道题，墙钟约 61 min，14 次模型调用，input 约 3.06M tokens（其中缓存命中约 2.46M），output 约 187K tokens；订阅登录方式拿不到美元成本，只记了 token。

**停止规则与之后的动作**：
| 规则 | 触发 | 触发后立即做 | 之后 |
|---|---|---|---|
| **S1** | N 次全为 NO_RESULT 或 PARTIAL | **不再加跑求解**。有 PARTIAL 时按下方"S1 收尾"做完；全部 NO_RESULT 时跳过收尾 | H4 汇报三个选项，**等研究者选**，期间不再调用模型 |
| **S2** | 出现 `REVIEWED` 结果（完整 PROVED/REFUTED；`REVIEWED(partial)` **不触发**） | 不再开新的求解；已在跑的让它结束并记日志 | 转步骤 6 验证，再到步骤 7 |
| **S3** | 辩论 2 轮仍有分歧 | 标 `DISPUTED`，H4 汇报 | 选项：交人类专家 / 经批准再辩一轮 / 归档。研究者不回复就停在 `DISPUTED` |
| **S4** | 预算用到 80% / 100% | 80%：预警汇报，继续；100%：不再发起新调用，在跑的调用结束后记日志，未完成的审稿、验证标 `NOT_RUN` | 按当前标签汇报；没有完整结果时附 S1 的三个选项；只有研究者明确批准才追加预算 |
| **S5** | 任何运行暴露题面歧义或漏洞 | 暂停，修题面，**重新签字**（H2） | 修改前的运行作废，留在日志里并单列分母；已用的调用照算预算；用新版本从步骤 3 重来 |
| **链式上限** | 接续深度 >3，或歧义修订 >3 轮 | 停止 | H4 汇报，由研究者决定 |

**S1 收尾**（S1 触发且至少有 1 份 PARTIAL 时固定执行，受审稿、验证阶段预算约束）：
1. **(g1) 提炼**：同质的 PARTIAL（同一路线、同一缺口）**只跑 1 次**，把全部同质稿件一起附上；路线不同的各跑 1 次。副产品一律先标 PROPOSED。
2. **审稿**：挑 1 份最佳 PARTIAL（证明内容最多、最容易核对），送 2 份隔离审稿 (d)。只有 `CLAIMED-SCOPE VERDICT` 决定标签（§7）；作者自己声明的缺口**不辩论**；只有作者未声明的 MAJOR/WRONG 才走 (g2)。预算只够 1 份时照做，标签停在 PROPOSED。
3. **验证（可选）**：可形式化或可计算的副产品走 6A/6B。
4. **汇报**（I-5）给三个选项：
   - ① **用 PARTIAL 做 (c) 接续**：`REVIEWED(partial)` 的结论（只限其声称范围）可作 PREMISES，其余放 PRIOR WORK；新目标作为子项目 `<id>.c1`，走简化签字（H2）。
   - ② **弱化变体**：写新的 `input.v(N+1).md`，**完整重新签字**；旧运行留在日志里，分母分开公布。
   - ③ **归档为失败**：写 `ARCHIVE.md`（首页状态行、全部运行、带标签的部分进展、已排除的路线、剩余难点、分母），然后停止，不再调用模型。

---

## 3. 交互协议
### I-1 先做功课再提问
不要用"你想研究什么具体问题？"开场。先完成步骤 1，交付 3–5 个候选，字段见 §9 的 `candidates.csv`。每个候选至少包含：精确陈述、核实过的出处、意义、难度及其依据。

### I-2 只问会改变命题的问题（每轮 ≤3 个）
值得问的只有 5 类：
1. **术语 / 方向消歧**：例如"下界"指资源下界（即参数上界）还是构造下界；
2. **方向**：证明 / 否证 / 都行；
3. **设定与允许的假设**；
4. **完成标准与反例标准**；
5. **目标强度**：完整结论、改进常数，还是部分结果也可接受。

其余一律用默认值补齐，并告诉研究者可以覆盖。默认值如下：
- 方向 = either；
- 设定 = 文献中最常用的版本；
- 部分结果 = 记录但不算完成；
- 终稿机算 = 证明型题"不鼓励"，存在性、构造型或计算型题"允许，但须给可重放证书"；
- 检索政策 = 允许检索，但必须列出所查来源（题面里写明；见 (f) 第 12 项）；
- N 和预算 = 见 §2；
- 验证 = 按 6A 分诊。

### I-3 模糊度阶梯
| 输入 | 处理 |
|---|---|
| 只有领域或关键词 | 步骤 1，交付 3–5 个候选 → H1 |
| 粗略问题（"X 能否推广到 Y"） | 1 个精确形式化 + 1–2 个变体（更弱 / 更强），说明变体之间的蕴含关系 |
| 精确陈述 | 直接进入步骤 2，只就清单暴露的缺口提问 |

### I-4 人工停点（只有这 4 个）
- **H1 选题**：研究者选一个候选，或合并、修改。
- **H2 签字**：展示 `input.vN.md` 全文（只含题面）、**N 个提示变体**（各自的 (a0)/(b) 组合，或注明"N 份相同，diversity=none"）、14 项清单结果（逐项写"通过 / 默认 / 有意不写"）、歧义攻击发现的问题及处理、**标记假设**、运行模式与隔离实现、预算。必须得到明确确认（如"确认""OK 开跑"），沉默或含糊的回复不算。
- **H3 Lean 陈述核对**：只在走 6A 时出现。研究者（或其指定的专家）逐字核对 Defs 和 Challenge 文件，记 `H3: HUMAN`。研究者不在时可由调度 agent 代核，记 `H3: AGENT_ONLY`：Lean 维度写 `VERIFIED(strict; H3 pending; …)`，研究者补核后改为 `H3 human`。H3 pending 的 claim 不得作为无条件前提进入链式推进。
- **H4 里程碑**：候选结果通过首轮审稿（定义见 §7）、审稿分歧或致命缺陷、需要修改题面、触发 S1/S3/S4。

### I-5 汇报模板
```text
【状态报告】项目 {{id}} ｜题面 v{{n}} ｜模式 {{FULL/STANDARD/SOLO}} ｜{{时间}}
触发：{{H4 子项 / S1–S5}}
进度：运行 {{k}}/{{N}}（diversity {{none/变体数}}）；PROVED {{a}} · REFUTED {{b}} · PARTIAL {{c}} · NO_RESULT {{d}}
最佳结果：{{一句话精确陈述}}  标签：{{PROPOSED / DISPUTED / REVIEWED(partial) / REVIEWED + 可选 VERIFIED(…)}}
原命题：{{已解决 / 未解决}}
审稿：R1 {{CLAIMED-SCOPE / vs-ASSERTION}}，R2 {{…}}；作者未声明的关键问题 ≤3 条：{{…}}
验证：{{claim_id: Lean/计算状态（范围）}}；未验证：{{…}}
预算：已用 {{调用数/花费/时长}}（{{%}}），分阶段 {{准备/求解/审稿/验证}}
需要你决定：{{是/否；如需决定，给 2–3 个选项及我的建议}}
```

### I-6 限度
- 澄清**最多 2 轮**。之后按最佳猜测继续，把自行做出的假设列为**标记假设**，写进 H2 材料。
- **绝不默默弱化命题**。任何弱化都要作为显式变体提出，并重新签字。

### I-7 示例（出处用〈〉占位，实际使用时必须真实、核实过）
```text
研究者：我想用 AI 做量子纠错码下界方向的研究。
助手：给你 4 个候选（另 1 题 2026 年已被声称解决，已移出，可改作复核任务）：① 〈陈述〉出处〈作者, 标题, arXiv 号, 节号〉；
 难度 中–高（依据…）… 问题：(1)"下界"指资源下界还是构造下界？（默认：都可）(2) 方向？（默认：都行）
研究者：选①。 助手：〈input.v1.md + 提示变体 + 14 项结果 + 标记假设 + 模式 + 预算〉请回复"确认"后开跑。
```

---

## 4. 流程
### 步骤 0：受理
`intake.md` 写 8 项：① 目标命题或候选；② 领域约定（记号、归一化、"标准定义"的具体版本）；③ 已知文献（最好结果、反例、已知障碍）；④ 成功标准；⑤ 验证途径；⑥ 预算（§2）；⑦ 运行模式与隔离实现（§1、§1.1，含冒烟结果）；⑧ 数据外发许可。缺项按 I-2 补齐。

### 步骤 1：文献与问题挖掘（带出处）
- **来源**：
  - 综述末尾的 open questions；
  - 论文中的 Question / Conjecture / Problem 环境，以及 "we expect" / "it would be interesting" 之类的段落；
  - 领域问题库或参数表（例如编码理论的 codetables.de、Erdős 问题库、MathOverflow）；
  - 近期 arXiv（API 关键词检索，按日期排序）。
- **是否已解决**（必做）：
  - 对每个候选做**前向引用检索**，例如 Semantic Scholar 的 `/graph/v1/paper/arXiv:<id>/citations`，或 Google Scholar 的 "cited by"；
  - 再搜一次"标题关键词 + resolved / proof / counterexample"；
  - 分类为 `OPEN`、`CLAIMED-RESOLVED`（有未审的声称证明）或 `RESOLVED`。只有 `OPEN` 能进开放题单；`CLAIMED-RESOLVED` 可以作为"复核任务"提供，用模板 (d) 审那份证明。
- **出处必须可打开**：模型给的文献一律当作待核。逐条打开，确认文献存在且**确实包含**所称的陈述。查不到就删，不要修补。
- 保留**完整题单和淘汰理由**。

### 步骤 2：写自包含题面
- **题面与发送提示分开**：`input.vN.md` 只含模板 (a) 的题面（从 `# Problem` 到 PERMITTED INPUTS，含 SEARCH POLICY）。求解前言 (a0)、研究包 (b)、输出格式 (a1) 在发送时拼接，**不写进** `input.vN.md`。歧义攻击 (i)、审稿 (d)、复核 (h) 只拿到 `input.vN.md`。
- 按清单 (f) 逐项写。需要给方向时写 (b)；(b) 的不同写法、"不给方向"、等价表述都可以作为 best-of-N 的**提示变体**，在 H2 一并签字。只有一份提示时，在 `runs.csv` 记 `diversity=none`，并在汇报里说明：N 次运行此时主要反映采样差异（试跑中 4 份相同提示走了同一条路线、停在同一个缺口）。
- **渐近型命题**（"存在常数…""对所有充分大的 n…"）：反面完成标准必须要求**带证明的无限族**，写明"有限个对象不构成反例"。
- **防恶意顺从**：凡是"存在一个构造"的题，都写明规模、显式性、可构造性要求，并定义 "explicit" 指什么（例如"一个确定性算法，对每个输入 j 终止并输出该对象"）。
- 用模板 (i) 在**隔离会话**里做歧义攻击和弱化攻击，反复修改到只剩一种合理解读；SOLO 模式下自己逐条做，并在 H2 材料里注明。
- 发送前检查：没有残留的 `{{`；版本号已写进文件名（`input.v1.md`）。

### 步骤 3：求解（best-of-N）
- 用当前可用的最强推理模型、最高的单 agent 思考档位（§1），每次运行彼此不可见（§1.1）。按 H2 签好的提示变体轮换。
- **发送内容** = (a0) + `input.vN.md` + 〔该变体的 (b)〕 + (a1)，逐字存进 `runs/<run_id>/prompt.txt`。
- **解析**：用正则 `^VERDICT: (PROVED|REFUTED|PARTIAL|NO_RESULT)\s*$` 匹配首个和最后一个非空行，两行必须完全相同。缺失、首末不一致或输出被截断，运行状态记为 `INVALID_OUTPUT`，不得从正文猜测。运行状态共 5 种：`OK / TIMEOUT / TOOL_ERROR / INVALID_OUTPUT / BUDGET_EXHAUSTED`。PARTIAL 与 NO_RESULT 的含义见 (a1)。
- 对每份 PROVED 或 REFUTED 成稿跑 (g1)；PARTIAL 成稿按 §2"S1 收尾"第 1 条（同质的只跑 1 次）。提炼出的副产品一律标 PROPOSED，另行送审。
- 结论相反的两份稿件**都送审**，至少有一份是错的。
- **日志**（`runs.csv`）：`problem_id, statement_version, template, prompt_variant, diversity, model, mode, isolation, run_index, thinking_level, start, wall_time, timeout, tokens_in, tokens_cached, tokens_out, cost, web_used, run_status, verdict, output_path, parent_claims`；每次运行**实际发送的完整提示词**、原始输出和事件日志存进 `runs/<run_id>/`。

### 步骤 4：独立审稿（信息隔离）
- **证明审稿**：用模板 (d)。审稿者只拿到冻结的 `input.vN.md` 和成稿（§1.1），看不到作者推理、代码或其他审稿意见。(d) 输出两行判词：`CLAIMED-SCOPE VERDICT`（只判作者声称证明的部分；作者自己声明的缺口不算错误）和 `vs-ASSERTION VERDICT`（作为原题解答来判）。两行都用正则解析，缺失记 `INVALID_OUTPUT`。第二轮可以加 `FOCUS`，做定向审稿。
- **辩论**：只有 `CLAIMED-SCOPE VERDICT` 为 MAJOR_GAPS 或 WRONG（即作者**未声明**的问题）才用 (g2) 写回应 → 由**新的**审稿会话复核，最多 2 轮，仍不过就标 DISPUTED（S3）。PROVED/REFUTED 成稿的声称范围就是原命题，所以照常辩论；PARTIAL 成稿的 `vs-ASSERTION VERDICT` 通常是 MAJOR_GAPS，只记录、不辩论。作者在辩论中把结论降为 PARTIAL 时，按新的声称范围对修订稿重新送审。
- **计算复核**：成稿依赖计算时用模板 (h)。复核者可读冻结题面和 `spec.md`（由 agent 从成稿逐字摘出的定义、公式、表格，带行号），不读代码、结果和审稿。冻结、哈希、运行、比对都由 agent 用工具完成（`sha256sum`、`chmod -w`），不能让模型报哈希。**比对失败**：先分类（执行错误 / 规格歧义 / 复核实现 bug / 原结果错误）；冻结代码不得原地改，修订版另行冻结为 v2，并注明"已看过差异"，不再算首次盲复核；原因查明前，相关 claim 的计算项记为 `FAILED`，按 H4 汇报。
- **证据分档**：源文阅读、文献核查、辩论、计算执行、终审各存一份记录。"审过了"本身不能当证明前提。
- **终审绑定终稿**：成稿修改后，必须对最终源文件做一次全新的整篇审稿。

### 步骤 5：链式推进
- 已审或已验证的结果作为**给定前提**，用模板 (c) 提出下一个更强的目标。前提要**逐字贴入**，并注明验证状态。`REVIEWED(partial)` 的结论只在其声称范围内可作前提。
- 未验证的前提放进 PRIOR WORK，**只复用技术，不复用结论**；非用不可时，新结论写成条件式"若 P1 则 Q"。
- **按 claim 复用**：只有 claim 的全部证明义务都被审稿或形式化覆盖，才能当无条件前提；只验证了局部的（如计算项、H3 pending 的 Lean 项）不行。前提被撤回时，依赖它的结果（`parent_claims`）全部降级并重审。

### 步骤 6：验证
**6A 分诊**：满足以下三条才走 Lean，否则走 6B：
1. 所需定义 Mathlib 都有，或者很少（先查 Mathlib 文档或源码）；
2. 陈述能在约 1 小时内写成 Lean 签名；
3. 环境有 Lean（elan、工具链和 Mathlib 缓存要数 GB；试跑中 `lake exe cache get` 下载 8908 个文件约 1.5 min）。
原命题不可形式化时，可以只形式化成稿里可形式化的**子引理或副产品**（试跑即如此），Scope 里写明。

**6A Lean 4 + Mathlib + comparator**

**6A-1 版本与工具**（试跑已验证：Linux x86_64，非 root 用户）
- **Lean / Mathlib**：选一个 Mathlib release tag，toolchain 用同号 Lean。试跑：`leanprover/lean4:v4.34.1` + Mathlib `d13f23b723b8…`（tag v4.34.1，与 openai/math 的 lake-manifest 一致）。安装 elan 后 `lake update && lake exe cache get`。
- **comparator 与 lean4export**：优先用与项目 Lean 版本同号的 tag。**没有同号 tag 时**取 ≤ 项目版本的最近 tag，把它的 `lean-toolchain` 改成项目版本再编译，并在 Scope 里记下 commit 和"toolchain 已改"。lean4export 由 comparator 的 `lake-manifest.json` 锁定，随 comparator 一起编译。试跑（v4.34.1 时两者都没有同号 tag）：
  ```bash
  git clone https://github.com/leanprover/comparator && cd comparator
  git checkout v4.34.0                          # = d03acab
  echo 'leanprover/lean4:v4.34.1' > lean-toolchain
  lake build comparator lean4export             # lean4export 锁定 076e8e5 = 其 tag v4.34.0
  # 产物：.lake/build/bin/comparator
  #       .lake/packages/lean4export/.lake/build/bin/lean4export
  ```
- **landrun**（从 main 分支源码编译，需 Go；试跑用 go 1.24.4，main 811cfff，自报 0.1.18）：
  ```bash
  git clone https://github.com/Zouuup/landrun && cd landrun && go build -o landrun ./cmd/landrun
  ./landrun --rox /usr --ro / -- /bin/true      # Landlock ABI 自检
  ```
  自检报 `missing kernel Landlock support. Got Landlock ABI vK, wanted {Landlock V9…}` 时，说明内核 Landlock 版本低于要求。comparator 调用 landrun 时自带 `--best-effort`，仍能运行，但沙箱只有 ABI vK 能提供的限制：Scope 里必须写 "landrun best-effort on Landlock ABI vK"。试跑在 ABI v6 上即如此。
- 把三个二进制放进 `PATH`，或用 `COMPARATOR_LANDRUN` / `COMPARATOR_LEAN4EXPORT` 指定完整路径。
- **冒烟测试**（建议在真题前做，试跑已验证）：一个不依赖 Mathlib 的小项目（Defs 定义一个函数，Challenge 一条 `sorry` 定理），正例 Solution 应 `EXIT=0`；Solution 换成 `sorry` 应报 `Illegal axiom detected: 'sorryAx'` 且退出码非 0。

**6A-2 项目布局**（受信任部分由人或调度 agent 生成，不由求解 agent 生成；试跑已验证）
```text
lean-toolchain            leanprover/lean4:<项目版本>
lakefile.toml             name、defaultTargets = ["Defs", "Challenges", "Solution"]；
                          [[require]] mathlib，git + rev 固定到 commit；[[lean_lib]] Defs / Challenges / Solution
lake-manifest.json
Defs.lean                 只 import Mathlib；全部自定义定义（反例对象也放这里）
Challenges/<Name>.lean    只 import Defs；每条待证定理一条 theorem，证明为 sorry
Challenges/<Name>.json    comparator 配置
Solution.lean             import Defs，绝不 import Challenges；定理名与陈述逐字相同
Axioms.lean               验收后由调度 agent 写
```
comparator 配置支持**多条定理**（试跑用了两条）：
```json
{"challenge_module": "Challenges.<Name>", "solution_module": "Solution",
 "theorem_names": ["<NS>.<thm1>", "<NS>.<thm2>"], "definition_names": [],
 "permitted_axioms": ["propext", "Quot.sound", "Classical.choice"]}
```
openai/math 的配置另含 `"enable_nanoda": false`（A-7）；试跑省略该键，comparator d03acab 正常运行。

**6A-3 陈述**：用 (e1) 写 Defs 与 Challenge，**按成稿结论写**，每条 claim 一个全限定名：PROVED 的 claim 写原陈述；REFUTED 的写 `¬ Statement`，或写成满足反面完成标准的具体反例（反例对象放 Defs）。在开发目录里 `lake build Defs Challenges`，只允许出现 `sorry` 警告 → 【H3 核对】，记下受信任文件（toolchain、lakefile、manifest、Defs、Challenges/*）的 sha256。只有题面本身要求"判定真假"时才用 `Prop` 定义洞（`definition_names` 非空），而且 hole 的取值必须额外人工审查，不能只凭 comparator 通过就标 VERIFIED（附录 A-7）。

**6A-4 证明（开发目录）**：用 (e2) 在 Lean jail 里写 `Solution.lean`（§1.1：只暴露开发目录可写、工具链只读），模型自己用 `lake build Solution` 编译。**严格路径**（白名单只有 propext / Quot.sound / Classical.choice）禁止 `sorry`、`admit`、新 `axiom`、`native_decide`、`implemented_by`、`extern`、`debug.skipKernelTC`。`native_decide` 会引入 `Lean.ofReduceBool` 一类额外公理（以 `#print axioms` 实际输出为准），**不可能**通过 3 公理白名单。研究者明确批准时走**扩展路径**：把实际列出的额外公理加进 `permitted_axioms`，标为 `VERIFIED(Lean+native)`，并在 Scope 里写明扩大了哪些信任基础。(e2) 结束后，重新计算受信任文件的 sha256，与 H3 时的记录比对；不一致即作废。

**6A-5 验收（干净目录）**：开发目录对求解 agent 可写，里面的任何东西都不可信。
1. 新建验收目录，放入 H3 时留存的受信任文件（**从调度 agent 自己的副本复制，不从开发目录复制**）和开发目录里的 `Solution.lean` 源文件（以及 `Solution/` 下的源文件，如有），不带任何 `.lake`。
2. **依赖来源**：只能从受信任源重新获取（`lake update && lake exe cache get`），或从求解 agent 没有写权限的位置复制（例如 jail 里看不到的 Mathlib 本地缓存）。获取后核对 `lake-manifest.json` 的 sha256 没变，并确认 `.lake/build` 不存在。
3. **第一次编译只能由 comparator 完成**（它在 landrun 沙箱里构建两边）。以非 root 用户运行，有 systemd 用户会话时按 comparator README 包一层：
   `systemd-run --property=RestrictAddressFamilies=~AF_UNIX --user --pty -E PATH="$PATH" --working-directory "$(pwd)" -- bash -c 'lake env comparator Challenges/<Name>.json'`
   **没有 systemd 用户会话时**（试跑即如此）直接运行：
   `lake env comparator Challenges/<Name>.json > comparator.log 2>&1; echo "COMPARATOR_EXIT=$?" >> comparator.log`
   comparator README 说明该包装用于防范 landrun 的一个已知漏洞；去掉包装就少了这层防护，Scope 里必须注明 "no systemd-run wrapper"。（未实测的补救：在一次性容器或虚拟机里做验收。）
4. 完整日志存进 `comparator.log`；退出码非 0 即不通过。通过时日志含 "Lean default kernel accepts the solution"。

**6A-6 公理与扫描**：在验收目录新建 `Axioms.lean`：`import Solution`，对 `theorem_names` 中每条定理和引理表中每个声明写 `#print axioms <name>`，用 `lake env lean Axioms.lean > print_axioms.log` 跑出真实输出并存档（模型不得代填）。辅助扫描**只用于定位**，不作为验收依据，只扫 Solution 和 Defs、不扫 Challenge（注释里的 `sorry` 也会命中，要人工排除）：
`rg -n --no-ignore -g '*.lean' -e '\b(sorry|admit|native_decide|implemented_by|extern|unsafe|debug\.skipKernelTC|ofReduceBool)\b' -e '^\s*(@\[[^]]*\]\s*)?((private|protected|noncomputable)\s+)*(axiom|opaque)\b' $(ls -d Solution.lean Solution Defs.lean Defs 2>/dev/null)`

**6A-7 降级**：没有 comparator（非 Linux，或装不了 landrun）时，在干净目录里 `lake build` + 6A-6 + H3，标签注明 "Lean, no comparator"。

**6A-8 交付**：引理对照表、DEVIATIONS 清单、`comparator.log`、`#print axioms` 原始输出、Scope 文档：覆盖了哪些 claim、**没覆盖**什么（包括只作为假设出现的命题），以及信任基础：Lean 版本、Mathlib rev、comparator commit（是否改过 toolchain）、lean4export commit、landrun commit 与 Landlock ABI（是否 best-effort）、有无 systemd-run 包装、H3 是 HUMAN 还是 AGENT_ONLY。

**6A-9 实测范围**
- **2026-10-08 试跑已验证**：上述版本组合与编译命令；冒烟正例/反例；开发与验收目录分离、验收侧重新获取依赖；一个 Challenge 含两条定理（`theorem_names` 两项）、无定义洞、严格 3 公理路径；Solution 为单文件；comparator 在无 systemd 包装、landrun best-effort（ABI v6）下 `COMPARATOR_EXIT=0`；`#print axioms`；rg 扫描；H3 为 AGENT_ONLY。
- **未实测**：systemd-run 包装；Landlock ABI ≥ v9 上的完整沙箱；扩展路径（native_decide）；定义洞；REFUTED 方向（`¬ Statement` 或反例）的 Challenge；外部内核（nanoda 等）；多文件 `Solution/`；非 Linux 降级路径；(e1) 会话自己编译陈述（试跑中 (e1) 会话没有 Lean，由调度 agent 编译）；其他 Lean 版本；人工 H3。

**6B 暂不形式化**（物理推导、渐近分析、数值常数，或形式化成本超出预算）：CAS 独立复算关键恒等式、量纲和极限情形；区间算术或高精度数值检查（覆盖边界、退化、随机参数，并对已知特例回归）；可重放脚本 + 哈希清单，按 (h) 复核；每项检查都写 "verifies X, not Y"（附录 A-6）。
- **两档结果**：按 (h) 做的盲复核记 `PASSED(clean-room; …)`；调度 agent 自写的检查记 `PASSED(sanity; …)`。成稿结论依赖计算时必须做 clean-room，只有 clean-room 才能给 `VERIFIED(computation; …)`；sanity 只作辅助证据，不升级标签。

### 步骤 7：意义筛选与成稿
- **筛选**：是否真的新（再查一次文献和前向引用）？是否只是已知方法的直接套用？是否好得可疑（能推出已知为假的命题就回到步骤 4）？
- **成稿结构**：标题；**首页状态行**（§7）；主定理（精确陈述）；claims 表；与已知结果的关系；证明；**外部输入清单**（版本号和定理号）；验证说明（各项覆盖了什么）；局限；AI 使用声明（模型、提示、尝试次数、模式、隔离方式）。
- **引用核查**：逐条打开引文，确认含有所称的结论，检查撤稿和勘误；引用有缺陷的旧工作时要在正文明说；检查有无未标注出处的措辞沿用。

## 7. 状态标签与判定规则
- 用"**一个主标签** + 可选的 VERIFIED 范围"表示。主标签共 4 个：
  - `PROPOSED`：没有足够的隔离独立审稿（包括只做过自检、只有 1 份审稿、SOLO 模式）；
  - `DISPUTED`：审稿有分歧，或作者未声明的 MAJOR / WRONG 问题在辩论后仍未解决；
  - `REVIEWED(partial)`：作者结论为 PARTIAL（或是 (g1) 提炼出的副产品），≥2 份隔离审稿的 `CLAIMED-SCOPE VERDICT` 为 ACCEPT 或 MINOR_GAPS，minor 问题已修改，终稿做过整篇复审。只担保**声称范围内**的结论，对原命题不作任何担保，不触发 S2；
  - `REVIEWED`：作者结论为 PROVED/REFUTED，≥2 份隔离审稿的两行判词都是 ACCEPT 或 MINOR_GAPS，minor 问题已修改，且终稿做过整篇复审。
  - 两种 REVIEWED 都注明审稿来源，例如 `REVIEWED(AI 2× same family)`、`REVIEWED(partial; 1 human expert)`。
  - `VERIFIED(Lean | computation; scope=…)`：附加在主标签之后，范围之外的部分仍按主标签算。Lean 在 H3 只由 agent 核对时写 `VERIFIED(Lean; H3 pending; scope=…)`；computation 只有 clean-room（6B）才可附加。
- **审稿结论映射**（标签只看 `CLAIMED-SCOPE VERDICT`；PROVED/REFUTED 成稿还要求 `vs-ASSERTION VERDICT` 也通过）：ACCEPT，或 MINOR_GAPS 且不触及主结论（修订后复审）→ 计为通过；MAJOR_GAPS → 辩论；WRONG → 辩论，复核仍为 WRONG 就撤回该结论。作者自己声明的缺口不进入辩论。
- **"通过首轮审稿"**（触发 H4，只针对 PROVED/REFUTED 成稿）：≥1 份隔离审稿两行判词都为 ACCEPT 或 MINOR_GAPS，且没有 WRONG。PARTIAL 成稿的审稿结果在 S1 汇报里给出。
- **Claims 编号**：成稿和 (g1) 提炼出的每条结论编号为 C1, C2, …（记入 `claims.md`），Lean 和计算两个维度都按 claim 写。
- **首页状态行**按 4 个维度分开写：作者结论 · 主标签 · Lean · 计算；作者结论为 PARTIAL 或 NO_RESULT 时再加 "原命题: 未解决"。
  - Lean：`<claim_id>: N/A | STATEMENT_ONLY | VERIFIED(strict|native; H3 human|H3 pending; scope=…) | FAILED | NOT_RUN`；
  - 计算：`<claim_id>: N/A | NOT_RUN | PASSED(clean-room|sanity; 范围) | FAILED`；
  - `N/A` = 没打算做；`NOT_RUN` = 本应做但没做。
  - 例：`PARTIAL · REVIEWED(partial; AI 2× same family) · Lean: C4,C5: VERIFIED(strict; H3 pending; scope=两条计数恒等式，秩公式只作假设) · 计算: C1–C3: PASSED(sanity; 有限实例，不含渐近界) · 原命题: 未解决`。
  - 不使用 `UNCHECKED` 一词，因为它在 openai/math 的 `formalization.yaml` 里另有含义。

## 8. 常见陷阱
| 陷阱 | 依据 | 对策 |
|---|---|---|
| 恶意顺从 / 字面满足（例如 3D 非周期铺砌用了超过 10^2,800,000 个立方体） | A-10 @ElliotGlazer | 题面写明规模和显式性要求；(d) 第 6 项 |
| 只形式化标题定理 | A-10 @ElliotGlazer | 引理对照表 + DEVIATIONS + Scope + H3 |
| 略过最难一步、空引"标准论证"、引用不含所称结论 | A-9 First Proof | (d) 第 2、3 项逐条判定；定向审稿 |
| 引用有缺陷的旧工作而不说明 | A-10 @littmath | (d) 第 3 项 |
| 只回答字面、漏掉更好的结果 | A-10 @ElliotGlazer（"best result"） | (g1) |
| 过度声称 | A-5 README | §7 标签 |
| 陈述漂移 / 偷换特例 | A-3 Kaplansky | 防偷换条款 + (d) 第 1 项 |
| Lean 陈述与原问题不符 | A-7 comparator 的信任假设："Challenge … controlled by you or trustworthy" | H3 人工核对；不用 Prop 洞 |
| 题目早已被（声称）解决 | 干跑实例：经典开放题 2026 年已有声称证明 | 步骤 1 的前向引用检索 + CLAIMED-RESOLVED 类 |
| 选择偏差 / 隐藏分母 | A-11 SciAm：OpenAI 只公开平均算力 "and no prompts" | 公开题单、尝试次数和失败 |
| 隔离泄漏：会话读到含解题思路的笔记或以往会话；求解者联网读到原文 | 附录 B | §1.1 文件隔离 + 题面 SEARCH POLICY + `web_used` |
| PARTIAL 稿件审稿空转（必然 MAJOR_GAPS，辩论只能 concede） | 附录 B | (d) 两行判词 + `REVIEWED(partial)` |
| 工具没有同号版本 / 内核 Landlock 太旧 / 没有 systemd | 附录 B | 6A-1、6A-5 的降级规则，并写进 Scope |

---

## §T 提示模板（重构，非 OpenAI 原文；直接发英文版；发送前删除未用的 `{{…}}`）
求解时发送 (a0) + `input.vN.md`（即 (a) 题面）+〔(b)〕+ (a1)；`input.vN.md` 只含 (a)。

### (a0) 求解前言（只发给求解者）
```text
Please give a complete rigorous solution to the following problem. Even if the problem is
considered open, the intention is that you resolve it and present a full solution. Your
solution may be either a complete rigorous proof or a well-presented rigorous refutation.
{{OPTIONAL: Brute-force enumerations and computer-assisted proofs are strongly discouraged in
the final writeup.}} A complete resolution is the goal, but PARTIAL and NO_RESULT (defined in
OUTPUT FORMAT below) are permitted honest outcomes: never present partial progress as
complete; report exactly what you proved and mark everything else as a gap.
```

### (a) 题面（= `input.vN.md` 的全部内容）
```text
# Problem
{{OPTIONAL title line: "Open Problem: ..." / "Conjecture (Author, ref): ..."}}
DEFINITIONS AND CONVENTIONS. {{all objects, normalizations, encodings, counting conventions,
degenerate/edge cases; what is fixed before what}}
ALIGNMENT WITH LITERATURE. {{which theorem/equation numbers of which source this matches}}

ASSERTION. Prove or refute: {{precise statement, all quantifiers explicit}}.

RESOLUTION CRITERIA.
- A complete affirmative resolution must establish {{exact scope}}.
- A complete negative resolution must provide {{exact certificate}}.
  {{IF ASYMPTOTIC: an explicit infinite family with complete proofs of the required bounds;
  no finite collection of objects and no numerical table is a refutation.}}
- Not sufficient (counts at most as PARTIAL): {{weakenings, special cases, extra hypotheses,
  known cases}}. Failure of one approach is not a negative resolution.
- Any restriction must be stated explicitly, never silently extended to the general case.
- {{IF CONSTRUCTIVE: size/explicitness/constructivity requirements; define "explicit"}}
NOT REQUIRED. {{stronger statements you do not ask for}}
PERMITTED INPUTS. {{which published results may be cited (with hypotheses); unrefereed
results must be listed as external inputs}}. Do not assume the assertion.
SEARCH POLICY. {{one of: "No web or literature search; use only the inputs above." |
"You may look up only the sources cited above." | "You may search the web and literature;
list every source you consulted."}} {{OPTIONAL: whether sources that discuss this exact
problem may be read}}
```

### (a1) 输出格式（只发给求解者，接在题面和 (b) 之后）
```text
OUTPUT FORMAT.
- The first line and the last line must be identical and must be exactly one of:
  VERDICT: PROVED / VERDICT: REFUTED / VERDICT: PARTIAL / VERDICT: NO_RESULT
- PROVED / REFUTED: you meet the affirmative / negative resolution criteria in full.
- PARTIAL: you rigorously establish at least one precise statement bearing on the ASSERTION
  that falls short of a complete resolution (e.g. an item of the "Not sufficient" list, a
  special case, a reduction). Every claimed statement must be fully proved; everything else
  is a declared gap.
- NO_RESULT: you establish no such statement. Do not invent lemmas; give instead the routes
  you tried and the precise obstacle for each.
- Between the two VERDICT lines: (1) one paragraph stating exactly what you established;
  (2) the full writeup with numbered lemmas (only for proof content actually supplied);
  (3) every external result used, with precise citations; (4) remaining gaps as a numbered
  list ("none" only if there are none); (5) sources consulted online, if any.
```

### (b) 研究包（求解时插在题面与 (a1) 之间）
```text
RESEARCH DIRECTION
{{1-3 sentences: promising approaches; known partial results as "useful starting points"}}.
A useful first milestone is {{intermediate result}}; if you reach only the milestone, report
VERDICT: PARTIAL. Milestones do not replace the full quantifiers of the target.
SELF-CHECK
- Check explicitly where {{key hypotheses}} enter.
- Verify the argument recovers {{known special/degenerate case}}.
- {{OMIT IF NONE: If your argument would also prove {{known-false statement}}, it is wrong;
  locate the error. If it would prove {{much stronger statement believed out of reach}}, flag
  this and re-audit the step that gives the extra strength.}}
STARTING PRIMARY REFERENCES
1. {{Author, Title, URL}} - provides {{what}}; NOT {{limitation}}.
```

### (c) 接续提示（自包含，可在新会话里使用；发送时同样前接 (a0)、后接 (a1)）
```text
# Problem
{{Next, stronger target, stated precisely, with DEFINITIONS as in (a).}}
PREMISES YOU MAY ASSUME (verified; status in brackets):
P1. {{verbatim statement}} [{{REVIEWED / REVIEWED(partial) / VERIFIED(Lean) ...}}]
PRIOR WORK FOR TECHNIQUES ONLY (unverified; do NOT assume its conclusions):
W1. {{statement or attached writeup}} [PROPOSED]
Before building on the premises, briefly re-audit the parts you rely on. State exactly which
premises you use and keep your new hypotheses separate. If you must use a W-item's
conclusion, state your result conditionally ("assuming W1, ...").
{{Append RESOLUTION CRITERIA, NOT REQUIRED, PERMITTED INPUTS and SEARCH POLICY as in (a).}}
```

### (d) 独立审稿人（只附 `input.vN.md` 与成稿）
```text
You are an independent referee. You are given ONLY (1) the problem statement (input, version
{{v}}) and (2) a writeup (sha256 {{h}}) whose first line declares its VERDICT; you may consult
cited primary literature, but not the author's reasoning, code, or other reviews.
Assume errors are likely; find the most serious one first. {{OPTIONAL FOCUS: scrutinize
especially {{section/lemma/step}}.}}
0. Record the declared VERDICT and copy the author's list of declared gaps. A gap the author
   declares is not an error in the author's claims; it only limits what they achieve
   against the ASSERTION.
1. Restate the exact claim(s) proved. List every mismatch with the problem's quantifiers and
   conventions (weaker statement, extra hypotheses, changed definitions), and every place
   where the writeup claims more than it proves.
2. For every lemma/step: VALID / GAP / ERROR / NOT_CHECKED (say why), with a precise reason.
   Flag any step justified only by "standard arguments" or an unspecified citation.
3. For every cited result: is it stated correctly, does the cited work contain it, does it
   apply here, is the work known to be correct? If you cannot access the source, write
   UNVERIFIED-CITATION instead of guessing. Flag borrowed text without attribution.
4. Sanity tests: special cases, counts, and whether the argument would prove something false.
5. If a counterexample is claimed, independently verify the certificate.
6. Check every item of the problem's "Not sufficient" list and the spirit of the problem
   (e.g. astronomically large or non-explicit objects where explicit ones are implied).
Output: CLAIM / DECLARED GAPS / MISMATCHES / STEP TABLE / CITATIONS / SANITY / the most
serious issue NOT declared by the author. End with exactly these two lines, each with one
of the four words:
CLAIMED-SCOPE VERDICT: ACCEPT | MINOR_GAPS | MAJOR_GAPS | WRONG
vs-ASSERTION VERDICT: ACCEPT | MINOR_GAPS | MAJOR_GAPS | WRONG
The first judges only what the writeup claims to establish (declared gaps excluded); the
second judges the writeup as a resolution of the ASSERTION. Do not repair the proof.
```
解析：`^CLAIMED-SCOPE VERDICT: (ACCEPT|MINOR_GAPS|MAJOR_GAPS|WRONG)\s*$` 与 `^vs-ASSERTION VERDICT: (ACCEPT|MINOR_GAPS|MAJOR_GAPS|WRONG)\s*$`。

### (e1) Lean：只写陈述
```text
Write Lean 4 + Mathlib statements for the theorems listed below. Lean/Mathlib version:
{{Lean toolchain; Mathlib commit}}. Output two files: `Defs.lean` (imports only Mathlib; every
custom definition, namespace {{NS}}) and `Challenges/{{Name}}.lean` (imports only Defs; one
theorem per item, every proof `sorry`). Theorem names, exact and fully qualified:
{{NS.thm1, NS.thm2, ...}}. State each item as the writeup claims it: the statement itself if
claimed proved; its negation, or the explicit counterexample property, if claimed refuted.
No Prop-valued definition holes. Use existing Mathlib definitions where possible; list every
new definition and justify that it matches the informal statement. Do not prove anything.
{{OPTIONAL PROJECT CONTEXT: you are in a Lake project pinned to the versions above; check the
statements with `lake build Defs Challenges` (only `sorry` warnings allowed); do not edit
lakefile.toml, lean-toolchain or lake-manifest.json.}}
Output the files and a line-by-line correspondence between each theorem and its informal
statement.
STATEMENTS:
{{numbered informal statements, each with its writeup reference and all hypotheses}}
```

### (e2) Lean：在给定 Challenge 下证明
```text
The Challenge file(s) are fixed and approved; do not modify them. Write the Solution module
(import Defs, never import Challenges) containing theorems with exactly these names and the
same statements: {{NS.thm1, NS.thm2, ...}}. Hard constraints: no `sorry`, `admit`, new
`axiom`, `opaque`/`implemented_by`/`extern` tricks, `debug.skipKernelTC`, or `native_decide`
(it adds axioms outside the list); permitted axioms: propext, Quot.sound, Classical.choice.
Formalize every lemma the paper's argument depends on, in order, with matching names
(Lemma 3.2 -> lemma_3_2); any change of route goes into a DEVIATIONS list. Never weaken the
statement; if you cannot prove it, say so. If formalization reveals an error in the paper,
stop and report it.
PROJECT CONTEXT. Your working directory is a Lake project pinned to {{Lean toolchain}} and
Mathlib {{commit}} (prebuilt Mathlib cache present). Do not edit `Defs.lean`,
`Challenges/{{Name}}.lean`, `lakefile.toml`, `lean-toolchain` or `lake-manifest.json`. Write
your proof to `Solution.lean` (library `Solution`) and check it with `lake build Solution`.
INFORMAL ARGUMENT: {{the writeup's argument for these items, verbatim or with references}}
Deliverables: proof files; lemma table (paper -> Lean -> status); DEVIATIONS; a Scope note
(what is and is not formalized).
```

### (f) 题面清单（14 项；括号内是对应的 (a) 槽位）
1. ☐ 标题（# Problem 下的标题行）
2. ☐ 定义全部自包含（DEFINITIONS）
3. ☐ 退化与边界情形：空、零、重复、被排除的实例（DEFINITIONS）
4. ☐ 量词与固定顺序（ASSERTION）
5. ☐ 双向断言（ASSERTION）
6. ☐ 正面完成标准（affirmative）
7. ☐ 反面完成标准，含伪反例；**渐近型命题要求带证明的无限族**（negative）
8. ☐ 不算解决的情形（Not sufficient；这些情形在 (a1) 下算 PARTIAL）
9. ☐ 不要求的内容（NOT REQUIRED）
10. ☐ 与文献对齐（ALIGNMENT）
11. ☐ 防恶意顺从：规模、显式性、可构造性，并定义 "explicit"（IF CONSTRUCTIVE）
12. ☐ 允许的输入、终稿机算政策（PERMITTED INPUTS / (a0) 开关），以及**检索政策（SEARCH POLICY，必填）**
13. ☐ 形式化友好度：能否写成 Lean 签名或可执行判据（6A 分诊）
14. ☐ 歧义攻击和弱化攻击（模板 (i)，只给 `input.vN.md`）；提示变体与研究包 (b)（或注明 diversity=none）

### (g1) 最佳结果提炼（基于成稿；同质的 PARTIAL 一起附上，只跑 1 次）
```text
Here are the problem and one or more writeups {{that follow the same route}}. Independently
of the literal question, what is the strongest correct statement the writeups actually
establish? List by-product results with exact hypotheses and the lemmas that prove them
(name the writeup); mark each as fully proved in the writeup or conjectural. Do not add new
arguments.
```

### (g2) 辩论回应（新会话扮演作者方；只针对作者未声明的问题）
```text
You are defending the attached writeup for the attached problem. A referee raised: {{numbered
points}}. For each point either (i) concede, state precisely what fails, and say whether the
main claim survives, or (ii) rebut with a complete argument or a checkable computation. Do
not introduce unstated assumptions. End with: claims unchanged / weakened / withdrawn.
```

### (h) Clean-room 计算复核（模型只写代码；agent 负责冻结和比对）
```text
Write an independent verifier. PERMITTED SOURCE: {{frozen problem input.vN.md and spec.md
lines X-Y: definitions, formulas and literal tables extracted verbatim from the writeup}}.
EXCLUDED: any submitted code, previous audit/check code, result files, peer reports.
Implement from the permitted source alone; put a header stating the origin and exclusions.
Output only the code and the exact command to run it, then END OF STAGE (return control to
the orchestrator; do not compute hashes). Also list
explicitly what this check does NOT verify (e.g. the analytic reduction, the formula-to-code
correspondence, the whole proof).
```
Agent 侧：① `sha256sum` 计算哈希、存只读副本、记入 `freeze.json`；② 首次完整运行；③ 冻结之后，在另一个隔离会话里写比对脚本，**只读**旧结果、不读旧实现；④ 输出键数、缺失、多余、不一致、容差和 passed。比对一致只说明可复现，不说明数学断言成立。

### (i) 歧义攻击与弱化攻击（隔离会话，只附 `input.vN.md`）
```text
You are attacking a problem statement, not solving it. Given the statement below:
1. List every reasonable interpretation that differs in truth value or difficulty (quantifier
   order, inclusive/exclusive bounds, counting conventions, what is fixed before what).
2. List every way a solver could "resolve" a weaker or different problem while appearing to
   satisfy the text (special cases, extra hypotheses, finite evidence for asymptotic claims,
   oversized or non-explicit constructions, misuse of cited theorems beyond hypotheses).
3. List undefined terms (including qualitative words such as "explicit") and missing edge
   cases, and check that the SEARCH POLICY is stated.
For each item propose an exact one-sentence fix. Do not attempt the problem.
```

---

## 9. 交付物（`research/<id>/`）
- `intake.md`：含运行模式、隔离实现与冒烟结果、预算、数据外发许可；
- `candidates.csv`：字段 `id, statement, source, status(OPEN/CLAIMED-RESOLVED/RESOLVED), checked_date, progress, formalizability, difficulty_basis, decision_reason`；
- `input.vN.md`：每个签字版本一份，**只含题面**；`prompts/`：签字的提示变体；`decisions.md`：H1–H3 的签字记录（原话 + 时间；H3 注明 HUMAN / AGENT_ONLY）；
- `runs.csv` 加 `runs/`（每次实际发送的 `prompt.txt`、原始输出、事件日志）；`reviews/`：审稿、辩论、定向审稿分档存放；
- `claims.md`：`claim_id, statement, source(稿件/引理), label, lean_status, compute_status, parent_claims`；
- `verification/`：Lean 开发目录与验收目录、challenge JSON、`comparator.log`、`print_axioms.log`、rg 扫描结果、Scope；或脚本 + `freeze.json` + 比对报告 + "未验证项"；
- `manuscript/`：首页状态行、外部输入清单、AI 使用声明；选 S1 选项③时为 `ARCHIVE.md`。

## 附录 A：逐字依据（2026-10-08 实时核对）
- **A-1** Mézard 前言（`reasoning_traces/mezard-parisi-formula.tex`）："Please give a complete rigorous solution to the following problem. Even if the problem is "open", the intention is that you should resolve it and present a full solution. Your solution can be either a complete, rigorous resolution or a well-presented, rigorous refutation. Brute-force enumerations and computer-assisted proofs are strongly discouraged in the final writeup."
- **A-2** basic-SDP："Failure of one rounding rule or one attempted reduction is not a negative resolution. No proof or candidate algorithm is supplied here."
- **A-3** Kaplansky："A useful first milestone is a new class of groups not covered by the known sufficient hypotheses, or a precise local-to-global criterion whose remaining hypothesis is explicitly isolated. These milestones do not replace the full quantifiers of the target." 以及 "must identify its restriction instead of silently extending it to arbitrary groups"。
- **A-4** 接续（quasipolynomial）："Please give quasipolynomial bounds for k-th arithmetic progressions for all k>=3. Use the previous bounds you have done."
- **A-5** README："On average, each result used three hours of ChatGPT Pro thinking compute with that model."；"Some of the unformalized results could have issues."
- **A-6** Bernoulli 手稿：
  - `verification/independent-certificate/first-freeze.json` 中有 "permitted_source": "input.md lines 1–1124 only; mathematical specification and literal Tables I/II"，以及 "excluded_sources": "optional submitted Python; all previous audit/check source; coefficient dictionaries; exact-results.json; peer reports"；
  - `…/independent-certificate/output/oracle-comparison.json` 中有 "oracle_implementation_read": false；
  - `verification/checks/README.md`："This command verifies finite algebra and coefficient comparisons, not the analytic reduction, the correspondence between mathematical formulas and code, or the whole proof."
- **A-7** comparator（https://github.com/leanprover/comparator README）：
  - 保证所列定理 "Prove the same statement as provided in `Challenge`"、"Use no more axioms than listed in `permitted_axioms`"、"Be accepted by the Lean kernel"；
  - 前提包括 "The transitive closure of imports of `Challenge.lean` … are controlled by you or trustworthy"，以及 "You have not previously tried to compile the `Solution` file or any other potentially adversarial files"；
  - 定义洞："all definition hole solutions **must** always be checked with an additional (potentially human) verifier"。
  - openai/math 的配置：`{"challenge_module": "Challenges.MyProblem", "solution_module": "My.Solution", "theorem_names": ["My.main"], "definition_names": [], "permitted_axioms": ["propext", "Quot.sound", "Classical.choice"], "enable_nanoda": false}`。
  - Scope 示例（`lean/docs/003.md`）："The paper's later applications are not included."
- **A-8** First Proof §4.3（arXiv:2606.18119）：
  - "The following prompt was provided to First Proof by Mehtaab Sawhney. For all 10 problems, we used this prompt and we set the reasoning/thinking mode to 'xhigh' (the highest option)."
  - 提示全文（6 句）："Consider the following research-level math question. [Question] Carefully consider the question and provide a complete and rigorous proof. Provide either complete proofs or careful citations for all intermediate statements in the proof. Work very hard to prove the question and do not return until a complete proof is achieved. Make sure to structure the proof with careful lemmas and theorem statements. The output must be a compilable LaTeX document conforming to the standards of rigor and scholarship prevailing in mathematical literature."
- **A-9** 同一报告：
  - 失败模式："glossing over the most difficult steps, sometimes asserting that a key claim follows from "standard arguments" without justification, or citing papers that do not actually contain the claimed results"；
  - 预跑："queried the following models with Zero Data Retention via Openrouter, with a 30 minute timeout: ChatGPT 5.4/5.5-high and -xhigh, Gemini 3.1 Pro, and Opus 4.7. These models did not solve any of the ten problems"；
  - Table 4 列出了逐题成本。
- **A-10** X：
  - @ElliotGlazer "Malicious compliance with the prompt request"（https://x.com/ElliotGlazer/status/2107617492319580583）；
  - @ElliotGlazer "When you ask GPT to formalize a paper's results, it will often focus on the headline result and take shortcuts on hard-to-formalize lemmas."（https://x.com/ElliotGlazer/status/2108020474592846058）；
  - @littmath "Paper seems to cite some wrong existing work without mentioning it's wrong"（https://x.com/littmath/status/2107616410897678765）；
  - @ElliotGlazer "…still has the tendency to literally answer the user query and not pay attention to "what's actually the best result that came out of this research project?""（https://x.com/ElliotGlazer/status/2107991269654065420）。
- **A-11** SciAm（https://www.scientificamerican.com/article/openai-unleashes-hundreds-more-math-results-upon-a-field-already-in-shock/）："OpenAI is choosing only to reveal the average compute time for a problem, with some additional statistics—and no prompts."；"some results might have taken multiple attempts"。

## 附录 B：2026-10-08 端到端试跑摘要（v2 → v3 的依据）
- **设置**：STANDARD 模式（单模型族：Codex CLI，最高单 agent 档位）；题目为一道量子 CSS 码的 kd² = O(n) 型开放问题（W4/D2 类）；N=4，提示完全相同（diversity=none）；每个会话关在 bubblewrap 里（§1.1 的形式），但题面没写检索政策，4 个求解者都联网读了题目出处的论文。
- **结果**：4/4 `VERDICT: PARTIAL`，都把原命题约化到同一个未证估计；原命题无结果（S1）。2 份隔离审稿对完整命题都判 MAJOR_GAPS，但部分结论各步都判 VALID，v2 的标签只能停在 PROPOSED，辩论只能 concede（→ v3 的两行判词与 `REVIEWED(partial)`）。(g1) 跑了 4 次，内容高度重复。歧义攻击发现：前言"必须解决"与允许 PARTIAL 冲突、PARTIAL/NO_RESULT 未定义、"explicit" 含义不明、VERDICT 行字面格式有歧义（→ (a0)/(a1)）。
- **验证**：6A 只用在一个子引理的两条恒等式上（`theorem_names` 两项），comparator `EXIT=0`，`#print axioms` 只有 3 条标准公理；H3 由调度 agent 代核。工具问题：comparator/lean4export 没有 v4.34.1 tag、Landlock ABI v6 < v9、没有 systemd 用户会话、验收侧依赖来源未定义（→ 6A-1…6A-9）。6B 只做了调度 agent 自写的 sanity 检查（→ `PASSED(sanity)`）。
- **开销**：墙钟约 61 min，14 次模型调用（冒烟 1、求解 4、(g1) 4、(d) 2、(i) 1、(e1) 1、(e2) 1），input 约 3.06M tokens（缓存命中约 2.46M），output 约 187K tokens。研究者最终选择 S1 选项③归档。

## 来源
- openai/math：https://github.com/openai/math （`reasoning_traces/`、`preprints/*/verification/`、`lean/ComparatorChallenges/`、`lean/docs/`）
- comparator：https://github.com/leanprover/comparator
- landrun：https://github.com/Zouuup/landrun
- First Proof 第二批：https://1stproof.org/assets/docs/report.pdf （arXiv:2606.18119）
- OpenAI《Ten advances in mathematics》：https://openai.com/index/ten-advances-in-mathematics/
- OpenAI 单位距离反证：https://openai.com/index/model-disproves-discrete-geometry-conjecture/
