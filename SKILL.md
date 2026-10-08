---
name: "OpenAI Math Skill"
description: "Use when a researcher gives a research topic, a rough question, or a precise open problem in mathematics, theoretical physics, CS theory, or another proof/derivation-heavy field and wants reasoning models to attack it the openai/math way: verified candidate problems from a vague topic, self-contained prove-or-refute statements, best-of-N solving, isolated review and debate, chaining, Lean or computational verification, and honest status labels. 中文触发：用 AI 做数学/理论物理/理论计算机研究、开放问题、猜想、证明或否证、写题面、best-of-N、AI 审稿、Lean 形式化、只有一个研究方向想找题。"
---

# OpenAI 式研究工作流（openai/math 模式）v2

> **定位**：本技能把 openai/math（2026-10 发布，722 篇手稿）公开的提示摘录、推理摘要和审核附件，加上 First Proof 第二批报告和 OpenAI 前作博客，整理成**与工具无关、可降级**的工作流。
> **声明**：
> - §T 的模板以及流程默认值（N、审稿人数、辩论轮数、预算）都是本技能的**重构或建议**，不是 OpenAI 原文或其实际 harness；逐字原文只出现在附录 A。
> - OpenAI 用的是未发布的内部模型。公开模型的产出率要**大幅调低**：First Proof 预跑中，公开模型在 30 分钟超时下，十题**一题也没解出**（附录 A-9）。

## 0. 快速开始（先读这一节）
1. **判定输入**（§3 I-3）：只有方向 → 步骤 1 找候选；粗略问题 → 1 个形式化 + 1–2 个变体；精确陈述 → 直接步骤 2。
2. **环境自检**（§1）：确定 FULL / STANDARD / SOLO 模式，它决定隔离方式和标签上限。
3. **定预算**（§2）：研究者没给就用默认值，**花费上限必须问一句**。
4. **步骤 1**：交付 3–5 个**核实过出处**的候选 → 【停点 H1：研究者选题】。
5. **步骤 2**：按清单 (f) 写 `input.md`（模板 (a)，可选 (b)），用模板 (i) 做歧义攻击 → 【停点 H2：签字】。
6. **步骤 3**：best-of-N 求解（模板 (a)/(b)），解析 VERDICT 行，必要时跑 (g1)。
7. **步骤 4**：隔离审稿（模板 (d)），有分歧就辩论（g2），涉及计算就做 clean-room 复核（h）。
8. **步骤 5–7**：链式推进（c）→ 验证（6A Lean 走 e1/e2 并经【H3】；6B 做计算或符号检查）→ 意义筛选与成稿。
9. 全程按 §2 的停止规则，只在【H4 里程碑】时汇报。所有产物放进 `research/<id>/`（§9）。

## 何时使用 / 不用
- **用**：结论能写成精确陈述（证明 / 否证 / 给出界 / 推导），且研究者愿意投入审稿和验证。
- **不用**：纯经验问题；只想要综述；无法写成可判定陈述、研究者又不愿意收窄的问题。
- 理论物理推导题：把 "Prove or refute" 改成 "Derive or refute"，`DERIVED` 等同于 PROVED。

## 核心原则
1. **杠杆在题面**。First Proof 上 OpenAI 员工用的求解提示只有 6 句；openai/math 的题面可长达数千字符，几乎都在消歧和堵捷径。
2. **双向开放**：prove **or** refute，两边都写清完成标准。
3. **防偷换**：特例、弱化、单一方法失败都**不算**解决，要写进题面。
4. **信息隔离**：审稿者只看题面和成稿；计算复核从规格出发重写，先冻结再比对。
5. **诚实分级**：标签按 §7 打；每项验证都写明"验证了什么、没验证什么"。
6. **公布分母**：完整记录题单、尝试次数和失败。

---

## 1. 环境自检与运行模式（开工前做一次，写进 `intake.md`）
先探测 6 项能力：① 能否开**隔离会话**（子 agent、无历史的 API 调用、研究者手动开新窗口都算；同一上下文"换个角色"**不算**）；② 可用的**模型族**数量；③ 能否**执行代码**（含 CAS，如 SymPy/mpmath）；④ 有无 **Lean 4 + Mathlib**；⑤ 有无 **comparator**（Linux + landrun + lean4export，见 6A）；⑥ 能否**联网核对文献**。

