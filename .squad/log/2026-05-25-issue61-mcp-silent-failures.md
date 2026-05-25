# Session Log: Issue #61 Silent MCP Failures — 6-Agent Pipeline

**Date:** 2026-05-25  
**Batch:** Issue #61 Close  
**Duration:** Ash (2026-05-25 16:23–16:27Z) → Dallas gate (2026-05-25 final review)  
**Team:** 6 agents in 2 decision gates + 1 merge gate

---

## Executive Summary

Silent MCP failures in live session (9de92b14-4b97-43d3-a69d-8427b2a8c3d1) triggered a 6-agent investigation and fix pipeline. Root causes: (1) parameter name inconsistency (Bug A), (2) uncaught exception types (Bug B). Result: 3 PRs merged, 2 bugs fixed, 1 follow-up issue filed, 1 calibration lesson preserved.

---

## 6-Agent Pipeline

### Phase 1: Investigation (Ash)

**Agent:** ash (claude-haiku-4.5, background)  
**Task:** Investigate silent failures from live session  
**Duration:** 2026-05-25 16:23–16:27Z  
**Output:** Root cause analysis (parameter + exception gaps) + fix recommendations  

**Key findings:**
- Bug A: buildId vs buildIdOrUrl parameter name mismatch (9 tools affected)
- Bug B: AggregateException and TaskCanceledException not in catch clauses (~8–10 async methods affected)
- Relationship to Finding #2 (deferred boilerplate): the pattern masks gaps

### Phase 2: Decision Gates (Dallas × 2)

**Agent:** dallas-1 (claude-haiku-4.5, background)  
**Task:** Issue #59 triage + re-triage (May 22 context)  
**Decision:** Hold deferral May 22; revisit if production silent failures reproduced

**Agent:** dallas-2 (claude-opus-4.6, background)  
**Task:** Issue #61 Bug B re-triage (May 25, in light of Ash's findings)  
**Decision:** PROMOTE Finding #2 to FIX NOW; centralize exception handling (3–4h effort)

### Phase 3: Implementation (Ripley × 2, Lambert × 1, Dallas merge gate)

**Agent:** ripley-1 (claude-sonnet-4.6, background)  
**Task:** PR #62 — Standardize parameter names (Bug A)  
**Duration:** 1–2h  
**Output:** 9 tools renamed buildId → buildIdOrUrl; CI green; MERGED ✅

**Agent:** ripley-2 (claude-sonnet-4.6, background)  
**Task:** PR #64 — Centralize exception handling (Bug B)  
**Duration:** 3–4h  
**Output:** McpExceptionHandler helper; replace 16 catch blocks; CI green; MERGED ✅

**Agent:** lambert (claude-sonnet-4.6, background)  
**Task:** PR #63 — Exception coverage baseline audit + 3 tests  
**Duration:** Parallel to Ripley  
**Output:** 25-tool audit (28% unhappy-path floor); 1 active test, 2 skipped (pending #64); MERGED ✅

**Agent:** dallas-3 (claude-opus-4.6, background)  
**Task:** Merge gate review (PR #62, #63, #64)  
**Duration:** Final technical verification  
**Decision:** All three APPROVED; merge order #62 → #64 → #63
**Technical finding:** Verified `await Task.WhenAll` unwraps to inner exception (not AggregateException); updated analysis narrative

---

## Merge Timeline

| PR | Title | Author | Merge Order | Status |
|---|---|---|---|---|
| #62 | Standardize `buildIdOrUrl` parameter | Ripley | 1st | ✅ Merged |
| #64 | Centralize MCP exception handling | Ripley | 2nd | ✅ Merged |
| #63 | Exception coverage audit + tests | Lambert | 3rd | ✅ Merged |

**All PRs merged within issue #61 batch. Issue ready for close.**

---

## Key Calibration Lesson (Preserved)

**Rule: Name the exception by exercising it, not by guessing from source-read.**

Ash's investigation correctly identified the gap and the right fix (centralized exception handling), but incorrectly named "AggregateException from Task.WhenAll" as the uncaught exception. In reality, `await Task.WhenAll` unwraps to the inner exception; only `.Wait()` throws AggregateException. The actual uncaught types were TaskCanceledException and OperationCanceledException.

**Better practice:**
1. Write 10-line repro
2. Run it and observe: `catch (Exception ex) { Console.WriteLine(ex.GetType()); }`
3. Only then name the type in narrative

This is especially critical for Task.WhenAll, Task.WhenAny, ConfigureAwait — the await machinery has non-obvious unwrapping behavior.

**Impact:** Narrative error (cosmetic); zero production risk. Fix still correct (catch-all pattern catches everything).

**Preservation:** This lesson appended to Ash's, Ripley's, Lambert's, and Dallas's history.md files.

---

## Follow-ups Filed (Issue #65)

1. **Schema metadata test** — Assert buildIdOrUrl parameter in MCP schema (prevents regression)
2. **Flatten AggregateException** — When unwrapping multiple inner exceptions, combine messages instead of taking [0]
3. **Unskip Lambert tests** — After #64 merges, unskip 2 tests + add companion test for real `await Task.WhenAll` path
4. **Rolling exception-coverage tests** — Prioritize unhappy-path tests for high-traffic tools (azdo_build, azdo_helix_jobs, azdo_search_timeline, helix_batch_status)
5. **Calibration learning** — Documented for future exception investigations

---

## Standing Rule (Lambert owns)

**Policy:** Every MCP tool method must have ≥1 test covering the unhappy path (exception → structured error).

- Baseline audit completed (PR #63)
- Rolling implementation: Week 2–3 per Ripley/Lambert's capacity
- CI gate: enforce at PR review time (future work)

---

## Statistics

- **Agents involved:** 6 (Ash, Ripley ×2, Lambert, Dallas ×3)
- **PRs merged:** 3
- **Bugs fixed:** 2 (Bug A + Bug B)
- **Tools affected:** 9 (parameter rename) + ~16 (exception centralization)
- **Lines removed:** ~48 LOC (boilerplate catch blocks)
- **Follow-ups filed:** 5 (Issue #65)
- **Calibration learnings:** 1 (exception naming by exercise)

---

## Session Status

✅ **Issue #61 Complete.** All root causes identified, 3 PRs merged, 2 bugs fixed. Issue ready for close. Follow-up issue #65 tracks remaining work (schema test, unskip tests, rolling exception coverage, calibration learning).

**No unresolved blockers. Issue #61 Closed.**
