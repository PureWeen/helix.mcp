# Archived Decisions — Before 2026-05-21

Archived on 2026-05-28 (decisions.md exceeded 51200 bytes; archiving >7 days old entries).

---

# Decisions
# MCP Exception Audit — Findings and Recommendations
# US-31: hlx_search_file Phase 1 Implementation
# Decision: Status filter refactored from boolean to enum-style string
# US-32: TRX Parsing Implementation Notes
# Decision: Status Filter Test Coverage Strategy
# Decision: Timeline search result types live in Core
# Decision: Test Quality Review — Tautological Test Findings
# Dallas decisions inbox — Discoverability review (2026-03-10)
# MCP SDK v1.0.0 → v1.3.0 Upgrade Research
# Ripley — MCP SDK 1.3.0 upgrade & CPM migration
# MCP SDK 1.1.0–1.3.0 Adoptable Features Evaluation
# Decision: MCP tool annotations + NU1507 cleanup batch (PR #47)
# Decision: MCP progress notifications via auto-injected `IProgress<T>`
# Decision: SDK 1.0.0 → 1.3.0 upgrade verdict (Lambert)
# Decision: Parallel squad work should use separate git worktrees
# Instead of:
# ... Ripley works ...
# ... Ripley works ...
# Use:
# Each agent works in isolation
# Cleanup: git worktree remove /path/to/worktree-N
# Dependency Audit Recommendations — v0.7.1 vs v0.8.0
# Decision drop: azdo_auth_status is not sync-safe

> Shared team decisions — the single source of truth for architectural and process choices.

### 2026-05-21: MCP Exception Audit

**Date:** 2026-05-21  
**Analyst:** Ash (Product Analyst)  
**Scope:** src/HelixTool.Mcp.Tools/ — all 27 MCP tool methods  
**Status:** Complete — ready for Ripley implementation

---

## Executive Summary

**27 MCP tools audited. Exception handling is EXCELLENT.** Only 1 minor issue identified (helix_ci_guide needs try/catch wrapper). All other 26 tools correctly use McpException pattern.

- **27 total tools:** 10 Helix, 14 AzDO, 1 CiKnowledge
- **26 tools (96%):** Pre-wrapped, follow McpException pattern ✅
- **1 tool (4%):** Raw exception propagation ⚠️ **helix_ci_guide**
- **0 tools:** Implicit uncaught exceptions, swallowed exceptions, or mixed patterns

---

## Audit Findings

### Single Issue: helix_ci_guide (CiKnowledgeTool.cs:11–20)

**Current code:**
```csharp
[McpServerTool(Name = "helix_ci_guide", Title = "CI Investigation Guide", ...)]
public string GetGuide(
    [Description("Repository name; omit for overview")] string? repo = null)
{
    if (string.IsNullOrWhiteSpace(repo))
        return CiKnowledgeService.GetOverview();

    return CiKnowledgeService.GetGuide(repo);  // ⚠️ LINE 19 — NO TRY/CATCH
}
```

**Problem:**
- CiKnowledgeService.GetGuide(repo) can throw ArgumentException or other exceptions
- No try/catch wrapper → raw exceptions bubble to JSON-RPC layer
- LLM agents see stack traces instead of actionable error messages

**Recommended fix:**
```csharp
try
{
    if (string.IsNullOrWhiteSpace(repo))
        return CiKnowledgeService.GetOverview();

    return CiKnowledgeService.GetGuide(repo);
}
catch (Exception ex)
{
    throw new McpException($"Failed to get CI guidance: {ex.Message}", ex);
}
```

**Effort:** Trivial (5-line change, add try/catch wrapper)

**User-facing impact:** Low (CiKnowledgeTool is informational, not critical path), but should still be fixed for consistency with other 26 tools.

---

## Pattern Analysis: What's Working Well

### Pattern 1: Broad Exception Catch (Most Common)
Used in 20+ tools (helix_status, helix_logs, helix_files, helix_download, helix_find_files, helix_work_item, helix_search, helix_parse_uploaded_trx, helix_batch_status, azdo_builds, azdo_timeline, azdo_log, azdo_changes, azdo_test_runs, azdo_test_results, azdo_artifacts, azdo_search_log, azdo_search_timeline, azdo_test_attachments, etc.)

