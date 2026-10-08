# Changelog

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