| 模式 | 条件 | best-of-N | 审稿 | 标签上限 |
|---|---|---|---|---|
| FULL | 能隔离，且有 ≥2 个模型族 | 各次在独立会话里跑，可混用模型 | 2 名审稿者，至少 1 名来自另一模型族 | VERIFIED |
| STANDARD | 能隔离，只有 1 个模型族 | 独立会话 | 2 个独立会话（同族）；标签注明 "AI 2× same family" | VERIFIED |
| SOLO | 不能隔离（只有你自己） | 在同一上下文里串行尝试，并记录"非独立" | **不能自审**：生成"审稿包"（input.md + 成稿 + 模板 (d)）交研究者到别处跑 | **PROPOSED** |

- 无代码执行：跳过 (h) 和数值检查，记为 `NOT_RUN`（不得写成通过）。无联网：步骤 1/7 的文献核查交给研究者，引用标"未核"，**不得进入 H2**。
- 无 Lean（或装不起）：交付未执行的 Lean 陈述草稿（Lean 记为 `STATEMENT_ONLY`）+ 6B 检查；Lean 永远不写成通过。
- **数据外发**：受理时问"题目和未发表的思路能否发给第三方模型 API？"，优先用零数据保留接口（First Proof 预跑即用 Zero Data Retention）。

## 2. 预算与停止规则
**默认预算**（研究者未指定时，写进 `intake.md`，研究者可覆盖）：
- 每题 N=4 次求解（SOLO 模式 N=2）；每份 PROVED/REFUTED 2 份审稿；辩论 ≤2 轮；
- 总模型调用 ≤20 次，墙钟 ≤24 h；
- **花费上限必须问**，没有回答就只按调用次数上限执行。

**量级参考**（已核实）：First Proof 报告 Table 4，单次 xhigh 的 ChatGPT 5.5 Pro 调用每题约 US$8–16（10 题合计 $117，墙钟 5.8 h），多 agent 编排系统每题 $36–951；openai/math README 称每个结果平均用 "three hours of ChatGPT Pro thinking compute"。

长时调用要放到后台或异步执行，避免撞上工具超时。

**停止规则**：
- **S1**：N 次全为 NO_RESULT 或 PARTIAL → **不自动加跑**；汇报并给 3 个选项：用 PARTIAL 做 (c) 接续 / 弱化变体（**需重新签字**）/ 归档为失败（runs、部分进展、已排除方法、剩余难点照样交付，分母照常公布）。
- **S2**：出现 REVIEWED 结果 → 停止求解，转入验证。
- **S3**：辩论 2 轮仍有分歧 → 标 `DISPUTED`，汇报，建议交给人类专家。
- **S4**：预算用到 80% 预警汇报，用到 100% 停止并按 S1 的三个选项汇报；研究者不在时，未完成的审稿、验证标 `NOT_RUN`。
- **S5**：任何一次运行暴露题面歧义或漏洞 → 暂停，修题面，**重新签字**；修改前的运行作废并记入日志。
- **链式**：每个接续目标都是一个新的小项目，要走一次简化签字（H2）；接续深度默认 ≤3，歧义修订默认 ≤3 轮。

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
- **H2 签字**：展示 `input.md` 全文、14 项清单结果（逐项写"通过 / 默认 / 有意不写"）、歧义攻击发现的问题及处理、**标记假设**、运行模式、预算。必须得到明确确认（如"确认""OK 开跑"），沉默或含糊的回复不算。
- **H3 Lean 陈述核对**：只在走 6A 时出现。研究者（或其指定的专家）逐字核对 Challenge 文件。
- **H4 里程碑**：候选结果通过首轮审稿（定义见 §7）、审稿分歧或致命缺陷、需要修改题面、触发 S1/S4。