```csharp
try
{
    var result = await _svc.SomeOperationAsync(...);
    return result;
}
catch (Exception ex) when (ex is HttpRequestException or HelixException or RestApiException or InvalidOperationException or ArgumentException)
{
    throw new McpException($"Failed to {action}: {ex.Message}", ex);
}
```

**Strengths:**
- ✅ Catches all expected service-layer exceptions
- ✅ Re-throws as McpException with actionable message
- ✅ Preserves original exception as InnerException for debugging
- ✅ Consistent across all tools

### Pattern 2: Context-Specific Error Detection (3 tools)
Used in azdo_build, azdo_helix_jobs, azdo_build_analysis

```csharp
try
{
    return await _svc.GetItemAsync(id);
}
catch (InvalidOperationException ex) when (ex.Message.Contains("not found", StringComparison.OrdinalIgnoreCase))
{
    throw new McpException(AppendNotFoundHint(ex.Message, id), ex);
}
catch (Exception ex) when (ex is InvalidOperationException or HttpRequestException or ArgumentException)
{
    throw new McpException($"Failed to {action}: {ex.Message}", ex);
}
```

**Strengths:**
- ✅ Detects semantic error condition ("not found")
- ✅ Appends contextual hints (auth suggestions, URL format guidance)
- ✅ Falls through to broad catch for other InvalidOperationException instances
- ✅ User-facing message is actionable: "Build not found in org/project. This org may require auth..."

### Pattern 3: Pre-Call Parameter Validation (All tools)
Used in all 27 tools before calling service layer

```csharp
if (string.IsNullOrEmpty(requiredParam))
    throw new McpException("Parameter 'requiredParam' is required.");

if (!filter.Equals("failed", StringComparison.OrdinalIgnoreCase) && ...)
    throw new McpException($"Invalid filter '{filter}'. Must be 'failed', 'passed', or 'all'.");
```

**Strengths:**
- ✅ Validates parameters before service calls
- ✅ Clear, actionable error messages listing valid values
- ✅ No ArgumentException leak from service layer

### Pattern 4: Config Guard Checks (8 tools)
Used in helix_search, helix_parse_uploaded_trx, azdo_search_log

```csharp
if (StringHelpers.IsFileSearchDisabled)
    throw new McpException("File content search is disabled by configuration.");
```

**Strengths:**
- ✅ Configuration failures surfaced as McpException
- ✅ No InvalidOperationException leak
- ✅ Clear user message

### Pattern 5: No-Op Methods (2 tools)
Used in helix_auth_status, azdo_auth_status

```csharp
public HelixAuthStatus HelixAuth()
{
    var token = _tokenAccessor.GetAccessToken();
    var hasToken = !string.IsNullOrEmpty(token);
    
    // ... compute state ...
    
    return new HelixAuthStatus { IsAuthenticated = hasToken, Source = source };
}
```

**Rationale:**
- ✅ Synchronous, no I/O or parsing
- ✅ Only accesses in-memory token accessor state
- ✅ No exception wrapping needed (correctly identified)
- ⚠️ Note: azdo_auth_status is currently async but does no I/O; consider making it sync

---

## Recommended McpException Pattern for Ripley

When implementing the helix_ci_guide fix or writing new tools, use these patterns:

### Template 1: Simple Service Call
```csharp
[McpServerTool(Name = "your_tool", ...)]
public async Task<YourResult> YourMethod(string param)
{
    if (string.IsNullOrEmpty(param))
        throw new McpException("Parameter 'param' is required.");
    
    try
    {
        var result = await _service.GetDataAsync(param);
        return new YourResult { /* ... */ };
    }
    catch (Exception ex) when (ex is HttpRequestException or ServiceException or InvalidOperationException or ArgumentException)
    {
        throw new McpException($"Failed to get data: {ex.Message}", ex);
    }
}
```

### Template 2: Context-Specific Error + Broad Fallback
```csharp
try
{
    return await _service.GetItemAsync(id);
}
catch (InvalidOperationException ex) when (ex.Message.Contains("not found", StringComparison.OrdinalIgnoreCase))
{
    // Add actionable context
    throw new McpException($"{ex.Message} Try using azdo_builds with org='dnceng' for internal builds.", ex);
}
catch (Exception ex) when (ex is InvalidOperationException or HttpRequestException or ArgumentException)
{
    throw new McpException($"Failed to get item: {ex.Message}", ex);
}
```

