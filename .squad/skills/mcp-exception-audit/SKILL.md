# MCP Exception Audit Skill

**Purpose:** Audit a set of MCP tool methods for exception handling hygiene and McpException wrapping compliance.

**Scope:** Any MCP server with tool methods (classes decorated with `[McpServerToolType]` or methods decorated with `[McpServerTool]`).

**Effort:** ~2 hours for a typical MCP tool set (20–30 tools)

**Audience:** Product Analysts performing code quality audits; Architects reviewing exception handling patterns; Implementation teams verifying compliance with established patterns (e.g., BinlogMcp, HelixTool).

---

## Methodology

### Phase 1: Enumeration (15 min)

1. **Locate all tool classes:**
   ```bash
   grep -r "\[McpServerToolType\]" src/
   ```

2. **Count total tools:**
   ```bash
   grep -rn "\[McpServerTool(" src/ | wc -l
   ```

3. **Extract tool metadata (name, file, line):**
   ```bash
   grep -rn "\[McpServerTool(" src/ | cut -d: -f1,2 | sort
   ```

**Deliverable:** List of all tool names, file paths, and line numbers.

### Phase 2: Classification (1.5 hours)

For each tool method, walk the code and classify by **exception posture:**

| Posture | Definition | Pattern | Fix Needed? |
|---------|-----------|---------|------------|
| **A** | Pre-wrapped | Has `try/catch` that throws `McpException` | No |
| **B** | Raw throw | No `try/catch`; raw .NET exceptions propagate uncaught | **Yes** |
| **C** | Implicit | No explicit `throw`, but calls can fail uncaught | **Yes** |
| **D** | Swallowed | Catches and returns error object instead of throwing | Depends on architecture |
| **E** | Mixed | Multiple strategies on different code paths | **Yes** |

**Classification checklist per method:**

```
□ Parameter validation (pre-call)
  - How are required parameters validated?
  - What exception type is thrown for invalid input?
  - Is it McpException or raw ArgumentException?

□ Service/I/O calls (try/catch)
  - Are service calls wrapped in try/catch?
  - What exception types are caught?
  - Are exceptions re-thrown as McpException?

□ Config/guard checks
  - Are configuration failures surfaced as McpException?
  - Or are they silently ignored?

□ Return paths
  - Does the method always throw on error?
  - Or does it return error objects instead?

□ Context-specific handling
  - Are semantic errors (e.g., "not found") detected and enhanced with hints?
  - Or is there a broad one-size-fits-all catch?
```

**Output:** For each tool, record:
- Tool name, file path, line number
- Posture (A, B, C, D, or E)
- Exception types likely to escape (be specific: "HttpRequestException from HelixService.GetJobStatusAsync", not just "network error")
- Whether error message is actionable (user-facing context: yes/no)
- Whether call is I/O-bound (network, file) or logic (validation, parsing)

### Phase 3: Scoring (30 min)

For each tool with **issues** (posture B, C, or E), score by:

**User visibility:**
- 🔴 High: User-critical path, data-facing, frequently-used tool
- 🟡 Medium: Informational, secondary tool, occasionally-used
- 🟢 Low: Config-only, rarely-invoked, edge-case tool

**Ease of fix:**
- ✅ **Trivial:** Add single `try/catch` wrapper at method entry point (1–5 lines)
- ⚠️ **Moderate:** Need argument validation + structured error mapping (HTTP status → message) (5–20 lines)
- ❌ **Complex:** Requires refactoring underlying service layer to surface typed errors first (20+ lines, cross-file changes)

**I/O surface area:**
- **High:** Network calls (HttpClient, HTTP REST), file I/O (File.*, Directory.*), external process invocation
- **Medium:** JSON/XML parsing, service layer calls that wrap I/O
- **Low:** In-memory validation, state queries, local computation

**Recommendations by impact × ease:**
- High impact + Trivial fix → **Fix first** (user-critical, fast implementation)
- High impact + Moderate fix → **Fix second** (user-critical, some effort)
- Medium/Low impact + Trivial fix → **Fix third** (low user impact, fast)
- Complex fixes → **Discuss with architecture** (may indicate service-layer design issue)