### I-5 汇报模板
```text
【状态报告】项目 {{id}} ｜题面 v{{n}} ｜模式 {{FULL/STANDARD/SOLO}} ｜{{时间}}
触发：{{H4 子项 / S1–S5}}
进度：运行 {{k}}/{{N}}；PROVED {{a}} · REFUTED {{b}} · PARTIAL {{c}} · NO_RESULT {{d}}
最佳结果：{{一句话精确陈述}}  标签：{{§7}}
审稿：{{R1 结论}} / {{R2 结论}}；关键问题 ≤3 条：{{…}}
验证：{{Lean/计算/无}}，范围 {{…}}；未验证：{{…}}
预算：已用 {{调用数/花费/时长}}（{{%}}）
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
研究者：选①。 助手：〈input.md + 14 项结果 + 标记假设 + 模式 + 预算〉请回复"确认"后开跑。
```

---

## 4. 流程
### 步骤 0：受理
`intake.md` 写 8 项：① 目标命题或候选；② 领域约定（记号、归一化、"标准定义"的具体版本）；③ 已知文献（最好结果、反例、已知障碍）；④ 成功标准；⑤ 验证途径；⑥ 预算（§2）；⑦ 运行模式（§1）；⑧ 数据外发许可。缺项按 I-2 补齐。

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
- 按清单 (f) 逐项写，填进模板 (a)。需要给方向时加 (b)；(b) 也可以作为 best-of-N 的一个变体。
- **渐近型命题**（"存在常数…""对所有充分大的 n…"）：反面完成标准必须要求**带证明的无限族**，写明"有限个对象不构成反例"。
- **防恶意顺从**：凡是"存在一个构造"的题，都写明规模、显式性、可构造性要求。
- 用模板 (i) 在**隔离会话**里做歧义攻击和弱化攻击，反复修改到只剩一种合理解读；SOLO 模式下自己逐条做，并在 H2 材料里注明。
- 发送前检查：没有残留的 `{{`；版本号已写进文件名（`input.v1.md`）。

### 步骤 3：求解（best-of-N）
- 用当前可用的最强推理模型、最高思考档位，每次运行彼此不可见。多样化手段：轮换 (b) 的写法，有时不给方向，有时换等价表述。
- **解析**：用正则 `^VERDICT: (PROVED|REFUTED|PARTIAL|NO_RESULT)\s*$` 匹配首行和末行。缺失、首末不一致或输出被截断，运行状态记为 `INVALID_OUTPUT`，不得从正文猜测。运行状态共 5 种：`OK / TIMEOUT / TOOL_ERROR / INVALID_OUTPUT / BUDGET_EXHAUSTED`。PARTIAL = 严格证明了一个比 ASSERTION 弱的精确陈述。
- 对每份结论为 PROVED、REFUTED 或 PARTIAL 的成稿跑 (g1)；提炼出的副产品一律标 PROPOSED，另行送审。
- 结论相反的两份稿件**都送审**，至少有一份是错的。
- **日志**（`runs.csv`）：`problem_id, statement_version, template, model, mode, run_index, thinking_level, wall_time, tokens, cost, run_status, verdict, output_path, parent_claims`；每次运行**实际发送的完整提示词**和原始输出存进 `runs/<run_id>/`。

### 步骤 4：独立审稿（信息隔离）
- **证明审稿**：用模板 (d)。审稿者只拿到冻结的 `input.vN.md` 和成稿，看不到作者推理、代码或其他审稿意见。第二轮可以加 `FOCUS`，做定向审稿。
- **辩论**：任何 WRONG 或 MAJOR → 用 (g2) 写回应 → 由**新的**审稿会话复核，最多 2 轮，仍不过就标 DISPUTED（S3）。
- **计算复核**：成稿依赖计算时用模板 (h)。复核者可读冻结题面和 `spec.md`（由 agent 从成稿逐字摘出的定义、公式、表格，带行号），不读代码、结果和审稿。冻结、哈希、运行、比对都由 agent 用工具完成（`sha256sum`、`chmod -w`），不能让模型报哈希。**比对失败**：先分类（执行错误 / 规格歧义 / 复核实现 bug / 原结果错误）；冻结代码不得原地改，修订版另行冻结为 v2，并注明"已看过差异"，不再算首次盲复核；原因查明前，相关 claim 的计算项记为 `FAILED`，按 H4 汇报。
- **证据分档**：源文阅读、文献核查、辩论、计算执行、终审各存一份记录。"审过了"本身不能当证明前提。
- **终审绑定终稿**：成稿修改后，必须对最终源文件做一次全新的整篇审稿。