### Template 3: Synchronous No-Op
```csharp
public AuthStatusResult AuthStatus()
{
    // No try/catch needed — no I/O or parsing
    var token = _tokenAccessor.GetAccessToken();
    return new AuthStatusResult { IsAuthenticated = !string.IsNullOrEmpty(token) };
}
```

---

## Error Message Guidelines

Follow BinlogMcp's established pattern (from decisions.md):

**Format:**
```
"Failed to {action}: {ex.Message} [optional: contextual hint]"
```

**Examples:**

✅ **Good:**
- "Failed to get build status: Build not found in dnceng-public/public. If this is an internal build, use org='dnceng' and project='internal' (requires auth via 'az login' or AZDO_TOKEN)."
- "Failed to download files: No files matching '*.binlog' found."
- "Failed to search: File content search is disabled by configuration."
- "Work item name is required. Provide it as a separate parameter or include it in the Helix URL."

❌ **Bad:**
- "An error occurred." (too vague)
- "System.NullReferenceException: Object reference not set to an instance of an object." (exposes internals)
- "Failed to get CI guidance." (no context about what went wrong)

---

## Cross-Check: BinlogMcp Pattern Alignment

From .squad/decisions.md (2026-03-13):
> "Use `McpException` for tool-surface failures and keep JSON property names/wire format stable even when internal C# type names change."