### Phase 4: Report & Recommendations (30 min)

**Output:** Comprehensive audit report including:

1. **Executive summary:** Total tools audited, posture breakdown (A/B/C/D/E counts), verdict
2. **Tool-by-tool inventory table:** Name, file:line, class, posture, exception types, user visibility, fix complexity
3. **Detailed findings:** For each tool with issues, explain:
   - Current code snippet
   - Problem description
   - Likely exceptions that escape
   - Recommended fix
   - Effort estimate
4. **Pattern analysis:** Catalog exception patterns used in "good" tools, highlight what works well
5. **Recommended patterns:** Provide 4–5 reusable templates for different tool shapes (simple async, context-specific handling, no-op methods, etc.)
6. **Priority fix list:** Top N tools by impact × ease (typically top 5)
7. **Cross-check with architecture:** Compare observed patterns against team standards (e.g., BinlogMcp, established error-handling decisions)
8. **Open questions:** Decisions for architecture team (should service layer validate or tool layer wrap? Structured error codes or message-only?)

---

## Pattern Catalog

### Pattern 1: Simple Service Call (Most Common)
**Used when:** Single async service call, no special error handling needed.

```csharp
try
{
    var result = await _service.SomeOperationAsync(param1, param2);
    return new ToolResult { /* populate from result */ };
}
catch (Exception ex) when (ex is HttpRequestException or ServiceException or InvalidOperationException or ArgumentException)
{
    throw new McpException($"Failed to {action}: {ex.Message}", ex);
}
```

**Strengths:**
- Catches all expected service-layer exception types
- Re-throws as McpException with user-facing message
- Preserves original exception as InnerException for debugging

### Pattern 2: Context-Specific Error + Broad Fallback
**Used when:** Service can fail with semantic errors (404, "not found") that deserve special handling.

```csharp
try
{
    return await _service.GetItemAsync(id);
}
catch (InvalidOperationException ex) when (ex.Message.Contains("not found", StringComparison.OrdinalIgnoreCase))
{
    throw new McpException(AppendContextualHint(ex.Message, id), ex);
}
catch (Exception ex) when (ex is InvalidOperationException or HttpRequestException or ArgumentException)
{
    throw new McpException($"Failed to get item: {ex.Message}", ex);
}
```

**Strengths:**
- Detects semantic error condition and enhances message
- Appends actionable context (auth hints, URL format guidance)
- Falls through to broad catch for unmatched error types

### Pattern 3: Pre-Call Validation
**Used when:** Tool accepts required or enum-like parameters.

```csharp
if (string.IsNullOrEmpty(requiredParam))
    throw new McpException("Parameter 'requiredParam' is required.");

if (!IsValidEnumValue(param, out var validValues))
    throw new McpException($"Invalid value '{param}'. Expected one of: {string.Join(", ", validValues)}.");

try
{
    return await _service.SafeOperationAsync(requiredParam, param);
}
catch (Exception ex) when (ex is HttpRequestException or ServiceException)
{
    throw new McpException($"Failed to perform operation: {ex.Message}", ex);
}
```

**Strengths:**
- Validates early, fails fast with clear message
- Prevents ArgumentException from service layer
- Message lists valid options for user guidance

### Pattern 4: Config Guard
**Used when:** Tool behavior depends on configuration (e.g., feature flags).

```csharp
if (ConfigurationService.IsFeatureDisabled("file_search"))
    throw new McpException("File content search is disabled by configuration.");

try
{
    return await _service.SearchFileAsync(jobId, workItem, fileName, pattern);
}
catch (Exception ex) when (ex is HttpRequestException or ServiceException)
{
    throw new McpException($"Failed to search: {ex.Message}", ex);
}
```

**Strengths:**
- Configuration failures are surfaced clearly
- No InvalidOperationException leak from service layer

### Pattern 5: No-Op Methods
**Used when:** Tool is synchronous, accesses only in-memory state, no I/O or parsing.