### 步骤 5：链式推进
- 已审或已验证的结果作为**给定前提**，用模板 (c) 提出下一个更强的目标。前提要**逐字贴入**，并注明验证状态。
- 未验证的前提放进 PRIOR WORK，**只复用技术，不复用结论**；非用不可时，新结论写成条件式"若 P1 则 Q"。
- **按 claim 复用**：只有 claim 的全部证明义务都被审稿或形式化覆盖，才能当无条件前提；只验证了局部的（如计算项）不行。前提被撤回时，依赖它的结果（`parent_claims`）全部降级并重审。

### 步骤 6：验证
**6A 分诊**：满足以下三条才走 Lean，否则走 6B：
1. 所需定义 Mathlib 都有，或者很少（先查 Mathlib 文档或源码）；
2. 陈述能在约 1 小时内写成 Lean 签名；
3. 环境有 Lean（安装 elan、工具链和 Mathlib 缓存要数 GB，`lake exe cache get`）。

**6A Lean 4 + Mathlib**：
1. **项目骨架**（受信任部分由人或调度 agent 生成，不由求解 agent 生成）：`verification/lean/` 下放 `lean-toolchain`（固定版本，openai/math 用 `leanprover/lean4:v4.34.1`）、`lakefile.toml`（`require mathlib` 固定 rev；声明 `[[lean_lib]]` 为 `Defs`、`Challenges`、`Solution`）、`lake-manifest.json`。自定义定义放 `Defs/`（只 `import Mathlib`），Challenge 和 Solution 都 `import Defs`；**Solution 不得 import Challenge**；主定理全限定名（如 `MyProblem.main`）和陈述逐字相同。
2. 用 (e1) 写 Challenge（证明为 `sorry`），**按成稿结论写**：PROVED 写 `theorem main : Statement`；REFUTED 写 `theorem main : ¬ Statement`，或写成满足反面完成标准的具体反例（反例对象放 `Defs/`）→ 【H3 人工逐字核对】。只有题面本身要求"判定真假"时才用 `Prop` 定义洞（`definition_names` 非空），而且 hole 的取值必须额外人工审查，不能只凭 comparator 通过就标 VERIFIED（附录 A-7）。
3. 用 (e2) 写 Solution。**严格路径**（白名单只有 propext / Quot.sound / Classical.choice）禁止 `sorry`、`admit`、新 `axiom`、`native_decide`、`implemented_by`、`extern`、`debug.skipKernelTC`。`native_decide` 会引入 `Lean.ofReduceBool` 一类额外公理（以 `#print axioms` 实际输出为准），**不可能**通过 3 公理白名单。研究者明确批准时走**扩展路径**：把实际列出的额外公理加进 `permitted_axioms`，标为 `VERIFIED(Lean+native)`，并在 Scope 里写明扩大了哪些信任基础。
4. **开发环境与验收环境分开**：agent 在开发目录里随意 build。验收时另建干净目录，只放人工核对过的受信任部分（Defs、Challenge、lakefile、manifest、toolchain）和 Solution 源文件（不带 `.lake/build`），**第一次编译只能由 comparator 完成**（comparator 自己在 landrun 沙箱里构建两边）。工具要求：`comparator`、版本兼容的 `lean4export`、从 main 分支源码编译的 `landrun`，放进 `PATH` 或用 `COMPARATOR_LANDRUN` / `COMPARATOR_LEAN4EXPORT` 指定。以非 root 用户运行：
   `lake update && lake exe cache get`
   `systemd-run --property=RestrictAddressFamilies=~AF_UNIX --user --pty -E PATH="$PATH" --working-directory "$(pwd)" -- bash -c 'lake env comparator Challenges/MyProblem.json'`
   退出码和完整日志存进 `comparator.log`，退出码非 0 即不通过。没有 systemd 用户会话时去掉包装，并在 Scope 里注明。