**HelixTool.Mcp.Tools compliance:**
- ✅ Tool-level errors (missing work item, no files, build not found) → McpException
- ✅ Parameter validation (invalid filter) → McpException with valid enum values listed
- ✅ Config failures (file search disabled) → McpException
- ✅ Service layer exceptions (HttpRequestException, HelixException) → wrapped as McpException
- ✅ No raw .NET exceptions escape to JSON-RPC (except helix_ci_guide)
- ✅ JSON property names stable (CamelCase in McpToolResults.cs, independent of C# class names)

---

## Summary of Action Items

### For Ripley (Implementation)

**Task 1: Fix helix_ci_guide**
- File: src/HelixTool.Mcp.Tools/CiKnowledgeTool.cs
- Lines: 11–20
- Change: Wrap GetGuide() call in try/catch → throw McpException
- Effort: Trivial (5 lines)
- Priority: P2 (low user impact, but maintains consistency)

### For Dallas (Architecture Review)

**Decision 1: helix_ci_guide error strategy**
- Should CiKnowledgeService validate repo names before lookup, or rely on wrapper catch-all?
- **Recommendation:** Do both — validate in service (fail fast) + wrap in tool (belt & suspenders)

**Decision 2: azdo_auth_status async-ness**
- Currently async but does no I/O. Should it be made synchronous?
- **Recommendation:** Yes, make synchronous (simpler, clearer intent)

**Decision 3: Future structured error codes**
- All tools currently use `ex.Message` only. Should we add error codes (e.g., "BUILD_NOT_FOUND") for LLM agent routing?
- **Recommendation:** P2 feature — improve error categorization later if agent routing becomes complex

### For Lambert (Testing)

**Test impact:** Existing test coverage already validates exception wrapping on all 26 tools. After Ripley's fix, add test case for helix_ci_guide exception wrapping (test that invalid repo name throws McpException, not raw ArgumentException).

---

## Skill Extraction: MCP Exception Audit Methodology

**Reusable process for auditing any MCP tool set:**

1. **Enumerate all tools** using grep `[McpServerTool` decorator
2. **Classify exception posture** by walking each method body:
   - Parameter validation (pre-call) → McpException
   - Service/I/O calls (try/catch) → wrapped or raw propagation
   - Config guards → McpException or error object
   - Return paths → exceptions or success results
3. **Categorize by pattern:**
   - **A:** Pre-wrapped (has try/catch → McpException)
   - **B:** Raw throw (no try/catch, raw exceptions propagate)
   - **C:** Implicit (no explicit throw, but calls can fail uncaught)
   - **D:** Swallowed (caught and returned as error object, not thrown)
   - **E:** Mixed (multiple strategies in same method)
4. **Score by user impact × ease of fix:**
   - User-facing visibility: How does end user see the exception?
   - Fix complexity: Trivial (add wrapper), Moderate (add validation), Complex (refactor service layer)
   - I/O-bound vs logic: Network/file I/O has broader exception surface
5. **Recommend by effort, not by tool:**
   - Group fixes by effort level (Trivial, Moderate, Complex)
   - Enables parallelized implementation

**Output:** Inventory table (tool name, file:line, posture, exception types, fix complexity) + summary by effort band.

---

## Appendix: Full Tool Inventory

**27 Tools Total:**

**Helix Tools (10):**
1. helix_status — Pre-wrapped ✅
2. helix_logs — Pre-wrapped ✅
3. helix_files — Pre-wrapped ✅
4. helix_download — Pre-wrapped ✅
5. helix_find_files — Pre-wrapped ✅
6. helix_work_item — Pre-wrapped ✅
7. helix_search — Pre-wrapped ✅
8. helix_parse_uploaded_trx — Pre-wrapped ✅
9. helix_batch_status — Pre-wrapped ✅
10. helix_auth_status — No wrapping needed (synchronous, no I/O) ✅

**AzDO Tools (14):**
11. azdo_build — Pre-wrapped with context-specific handling ✅
12. azdo_builds — Pre-wrapped ✅
13. azdo_timeline — Pre-wrapped ✅
14. azdo_log — Pre-wrapped ✅
15. azdo_changes — Pre-wrapped ✅
16. azdo_test_runs — Pre-wrapped ✅
17. azdo_test_results — Pre-wrapped ✅
18. azdo_artifacts — Pre-wrapped ✅
19. azdo_search_log — Pre-wrapped ✅
20. azdo_search_timeline — Pre-wrapped ✅
21. azdo_test_attachments — Pre-wrapped ✅
22. azdo_helix_jobs — Pre-wrapped with context-specific handling ✅
23. azdo_build_analysis — Pre-wrapped with context-specific handling ✅
24. azdo_auth_status — No wrapping needed (synchronous, no API call) ✅

**CI Knowledge Tool (1):**
25. helix_ci_guide — **Raw exceptions propagate** ⚠️ **Needs fix**

---

**Status:** Ready for Ripley to implement helix_ci_guide fix. All other tools verified and approved.

### 2026-05-21: HelixTool RollForward Policy

- Use `<RollForward>Major</RollForward>` in `src/HelixTool/HelixTool.csproj`.
- **Not chosen:** `LatestMajor`, because we want conservative behavior that prefers the exact target runtime when it is installed and only moves to the lowest higher major when the target major is missing.
- **Rationale:** `hlx` is shipped as a global `dotnet tool`, so its generated runtimeconfig controls whether the executable starts on machines that only have a newer shared framework installed. `Major` allows `net10.0` to run on .NET 11+ when .NET 10 is absent, avoiding startup failures for both the CLI and `hlx mcp serve`.
- **Scope:** This applies only to the executable project `HelixTool`. Library projects do not produce the tool runtimeconfig and should not be changed for this policy.


**By:** Ripley

**By:** Ripley

**By:** Ripley  

**By:** Lambert (Tester)

**By:** Ripley


1. **Do not add a new composite failure-investigation tool in this increment.**
   Improve discoverability through existing surfaces: MCP tool descriptions, fallback/error messages, `helix_ci_guide`, README, and llmstxt/help output.

2. **Make `helix_ci_guide(repo)` the recommended repo-specific entry point for workflow selection.**
   Tool descriptions and failure messages should direct agents there when pattern choice or result-location expectations vary by repo.

3. **Clarify the behavioral contract of `helix_test_results`.**
   It should be described as structured Helix-hosted test-result parsing with existing fallback support, but not as the universal first choice across repos. Failure guidance must route callers to the correct next tool sequence.

4. **Clarify the behavioral contract of `helix_search_log`.**
   It should be positioned as the preferred remote-first console-log search path, with explicit note that search patterns vary by repo/test runner.

5. **Keep discoverability surfaces synchronized.**
   MCP descriptions, README, llmstxt/help output, and CI-guide wording must align on when to use `helix_test_results`, when to pivot to AzDO structured results, and when to use `helix_search_log`.


**Requested by:** Larry Ewing  