```csharp
public AuthStatusResult AuthStatus()
{
    // No try/catch needed — purely computational
    var token = _tokenAccessor.GetAccessToken();
    var isAuthenticated = !string.IsNullOrEmpty(token);
    
    return new AuthStatusResult
    {
        IsAuthenticated = isAuthenticated,
        Source = _tokenAccessor.GetType().Name
    };
}
```

**Strengths:**
- No exceptions possible (no I/O, no parsing, no logic)
- Code is simpler, intent is clearer
- Correctly identified as not needing try/catch

---

## Error Message Format Guidelines

**Format:** `"Failed to {action}: {ex.Message} [optional: contextual hint]"`

**Principle:** Preserve original exception message, add actionable context where applicable.

### Examples

✅ **Good:**
```
"Failed to get build status: Build not found in dnceng-public/public. 
If this is an internal build, use org='dnceng' and project='internal' 
(requires auth via 'az login' or AZDO_TOKEN)."

"Failed to download files: No files matching '*.binlog' found."

"Failed to search: File content search is disabled by configuration."

"Work item name is required. Provide it as a separate parameter or include it in the Helix URL."
```

❌ **Bad:**
```
"An error occurred." 
  → Too vague, no actionable context

"System.NullReferenceException: Object reference not set to an instance of an object."
  → Exposes internals, not user-friendly

"Failed to get CI guidance."
  → What went wrong? Invalid repo? Network error? Config issue?

"Service unavailable. Please try again later."
  → Could be auth issue, network issue, or service maintenance
```

---

## Checklist for Auditors

### Before Starting
- [ ] Identify all MCP tool classes in target codebase
- [ ] Confirm you have read access to source files
- [ ] Understand team's exception handling standards (e.g., BinlogMcp pattern)
- [ ] Set up environment: code editor, grep/IDE with cross-file search

### During Audit
- [ ] Enumerate all tools (grep for `[McpServerTool`)
- [ ] For each tool, walk the method code and classify posture
- [ ] Record exception types that can escape
- [ ] Note whether error messages are actionable (user-facing context)
- [ ] Assess I/O surface area (network, file, parsing, etc.)
- [ ] Score by user impact × ease of fix
- [ ] Document patterns observed (what's working, what's not)

### Report Generation
- [ ] Create tool-by-tool inventory table
- [ ] Write detailed findings for each issue
- [ ] Extract and catalog reusable patterns
- [ ] Provide 4–5 recommended pattern templates
- [ ] Prioritize top 5 fixes
- [ ] Cross-check against team standards
- [ ] List open questions for architecture team

### After Audit
- [ ] Append findings to project history
- [ ] Create decision document for implementation team
- [ ] Extract reusable skill methodology
- [ ] Archive audit report as reference

---

## FAQ

**Q: Should I fix issues myself?**
A: No. Audits are read-only investigations. Document findings in decision documents; implementation team (Ripley) will fix based on priority.

**Q: What if a tool has multiple exception types?**
A: List all observed types. Prioritize by visibility + surface area. E.g., "HttpRequestException from HelixService (I/O-bound, high visibility)" vs "ArgumentException from validation (logic, preventable)".

**Q: How do I know if an exception is "actionable"?**
A: Would an LLM agent or user know what to do with this error? "Build not found" → actionable (try different org). "Null reference" → not actionable (what does it mean for the agent?).

**Q: Should auth-status tools wrap async calls?**
A: Only if they do I/O or parsing. If they're purely computational (read in-memory token state), no wrapping needed. Mark for review if auth logic changes.

**Q: What if the service layer itself has unhandled exceptions?**
A: That's a "Complex" fix requiring service-layer refactoring. Document it and flag for architecture team. The audit's job is to surface, not to fix.

---

## Related Resources

- BinlogMcp exception pattern (decisions.md, 2026-03-13)
- C# MCP SDK v1.3.0 exception types (ModelContextProtocol namespace)
- Team error-handling standards (decisions.md)

---

**Last Updated:** 2026-05-21 (Extracted from ash-mcp-exception-audit.md, HelixTool audit)