5. **公理与扫描**：新建 `Axioms.lean`，写 `import Solution` 和对每个主定理、引理表中每个声明的 `#print axioms <name>`，用 `lake env lean Axioms.lean` 跑出真实输出并存档（模型不得代填）。辅助扫描**只用于定位**，不作为验收依据，只扫 Solution 和 Defs、不扫 Challenge（注释里的 `sorry` 也会命中，要人工排除）：
   `rg -n --no-ignore -g '*.lean' -e '\b(sorry|admit|native_decide|implemented_by|extern|unsafe|debug\.skipKernelTC|ofReduceBool)\b' -e '^\s*(@\[[^]]*\]\s*)?((private|protected|noncomputable)\s+)*(axiom|opaque)\b' Solution/ Defs/`
6. **没有 comparator**（非 Linux，或装不了 landrun）时降级：在干净目录里 `lake build` + 第 5 步 + H3，标签注明 "Lean, no comparator"。
7. 交付：引理对照表、DEVIATIONS 清单、`#print axioms` 原始输出、Scope 文档（覆盖了什么、**没覆盖**什么，以及 Lean、Mathlib rev、comparator commit、lean4export 版本）。

**6B 暂不形式化**（物理推导、渐近分析、数值常数，或形式化成本超出预算）：CAS 独立复算关键恒等式、量纲和极限情形；区间算术或高精度数值检查（覆盖边界、退化、随机参数，并对已知特例回归）；可重放脚本 + 哈希清单，按 (h) 复核；每项检查都写 "verifies X, not Y"（附录 A-6）。

### 步骤 7：意义筛选与成稿
- **筛选**：是否真的新（再查一次文献和前向引用）？是否只是已知方法的直接套用？是否好得可疑（能推出已知为假的命题就回到步骤 4）？
- **成稿结构**：标题；**首页标签行**；主定理（精确陈述）；与已知结果的关系；证明；**外部输入清单**（版本号和定理号）；验证说明（各项覆盖了什么）；局限；AI 使用声明（模型、提示、尝试次数、模式）。
- **引用核查**：逐条打开引文，确认含有所称的结论，检查撤稿和勘误；引用有缺陷的旧工作时要在正文明说；检查有无未标注出处的措辞沿用。

## 7. 状态标签与判定规则
- 用"**一个主标签** + 可选的 VERIFIED 范围"表示：
  - `PROPOSED`：没有隔离的独立审稿（包括只做过自检、SOLO 模式）；
  - `DISPUTED`：审稿有分歧，或 MAJOR / WRONG 问题在辩论后仍未解决；
  - `REVIEWED`：≥2 份隔离审稿为 ACCEPT 或 MINOR，minor 问题已修改，且终稿做过整篇复审；注明审稿来源，例如 "AI 2× same family" 或 "1 human expert"；
  - `VERIFIED(Lean | computation; scope=…)`：附加在主标签之后。范围之外的部分仍按主标签算。
- **审稿结论映射**：ACCEPT，或 MINOR_GAPS 且不触及主结论（修订后复审）→ 计为通过；MAJOR_GAPS → 辩论；WRONG → 辩论，复核仍为 WRONG 就撤回该结论。
- **"通过首轮审稿"**（触发 H4）：≥1 份隔离审稿为 ACCEPT 或 MINOR，且没有 WRONG。
- **首页状态行**把 4 个维度分开写：作者结论 · 主标签 · Lean（`NONE / STATEMENT_ONLY / VERIFIED(strict|native)`）· 计算（`NONE / NOT_RUN / PASSED(范围) / FAILED`），例如 `PROVED · REVIEWED(AI 2× same family) · Lean: STATEMENT_ONLY · 计算: PASSED(表 I/II，不含解析归约)`。未执行的检查一律写 `NOT_RUN`。不使用 `UNCHECKED` 一词，因为它在 openai/math 的 `formalization.yaml` 里另有含义。

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

---

## §T 提示模板（重构，非 OpenAI 原文；直接发英文版；发送前删除未用的 `{{…}}`）

