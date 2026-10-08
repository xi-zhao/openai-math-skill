# Changelog

## v6 snapshot (2026-10-08)

Snapshot while end-to-end polishing continues (not the final mature version). / 打磨进行中的快照，不是最终成熟版。

- **Timeout**: `setsid --wait` so the timeout wrapper actually waits (v5's `setsid` returned immediately). / 用 `setsid --wait` 修复超时包装立刻返回的问题。
- **429 retries**: explicit retry limit for rate-limit errors. / 为 429 限流错误规定重试上限。
- **Ambiguity attack**: web flag for the (i) ambiguity-attack step. / 歧义攻击步骤的联网标志。
- **H2 stop**: deliverables required when stopping at H2. / 停在 H2 时的交付物要求。

## v5 (2026-10-08)

Fixes from an isolation leak found by the skill's own smoke check, plus acceptance and SOLO clarifications. / 技能自身冒烟检查发现的隔离泄漏修复，以及验收与 SOLO 澄清。

- **Isolation / Codex spawn leak**: Codex exposed agent-spawn (collaboration) tools; document the `model_catalog` fix (`multi_agent_version=null`) and an objective smoke criterion (marker file / stderr; self-reports can only make it stricter), with failure handling and SOLO fallback. / 记录 Codex 暴露 spawn 工具的问题、`model_catalog` 修复与客观冒烟判据。
- **Timeouts & parallel launch**: timeout mechanism and a RUNNING reservation for parallel launches. / 超时机制与并行启动的 RUNNING 预留。
- **SOLO**: who does (i)/(e1)/(e2), `attempts.csv`, Lean dimension unaffected. / SOLO 模式下各职责划分。
- **Acceptance deps**: allow read-only symlinked / mounted prebuilt Mathlib deps in acceptance (not only full copies). / 验收允许只读符号链接/挂载预构建依赖。

## v4 (2026-10-08)

After polishing round 1 (calibration problem with a known answer, refutation problem, multi-theorem / clean-acceptance / forbidden-construct checks, new vague-topic entry). Mapped improvisations agents needed. / 第 1 轮打磨后修订，并映射 agent 临场规则。

- **REFUTED Lean recipe**: explicit `decide` +kernel recipe for refutations. / 明确的 REFUTED Lean 配方。
- **Multi-theorem acceptance build**: fixed lakefile globs (`Challenges.+`) so `lake build Defs Challenges` works. / 修正 lakefile globs，多定理验收可构建。
- **Comparator checks (verified)**: honest 3-theorem challenge including REFUTED form passes; sorry, private axiom, `native_decide`, and statement drift are rejected. / 诚实 3 定理（含 REFUTED）通过；sorry / 私有公理 / native_decide / 陈述漂移被拒。
- **Vague topic**: candidate citations on a new vague topic were all real; improvisation gaps from round 1 mapped into the skill. / 新模糊主题引文全部真实；第 1 轮临场缺口写入技能。

## v3 (2026-10-08)

Revised after a real end-to-end trial run (STANDARD mode, single model family: Codex; 4/4 solver runs returned PARTIAL, the original problem stayed unresolved; Lean + comparator ran on one sub-lemma). 根据一次真实端到端试跑修订。

- **Partial results**: the referee template (d) now ends with two verdict lines, `CLAIMED-SCOPE VERDICT` and `vs-ASSERTION VERDICT`; new main label `REVIEWED(partial)`; author-declared gaps no longer trigger debate. / 审稿拆成两行判词，新增 `REVIEWED(partial)`，作者自认的缺口不再辩论。
- **Stop rules**: every stop rule (S1–S5, chaining limits) now says what happens next; fixed "S1 wrap-up" (one (g1) per route, 2 reviews of the best PARTIAL, optional verification, then the three options). / 每条停止规则写明之后的动作。
- **Isolation**: executable definition (fresh session, file-visibility table per role, no session logs, single agent), a tested bubblewrap wrapper, and a mandatory SEARCH POLICY in the problem statement. / 隔离的可执行定义与检索政策。
- **Statement vs. prompt**: `input.vN.md` holds only the problem statement; solver preamble (a0) and output format (a1) are attached at send time; reviewers get only statement + writeup; PARTIAL / NO_RESULT defined. / 题面与求解提示分开。
- **Lean**: tested setup (Lean/Mathlib v4.34.1; building comparator/lean4export when no matching tag exists; landrun Landlock ABI self-check and best-effort fallback; no-systemd fallback; dev vs. acceptance directories and where acceptance dependencies come from; project layout; build and run commands; multiple theorems), with tested and untested parts listed. / Lean 一节换成实测流程。
- Also: H3 `AGENT_ONLY` / `H3 pending`; per-claim Lean and computation status; call-counting rules and per-phase budget; highest single-agent thinking level and per-call timeouts; signed prompt variants or `diversity=none`; `PASSED(clean-room)` vs `PASSED(sanity)`; trial cost data point; Appendix B (trial summary).

## v2 (2026-10-08)

Initial public release.
