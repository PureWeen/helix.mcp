# Issue #61 Merge Gate — PR #62, #63, #64 Review Verdicts

**Status:** FINAL  
**Reviewer:** Dallas (Lead)  
**Date:** 2026-05-25  

---

## Part 1: Technical Verification — `await Task.WhenAll` Exception Behavior

**Copilot reviewer is CORRECT.** Verified via C# repro (dotnet 11 preview):

```
=== Two faulting tasks ===
Caught type: System.Net.Http.HttpRequestException  ← NOT AggregateException
Is AggregateException: False

=== .Wait() instead of await ===
Caught type: System.AggregateException  ← only with .Wait()/.Result
```

**Key findings:**

1. `await Task.WhenAll(t1, t2)` unwraps to the **first inner exception** (e.g., `HttpRequestException`). It does NOT throw `AggregateException`.
2. Only `.Wait()` or `.Result` throws `AggregateException`. The codebase uses `await` exclusively.
3. `Task.Exception` IS an `AggregateException` (for logging/inspection), but `await` unwraps it.

**Source confirmation:** `AzdoService.GetBuildAnalysisAsync` (line 594) and `SearchBuildLogAcrossStepsAsync` (line 356) both use `await Task.WhenAll(...)` — never `.Wait()`.

**Impact on Ash's analysis:** Ash correctly identified the gap (exceptions escaping the catch clauses) and the right fix (catch-all centralization). But the narrative — "AggregateException from Task.WhenAll is uncaught" — is wrong. In the `await` path, `Task.WhenAll` unwraps to `HttpRequestException` or `InvalidOperationException`, which the original catch clauses **already caught**. The actual uncaught family was `TaskCanceledException` / `OperationCanceledException`, which were missing from the original `when (ex is ...)` filter.

**Impact on PR #64:** The `AggregateException` unwrap logic in `McpExceptionHandler.WrapServiceException` (line 55) is **defensive dead code** in the common production path. It's harmless (guards against `.Wait()` or explicit `AggregateException` wrapping), but won't fire under normal `await` usage. The `InnerExceptions[0]`-only concern raised by Copilot is technically valid but moot since this path rarely executes.

**What ACTUALLY caused session 9de92b14 silent failure:** Most likely Bug A (parameter name mismatch: agent sent `buildIdOrUrl`, method expected `buildId` → MCP SDK parameter binding failure before method entry). Secondary contributor: `TaskCanceledException` not in catch clauses.

---

## Part 2: Per-PR Verdicts

### PR #62 — Rename `buildId` → `buildIdOrUrl` (Bug A)

**Verdict: APPROVE & MERGE**

- Technically correct and clean. 9 AzDO tools standardized.
- Low blast radius — parameter schema change only, no logic change.
- CI green, no regressions.
- **Follow-up filed:** Schema metadata test asserting `buildIdOrUrl` exposure (prevents silent regression to `buildId`). Per Copilot reviewer's valid observation that positional test invocations wouldn't catch a name regression.

### PR #63 — Exception coverage audit + 3 tests (Lambert)

**Verdict: APPROVE WITH FOLLOW-UP**

- Coverage matrix audit is excellent work — baseline documented, gaps identified.
- 1 active test (`Builds_WhenHttpRequestExceptionOccurs`) passes and validates the pattern.
- 2 skipped tests correctly annotated as pending PR #64.
- **Semantic concern (from Copilot reviewer):** The `BuildAnalysis_AggregateException` test uses `Task.FromException(new AggregateException(new HttpRequestException(...)))` — this manufactures an `AggregateException` the mock returns directly, which isn't the path `await Task.WhenAll` takes. When unskipped after #64, the test validates `McpExceptionHandler`'s AggregateException unwrap logic (defensive code), not the real production path. This is acceptable but should be documented.
- **Follow-up:** Add a companion test modeling the REAL `await Task.WhenAll` production path (mock throws `HttpRequestException` directly, not wrapped in `AggregateException`). This was the real exception family at the MCP boundary.

### PR #64 — Centralize MCP exception handling (Bug B)

**Verdict: APPROVE & MERGE**

- Core fix is correct: replaces 16 scattered catch-when blocks with centralized `McpExceptionHandler`.
- Adds `TaskCanceledException` and `OperationCanceledException` to known exception types — THIS is the actual production fix.
- The `AggregateException` unwrap at line 55 is defensive dead code under `await`, but harmless and architecturally sound (guards against future `.Wait()` callers).
- `catch (McpException) { throw; }` re-throw pattern is correct — preserves pre-validated MCP errors from guard clauses.
- CI green.
- **Follow-up filed:** The `InnerExceptions[0]`-only unwrap should flatten to a combined message when multiple inner exceptions exist (per Copilot reviewer). Low priority since the path is defensive, but worth fixing for correctness.

---

## Part 3: Merge Sequencing

**Order: #62 → #64 → #63**

1. **#62 first** — Independent param rename. No dependencies. Lowest risk.
2. **#64 second** — Provides `McpExceptionHandler` that Lambert's skipped tests depend on.
3. **#63 last** — After #64 merges, Lambert can unskip the 2 pending tests in a follow-up PR.

This ordering lets Lambert's `[Skip]` tests unskip cleanly against the merged centralized handler.

---

## Part 4: Analytic Miss — Calibration Learning

### What happened

Ash's investigation correctly identified:
- ✅ The symptom (silent `success=False, result=None`)
- ✅ The fix (centralize exception handling, catch broader families)
- ✅ The root cause of Bug A (parameter name mismatch)

But incorrectly named:
- ❌ "AggregateException from Task.WhenAll is uncaught" — `await` unwraps, so AggregateException never reaches the catch clause
- ❌ The narrative propagated through Lambert's test design (manufactured AggregateException) and Ripley's handler (defensive AggregateException unwrap)

### Why it happened

The exception type was inferred from **source-reading** (`Task.WhenAll` → "must be AggregateException") rather than **exercised** (actually running `await Task.WhenAll` with faulted tasks and observing what `catch` sees). The inference followed the `.Wait()` mental model, not the `await` model.

### Calibration rule for future investigations

**Name the exception by exercising it, not by guessing from source-read.**

Before naming an exception type in a root-cause analysis:
1. Write a 10-line repro that forces the failure path
2. Observe what `catch (Exception ex) { Console.WriteLine(ex.GetType()); }` actually catches
3. Only then name the type in the narrative

This is especially critical for `Task.WhenAll` / `Task.WhenAny` / `ConfigureAwait` — the await machinery has non-obvious unwrapping behavior that differs from `.Wait()` / `.Result`.

### Impact assessment

The wrong narrative did NOT produce a wrong fix. The centralized handler catches everything (`catch (Exception ex)`), which is correct regardless of which exception type actually surfaces. The defensive `AggregateException` unwrap is harmless. **Net impact: cosmetic narrative error, zero production risk.**

---

## Follow-up Issues to File

1. **Schema metadata test for buildIdOrUrl** — Assert each renamed AzDO tool exposes `buildIdOrUrl` in MCP schema, not `buildId`.
2. **Flatten AggregateException inner exceptions** — When `AggregateException` handler fires (defensive path), combine all inner exception messages instead of taking only `InnerExceptions[0]`.
3. **Lambert test follow-up** — Unskip 2 tests after #64 merges; add companion test for real `await Task.WhenAll` exception path.