### (a) 主求解提示
```text
Please give a complete rigorous solution to the following problem. Even if the problem is
considered open, the intention is that you resolve it and present a full solution. Your
solution may be either a complete rigorous proof or a well-presented rigorous refutation.
{{OPTIONAL: Brute-force enumerations and computer-assisted proofs are strongly discouraged in
the final writeup.}} If you cannot obtain a complete resolution, do not present partial
progress as complete: report exactly what you proved and mark everything else as a gap.

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
- {{IF CONSTRUCTIVE: size/explicitness/constructivity requirements}}
NOT REQUIRED. {{stronger statements you do not ask for}}
PERMITTED INPUTS. {{which published results may be cited (with hypotheses); unrefereed
results must be listed as external inputs; tool/search policy}}. Do not assume the assertion.

OUTPUT FORMAT.
- First line exactly: VERDICT: PROVED | REFUTED | PARTIAL | NO_RESULT
- Then: (1) a paragraph stating exactly what you established; (2) the full writeup with
  numbered lemmas; (3) every external result used, with precise citations; (4) remaining
  gaps ("none" only if none).
- Last line: repeat the VERDICT line.
```

### (b) 研究包（追加在 (a) 的 RESOLUTION CRITERIA 之后）
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

### (c) 接续提示（自包含，可在新会话里使用）
```text
# Problem
{{Next, stronger target, stated precisely, with DEFINITIONS as in (a).}}
PREMISES YOU MAY ASSUME (verified; status in brackets):
P1. {{verbatim statement}} [{{REVIEWED / VERIFIED(Lean) ...}}]
PRIOR WORK FOR TECHNIQUES ONLY (unverified; do NOT assume its conclusions):
W1. {{statement or attached writeup}} [PROPOSED]
Before building on the premises, briefly re-audit the parts you rely on. State exactly which
premises you use and keep your new hypotheses separate. If you must use a W-item's
conclusion, state your result conditionally ("assuming W1, ...").
{{Append RESOLUTION CRITERIA, NOT REQUIRED, PERMITTED INPUTS and OUTPUT FORMAT from (a).}}
```

### (d) 独立审稿人
```text
You are an independent referee. You are given ONLY (1) the problem statement (input, version
{{v}}) and (2) a writeup (sha256 {{h}}); you may consult cited primary literature, but not
the author's reasoning, code, or other reviews.
Assume errors are likely; find the most serious one first. {{OPTIONAL FOCUS: scrutinize
especially {{section/lemma/step}}.}}
1. Restate the exact claim proved. List every mismatch with the problem's quantifiers and
   conventions (weaker statement, extra hypotheses, changed definitions).
2. For every lemma/step: VALID / GAP / ERROR / NOT_CHECKED (say why), with a precise reason. Flag any step justified
   only by "standard arguments" or an unspecified citation.
3. For every cited result: is it stated correctly, does the cited work contain it, does it
   apply here, is the work known to be correct? If you cannot access the source, write
   UNVERIFIED-CITATION instead of guessing. Flag borrowed text without attribution.
4. Sanity tests: special cases, counts, and whether the argument would prove something false.
5. If a counterexample is claimed, independently verify the certificate.
6. Check every item of the problem's "Not sufficient" list and the spirit of the problem
   (e.g. astronomically large or non-explicit objects where explicit ones are implied).
Output: CLAIM / MISMATCHES / STEP TABLE / CITATIONS / SANITY / most serious issue, then a
final line "REFEREE VERDICT: ACCEPT | MINOR_GAPS | MAJOR_GAPS | WRONG". Do not repair the proof.
```

### (e1) Lean：只写陈述
```text
Write Lean 4 + Mathlib statement(s) for the following theorem(s), in a single file that
imports only Mathlib, with every proof `sorry`. Lean/Mathlib version: {{pin}}. Use existing
Mathlib definitions where possible; list every new definition and justify that it matches
the informal statement. Put custom definitions in module Defs (imports only Mathlib). State
`theorem {{Name}}.main : {{Statement}}` if the writeup claims PROVED, or `: ¬ {{Statement}}`
(or the explicit counterexample property) if it claims REFUTED; no Prop-valued definition
holes. Do not prove anything. Output the file
and a line-by-line correspondence to the informal statement.
```

