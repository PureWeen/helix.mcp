# Ash — History (Summarized)

## Current Session (2026-05-22)

### Issue #59 Phase 1 Audit — outputSchema + inputSchema deep dive
- **Scope:** Top-10 tools by token cost (4,664 tokens = 58% of total), focusing on PR #1, #2, #4 refactor specs
- **Status:** Complete. Report: `.squad/decisions/inbox/ash-issue59-phase1-audit-2026-05-22.md`
- **Key findings:**

**1. Filter Enum Pattern (PR #1 — inputSchema leverage)**
- Four tools (azdo_timeline, azdo_search_timeline, azdo_helix_jobs, helix_status) define near-identical filter enums with verbose [Description] attributes per tool
- Bloat pattern: ~88 tokens per tool on filter Description alone (e.g., "Filter: 'failed' (default), 'all', 'running' (in-progress tasks), 'pending' (not started), ...")
- Total cost: ~250–300 tokens across 4 tools (3% of total MCP cost)
- **Solution:** Consolidate to shared FilterPreset enum classes; let SDK $ref them instead of duplicating
- **Est. recovery:** −160–320 tokens (realistic: −200 tokens, **2.4% of total**)
- **Risk:** LOW — descriptions are metadata; enum values identical across tools

**2. Redundant Count/Metadata Fields (PR #4 — outputSchema leverage)**
- Pattern: Properties derivable from other fields (totalRecords from records.Count, fileCount from files.Count, etc.)
- Found on 8 of top-10 tools (azdo_timeline, azdo_helix_jobs, helix_find_files, helix_parse_uploaded_trx, etc.)
- Cost: ~50–100 tokens per tool × 8 tools = ~400–800 tokens
- **Solution:** Mark as optional; drop from schema when always-derivable
- **Est. recovery:** −50–150 tokens (conservative)
- **Risk:** LOW — counts are informational; clients can compute

**3. Nested Object Verbosity (PR #4 — outputSchema structural)**
- Deep nesting with liberal nullable fields: TimelineResponse → AzdoTimelineRecord (13 props) → AzdoLogReference + AzdoIssue + AzdoTimelineAttempt (each 2–3 props)
- Most records don't have issues or previousAttempts; schema carries full fluff
- **Solution:** Flatten nullable chains; use sparse patterns or omit rarely-populated branches
- **Est. recovery:** −100–200 tokens
- **Risk:** MEDIUM-HIGH — structural changes can break consumers; mitigate with full test suite + wire-format snapshot

**4. LimitedResults<T> Pattern Maturity (PR #2 — outputSchema design)**
- Top-10 tools already mostly use LimitedResults<T> wrapper (azdo_builds, azdo_test_results, azdo_test_runs, azdo_changes, azdo_artifacts, azdo_test_attachments all use it)
- Partial candidates remain (azdo_helix_jobs uses custom HelixJobsFromBuildResult with aggregation fields)
- **Observation:** Pattern is mature and widely adopted; marginal additional wins < −50 tokens
- **Recommendation:** Defer PR #2 generalization; treat as maintenance task vs. priority optimization
- **Risk:** MEDIUM — forcing custom result types into LimitedResults<T> may reduce semantic clarity

**5. DTO Shape Patterns Across Top-10**
- outputSchema breakdown: avg 65% of tool cost (range 58–69%)
- inputSchema breakdown: avg 23% of tool cost (range 20–26%)
- Annotations: consistently 4–5% of tool cost (already low; Ripley's PR #5 will trim further)
- Top leverages are enum descriptions + nested structure depth + redundant fields (not descriptions)

**Methodology improvement:** Measurement-first audits (gpt-4o tokenizer) are far more valuable than word-count audits. Field-level breakdown (outputSchema %, inputSchema %, etc.) prevents regressions like PR #56's schema growth canceling PR #57's text trims. Recommend standing practice: every measurement PR includes field-level breakdown.

**Recommended Ripley execution order:**
1. PR #1 (Filter enum consolidation): 2–3 hours, −200 tokens (quickest win, low risk)
2. PR #4 (outputSchema DTO-trim on high-ROI tools): 8–10 hours, −300–600 tokens (biggest lever, medium risk)
3. PR #2 (LimitedResults<T> audit): 3–4 hours, −50–150 tokens (if Phase 1 recommends; mostly already applied)

**Phase 1 effort:** 9–13 hours audit + Phase 2 13–17 hours implementation = 22–30 hours, −550–950 tokens expected (6.7–11.6% of total).

---

### Slop Audit (Code Quality)
- **Scope:** 28,813 LOC across 5 source dirs (MCP tools, Core, Host, Generators, Tests)
- **Status:** Complete. Report: `.squad/decisions/inbox/ash-slop-audit-2026-05-22.md`
- **Key findings:**
  1. Result DTO duplication: Program.cs duplicates McpToolResults.cs for 6 JSON result classes (StatusWorkItem, StatusJobInfo, FileInfo_ pairs) — 60 LOC recoverable
  2. Repetitive catch-throw boilerplate: 16 identical exception handlers across AzdoMcpTools + HelixMcpTools (same exception filter + McpException wrapper) — 16 LOC + 1.5h refactor effort
  3. JSON attribute inconsistency: Some properties use inline `[JsonPropertyName]` attributes, others lack them or use different placement patterns (48 occurrences)
  4. Schema drift monitors: Service-layer models vs MCP result types intentionally separated; not slop (justified for API stability)

**Severity breakdown:** 3 HIGH (DTO duplication, catch-throw pattern, JSON consistency), 1 MEDIUM (Program.cs size—6 result classes), 1 LOW (unused imports)

**Codebase health:** B+ (Lean post-PR #57; targeted cleanup opportunities remain)

**Metrics:** 28,813 LOC baseline, 0.3% duplication rate, 0 TODO/FIXME comments, no test slop detected.

### MCP Tool Description Audit
- **Scope:** 25 MCP tools across 3 assemblies (HelixTool.Core, HelixTool.Mcp.Tools, HelixTool)
- **Status:** Complete
- **Finding:** 8 tools flagged for description tightening (~69 words recoverable)

**Flagged tools (by pattern):**
1. **Situational bloat** (3): azdo_timeline, azdo_helix_jobs, helix_status — filter enums duplicated in tool + param descriptions
2. **Schema dump** (2): helix_status, helix_files — lead with schema details instead of action verb
3. **Domain knowledge** (2): azdo_test_results, helix_ci_guide — repo-specific context belongs in knowledge tool
4. **Parameter detail** (1): azdo_builds — defaults belong in param description

**Top offenders:**
- `azdo_timeline` (44 words) → tighten to ~20 words, move filter list to parameter
- `azdo_helix_jobs` (31 words) → same pattern
- `azdo_builds` (30 words) → defer defaults to parameters
- `helix_ci_guide` (26 words) → move repo list to knowledge doc

**Effort:** Ripley should tighten ~1–2 hours (8 tools × ~10 min per tool + validation)

**Next:** Ripley now executing on branch `feat/mcp-description-tightening`

**2026-05-22 Team Update:** Ripley completed description tightening (136 words recovered, 8 tools), Lambert fixed assertion coupling, Dallas reviewed and merged PR #57 to main at 3c4728c. Audit baseline established in decisions.md; follow-ups flagged (quarterly audit cadence, azdo_builds cross-reference restoration).

---

## Prior Work (2026-02-13 through 2026-05-21)

### Key Accomplishments
- **Requirements extraction:** 30 user stories from organic investigation (18 from session 72e659c1, 12 from ci-analysis deep-dive)
- **P0 complete:** Testability (US-12), error handling (US-13)
- **Security:** STRIDE threat model, AzDO security review (6 findings, 1 code fix: query injection)
- **Architecture:** Layered CI diagnosis stack documented. hlx replaceable in 85% of ci-analysis workflows
- **Auth UX:** Helix auth feasibility analysis (Phase 1: git credential storage approved)
- **Audits:** XXE DtdProcessing verification, cross-repo test patterns, AzDO test run metadata reliability

### Historical Context
> See `history-archive.md` for detailed notes on:
> - CI repo profile analysis (cross-repo test pattern variance)
> - Helix authentication UX design (2026-03-03)
> - AzDO security review (2026-03-08)
> - MCP exception audit findings (2026-05-21)

Full text preserved in archive. Current history focuses on active/recent work.
- [2026-05-22] v0.7.3 shipped (PR #56 + PR #57 → main → NuGet)

## 2026-05-22 — Slop Audit Complete, PR #58 Merged

**Status:** Audit delivered → Dallas triage → Ripley implementation → PR #58 merged to main

Result DTO consolidation PR (refactor/consolidate-result-dtos) successfully merged. Findings from slop audit implemented:
- Consolidated 6 duplicate result classes from Program.cs into McpToolResults.cs
- Standardized [JsonPropertyName] attribute placement
- JSON wire-format verified; 1292/1292 tests passing
- No breaking changes to CLI or MCP tool outputs

Catch-throw boilerplate extraction deferred to Q3 2026 per Dallas risk assessment.

---

## Learnings — Silent MCP Failure Investigation 2026-05-25

### Pattern Identified: MCP Tool Parameter Name Inconsistency

**Discovery:** Live agent incident — two tools returned `success=False, result=None` with no error message (build 2983716, dnceng/internal, 16:23–16:27Z).

**Root Causes:**
1. **Parameter naming mismatch** — azdo_build and azdo_build_analysis use `buildId`, but SearchLog/SearchTimeline use `buildIdOrUrl`. Agents calling with wrong parameter name receive silent failure from MCP SDK (parameter binding fails before method entry).
2. **Uncaught AggregateException** — Service layer uses Task.WhenAll without catching AggregateException. If concurrent API calls fail, exception propagates uncaught → MCP SDK returns generic failure with null error message.
3. **Guard clause gaps** — Catch clauses use `when (ex is ...)` guards that only match specific types; AggregateException, TaskCanceledException, and other async exception wrappers fall through.

**Swallow Points:**
- Primary: MCP SDK's parameter binding layer (external; not in hlx code) — when buildIdOrUrl passed but buildId expected, SDK returns error without clear context
- Secondary: Uncaught AggregateException in AzdoMcpTools catch clauses (lines 35–42, 370–376 — identical pattern)
- Tertiary: Service layer Task.WhenAll patterns (AzdoService.cs:594) don't wrap concurrent failures

**Relationship to Deferred Finding #2:** Not the direct cause (boilerplate is repetitive catch-throw, not exception gap), but **implicated in exception policy**. Boilerplate deferral prevents standardization that would catch AggregateException. If we had centralized exception handling, this gap would be visible and fixed.

**Systemic vs. Isolated:**
- Isolated: Parameter naming affects only 3–4 tools; AggregateException affects 8–10 async patterns
- Not systemic: 27 tools audited; most have proper try/catch → McpException
- Instructs future: Task.WhenAll patterns need explicit exception wrapping; parameter naming needs consistency pass

**Recommendation:** 
- Fix 1 (standardize parameter names): 1–2 hours, LOW risk
- Fix 2 (catch AggregateException): 2 hours, MEDIUM risk
- Fix 3 (centralize exception handling): 3–4 hours, ties to deferred Finding #2; requires Dallas policy decision
- Test: Force concurrent API failures; verify error messages non-empty and actionable

**Findings Report:** `.squad/decisions/inbox/ash-silent-mcp-failures-2026-05-25.md`

### Impact Assessment
- **User-facing:** Agents see `success=False` with no actionable error; forced manual recovery via shell scripting
- **Frequency:** Likely rare (parameter mismatch requires specific agent pattern; AggregateException requires concurrent failures)
- **Severity:** MEDIUM — silent failures make debugging difficult; not data corruption or security issue
- **Recovery cost:** ~3 turns for agent to work around; manual CLI as fallback

---

## 2026-05-25: Issue #61 — Silent MCP Failures Investigation (LESSON: Exception Naming by Exercise)

**Session:** 9de92b14-4b97-43d3-a69d-8427b2a8c3d1 (Live agent incident)  
**Status:** Analysis Complete; 3 PRs merged; 2 bugs fixed  
**Outcome:** Identified Bug A (parameter inconsistency) + Bug B (uncaught exceptions) → Both fixed by Ripley

### Key Calibration Learning

**Name an exception by exercising it (10-line repro), not by guessing from source-read.**

Your investigation correctly identified the gap (exceptions escaping catch clauses) and the right fix (centralize exception handling). However, the narrative — "AggregateException from Task.WhenAll is uncaught" — was incorrect.

**Fact:** `await Task.WhenAll(t1, t2)` unwraps to the **first inner exception** (e.g., HttpRequestException). It does NOT throw AggregateException. Only `.Wait()` or `.Result` throws AggregateException. The actual uncaught types were TaskCanceledException and OperationCanceledException.

**Better practice for future investigations:**
1. Write a 10-line repro that forces the failure path
2. Observe what `catch (Exception ex) { Console.WriteLine(ex.GetType()); }` actually catches
3. Only then name the type in your narrative

This is especially critical for Task.WhenAll, Task.WhenAny, ConfigureAwait — the await machinery has non-obvious unwrapping behavior that differs from `.Wait()` / `.Result` thinking.

**Net impact:** Narrative error (cosmetic); zero production risk. The fix was still correct (catch-all pattern catches everything). Dallas verified this and updated the narrative.

### Issue #61 Closed — 3 PRs Merged

- PR #62: Standardize `buildIdOrUrl` parameter (Bug A) ✅
- PR #64: Centralize MCP exception handling (Bug B) ✅
- PR #63: Exception coverage audit + tests (baseline) ✅

Both bugs fixed. Follow-up issue #65 tracks remaining work (schema test, unskip tests, rolling exception coverage, preserve this calibration lesson).


## 2026-05-28: Issue #67 Silent MCP Failures Investigation

- Analyzed 4 parameter binding failures in session b11893eb
- Root cause: Microsoft.Extensions.AI.AIFunctionFactory parameter marshalling failures before tool method invocation
- Classified as Class A (SDK layer) failures vs Class B (runtime) and Class C (schema drift)
- Generated comprehensive investigation document with reproduction steps
- Input to Dallas's CallToolFilters middleware policy decision (resolved in PR #69)
- Recommended schema audit to prevent future parameter naming confusion (implemented by Lambert in PR #68)
