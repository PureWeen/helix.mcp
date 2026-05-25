# Orchestration Log: Dallas (Bug B Re-triage Decision)

**Date:** 2026-05-25T17:56:16Z  
**Agent:** dallas (claude-haiku-4.5, background)  
**Issue:** #61 — Bug B re-triage (promote Finding #2 to FIX NOW)  
**Status:** DECISION DELIVERED ✅

## Executive Summary

The May 22 deferral of Finding #2 (boilerplate catch-throw pattern, Q3 2026) is **no longer valid.** Ash's investigation revealed production-impact evidence: azdo_build_analysis and ~8–10 async sites throw exceptions NOT in existing catch-when guards, causing silent failures in production.

**VERDICT: PROMOTE Finding #2 to FIX NOW** — 3–4h Ripley work to centralize exception handling.

## Decision Reasoning

1. **User-visible production bug confirmed** — Not hypothetical (Ash reproduced in live session 9de92b14)
2. **Exception types NOT caught:** AggregateException, TaskCanceledException missing from filters
3. **Centralization is lower risk** — Scattered catch blocks invite inconsistency; centralization enforces consistency
4. **Test coverage is NOT a blocker** — User-visible bug justifies proceeding; Lambert adds tests in parallel (non-critical path)
5. **Timeline is predictable** — 3–4h effort: define helper, replace 16 catches, test

## May 22 Deferral Calibration

**Why May 22 deferral was defensible:**
- No documented user-visible impact
- Extraction risk real without test evidence
- Precondition ("wait for test coverage") was sound

**Why it's no longer valid:**
- Ash's investigation provides test evidence (exception types not caught)
- Test coverage is now a parallel validation gate, not a prerequisite

## Implementation Plan (For Ripley)

- Extract `WrapServiceException` helper in AzdoMcpTools
- Replace 16 repetitive catch blocks
- Add TaskCanceledException, OperationCanceledException to known types
- Test: mock service to throw each exception type, verify McpException wrapping

## Sequencing (Issue #61 Close Target: EOW 2026-05-29)

**Week 1:**
- Bug A PR (#62): land first (lower risk, 1–2h) — MERGED ✅
- Bug B PR (#64): land second (higher scope, 3–4h) — MERGED ✅
- Both PRs merge by Thu/Fri; issue #61 closes when B merges

**Parallel (non-critical path):**
- Lambert: exception-coverage audit baseline (Week 1) — MERGED ✅
- Lambert: write missing exception tests (Week 2–3, after #64 merges)

## Standing Rule (Lambert owns, separate ceremony)

**Policy:** Every MCP tool method must have ≥1 test covering the unhappy path (exception → structured error).

- Week 2: Lambert audits baseline coverage (completed with PR #63)
- Week 3–4: Lambert writes missing tests in rolling PR

## Status

✅ Decision finalized. Both PRs implemented by Ripley and merged successfully. Issue #61 ready for close.

## Follow-ups Filed

- Issue #65: Schema metadata test for buildIdOrUrl parameter
- Issue #65: Flatten AggregateException inner exceptions
- Issue #65: Lambert unskips 2 skipped tests after #64
- Issue #65: Calibration learning for exception naming by exercise (documented in this batch)