### (e2) Lean：在给定 Challenge 下证明
```text
The attached Challenge file is fixed and human-approved; do not modify it. Write a Solution
module (import Defs, never import Challenge) whose theorems have exactly the same names and
statements. Hard constraints: no `sorry`, `admit`, new `axiom`, `opaque`/`implemented_by`/
`extern` tricks, `debug.skipKernelTC`, or `native_decide` (it adds axioms outside the list);
permitted axioms: propext, Quot.sound, Classical.choice.
Formalize every lemma the paper's argument depends on, in order, with matching names
(Lemma 3.2 -> lemma_3_2); any change of route goes into a DEVIATIONS list. Never weaken the
statement; if you cannot prove it, say so. If formalization reveals an error in the paper,
stop and report it. Deliverables: proof files; lemma table (paper -> Lean -> status);
DEVIATIONS; a Scope note (what is and is not formalized).
```

### (f) 题面清单（14 项；括号内是对应的 (a) 槽位）
1. ☐ 标题（# Problem 下的标题行）
2. ☐ 定义全部自包含（DEFINITIONS）
3. ☐ 退化与边界情形：空、零、重复、被排除的实例（DEFINITIONS）
4. ☐ 量词与固定顺序（ASSERTION）
5. ☐ 双向断言（ASSERTION）
6. ☐ 正面完成标准（affirmative）
7. ☐ 反面完成标准，含伪反例；**渐近型命题要求带证明的无限族**（negative）
8. ☐ 不算解决的情形（Not sufficient）
9. ☐ 不要求的内容（NOT REQUIRED）
10. ☐ 与文献对齐（ALIGNMENT）
11. ☐ 防恶意顺从：规模、显式性、可构造性（IF CONSTRUCTIVE）
12. ☐ 允许的输入、终稿机算政策（PERMITTED INPUTS / 前言开关）
13. ☐ 形式化友好度：能否写成 Lean 签名或可执行判据（6A 分诊）
14. ☐ 歧义攻击和弱化攻击（模板 (i)）；（可选）研究包 (b)

### (g1) 最佳结果提炼（基于成稿；同一会话或新会话皆可）
```text
Here are the problem and a writeup. Independently of the literal question, what is the
strongest correct statement the writeup actually establishes? List by-product results with
exact hypotheses and the lemmas that prove them; mark each as fully proved in the writeup or
conjectural. Do not add new arguments.
```

### (g2) 辩论回应（新会话扮演作者方）
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

### (i) 歧义攻击与弱化攻击（隔离会话）
```text
You are attacking a problem statement, not solving it. Given the statement below:
1. List every reasonable interpretation that differs in truth value or difficulty (quantifier
   order, inclusive/exclusive bounds, counting conventions, what is fixed before what).
2. List every way a solver could "resolve" a weaker or different problem while appearing to
   satisfy the text (special cases, extra hypotheses, finite evidence for asymptotic claims,
   oversized or non-explicit constructions, misuse of cited theorems beyond hypotheses).
3. List undefined terms and missing edge cases.
For each item propose an exact one-sentence fix. Do not attempt the problem.
```

---

## 9. 交付物（`research/<id>/`）
- `intake.md`：含运行模式、预算、数据外发许可；
- `candidates.csv`：字段 `id, statement, source, status(OPEN/CLAIMED-RESOLVED/RESOLVED), checked_date, progress, formalizability, difficulty_basis, decision_reason`；
- `input.vN.md`：每个签字版本一份；`decisions.md`：H1–H3 的签字记录（原话 + 时间）；
- `runs.csv` 加 `runs/`；`reviews/`：审稿、辩论、定向审稿分档存放；
- `verification/`：Lean 项目 + challenge JSON + `#print axioms` + Scope；或脚本 + `freeze.json` + 比对报告 + "未验证项"；
- `manuscript/`：首页标签行、外部输入清单、AI 使用声明。

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

## 来源
- openai/math：https://github.com/openai/math （`reasoning_traces/`、`preprints/*/verification/`、`lean/ComparatorChallenges/`、`lean/docs/`）
- comparator：https://github.com/leanprover/comparator
- First Proof 第二批：https://1stproof.org/assets/docs/report.pdf （arXiv:2606.18119）
- OpenAI《Ten advances in mathematics》：https://openai.com/index/ten-advances-in-mathematics/
- OpenAI 单位距离反证：https://openai.com/index/model-disproves-discrete-geometry-conjecture/
