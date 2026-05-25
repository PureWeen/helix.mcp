# Orchestration Log: Dallas (Merge Gate Review — PR #62, #63, #64)

**Date:** 2026-05-25T17:56:16Z  
**Agent:** dallas (claude-opus-4.6, background — reviewer gate)  
**Issue:** #61 — Three-PR merge gate (Bug A, Bug B, exception coverage)  
**Status:** APPROVED & MERGED ✅

## Technical Verification — `await Task.WhenAll` Exception Behavior

**Copilot reviewer is CORRECT.** Verified via C# repro (dotnet 11 preview):

```csharp
=== Two faulting tasks ===
Caught type: System.Net.Http.HttpRequestException  ← NOT AggregateException
Is AggregateException: False

=== .Wait() instead of await ===
Caught type: System.AggregateException  ← only with .Wait()/.Result
```

**Key findings:**
1. `await Task.WhenAll(t1, t2)` unwraps to the **first inner exception** (HttpRequestException, etc.). NOT AggregateException.
2. Only `.Wait()` or `.Result` throws AggregateException. Codebase uses `await` exclusively.
3. `Task.Exception` IS AggregateException (for logging), but `await` unwraps it before reaching catch.

## Impact on Ash's Analysis

- ✅ **Correctly identified:** gap (exceptions escaping catch clauses) and the right fix (centralized exception handling)
- ❌ **Incorrectly named:** "AggregateException from Task.WhenAll is uncaught" — `await` unwraps to inner exception
- **Actual uncaught family:** TaskCanceledException / OperationCanceledException (now fixed in PR #64)

## Impact on PR #64

- `AggregateException` unwrap logic at line 55 is **defensive dead code** in the common `await` path
- Harmless and architecturally sound (guards against future `.Wait()` callers)
- Won't fire under normal production usage; doesn't reduce fix value

## Per-PR Review Verdicts

### PR #62 (Ripley) — Standardize `buildId` → `buildIdOrUrl`
- **Verdict: APPROVE & MERGE** ✅
- Low blast radius (parameter schema only)
- 9 AzDO tools standardized
- CI green; no regressions
- Follow-up filed: Schema metadata test (Issue #65)

### PR #63 (Lambert) — Exception Coverage Audit + 3 Tests
- **Verdict: APPROVE WITH FOLLOW-UP** ✅
- Baseline matrix audit is excellent work (25 tools, 28% unhappy-path floor documented)
- 1 active test (HttpRequestException) passes; validates pattern
- 2 skipped tests correctly annotated as pending #64
- Semantic concern valid: skipped tests manufacture AggregateException, not the real `await Task.WhenAll` path
- Follow-up: Add companion test for real production path (Issue #65)

### PR #64 (Ripley) — Centralize MCP Exception Handling
- **Verdict: APPROVE & MERGE** ✅
- Core fix correct: replaces 16 scattered catch-when blocks
- Adds TaskCanceledException and OperationCanceledException (actual production fix)
- AggregateException unwrap is defensive dead code; harmless
- catch (McpException) re-throw pattern correct
- CI green; no regressions
- Follow-up filed: Flatten InnerExceptions when multiple exceptions exist (Issue #65)

## Merge Sequencing

**Order: #62 → #64 → #63** (all executed successfully)

1. **#62 first** — Independent parameter rename; lowest risk
2. **#64 second** — Provides McpExceptionHandler for Lambert's skipped tests to depend on
3. **#63 last** — After #64 merges, Lambert can unskip tests in follow-up

## Calibration Learning: Exception Naming by Exercise, Not Inspection

**Critical insight:** Name the exception by exercising it (10-line repro), not by guessing from source-read.

**What happened:**
- Ash inferred "AggregateException from Task.WhenAll" from source code reading
- Did NOT actually run `await Task.WhenAll` with faulted tasks and observe what `catch` sees
- Narrative followed the `.Wait()` mental model, not the `await` model

**Better practice for future investigations:**
1. Write 10-line repro that forces failure path
2. Run it and observe: `catch (Exception ex) { Console.WriteLine(ex.GetType()); }`
3. Only then name the type in root-cause analysis

**This is especially critical for:**
- Task.WhenAll, Task.WhenAny (unwrapping behavior)
- ConfigureAwait(false) edge cases
- Async/await machinery non-obvious exception handling

**Net impact:** Cosmetic narrative error (AggregateException vs TaskCanceledException). Zero production risk. Fix is correct regardless (catch-all pattern catches everything). **This lesson should be preserved prominently in squad history.**

## Status

✅ All three PRs merged successfully.
✅ Issue #61 ready for close (both Bug A and Bug B fixed).
✅ Issue #65 filed with follow-up items (schema test, flatten inner exceptions, unskip tests, calibration learning).

## Follow-ups

- Issue #65: 5 follow-up items filed (schema test, flatten exceptions, unskip tests, rolling exception-coverage tests, calibration learning)
