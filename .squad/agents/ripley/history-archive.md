# Ripley — History Archive

> Entries older than 2 weeks, archived on 2026-02-15.

## 2025-07-23: Threat model action items

📌 Team update (2025-07-23): STRIDE threat model approved — Ripley has two action items: (1) E1: add URL scheme validation in DownloadFromUrlAsync (reject non-http/https), (2) D1: add batch size guard in GetBatchStatusAsync (cap at 50) — decided by Dallas

## 2025-07-23: P1 Security Fixes (E1 + D1)

- **E1 — URL scheme validation:** Added scheme check in `DownloadFromUrlAsync` (HelixService.cs line ~388). After parsing the URL to a `Uri`, validates `uri.Scheme` is `"http"` or `"https"`. Throws `ArgumentException` with message including the rejected scheme. This runs before any HTTP request, blocking `file://`, `ftp://`, etc.
- **D1 — Batch size limit:** Added `internal const int MaxBatchSize = 50` to `HelixService` (line ~488). `GetBatchStatusAsync` now checks `idList.Count > MaxBatchSize` and throws `ArgumentException` with the actual count and the limit. Tests can reference the constant via `HelixService.MaxBatchSize`.
- **MCP tool description updated:** `hlx_batch_status` description in `HelixMcpTools.cs` now documents "Maximum 50 jobs per request."
- All three changes use `ArgumentException`, consistent with the existing `ArgumentException.ThrowIfNullOrWhiteSpace` pattern in the codebase.

📌 Team update (2026-02-13): Security validation test strategy for E1+D1 fixes (18 tests, negative assertion pattern) — decided by Lambert


📌 Team update (2026-02-13): Remote search design — US-31 (hlx_search_file) and US-32 (hlx_test_results) designed. Phase 1: refactor SearchConsoleLogAsync + add SearchFileAsync (~100 lines). Phase 2: TRX parsing with XmlReaderSettings (DtdProcessing.Prohibit, XmlResolver=null). 50MB file size cap. — decided by Dallas

## 2025-07-23: US-31 — hlx_search_file Implementation

- **Extracted `SearchLines` private static helper** from `SearchConsoleLogAsync` in HelixService.cs. Takes `identifier`, `lines[]`, `pattern`, `contextLines`, `maxMatches` and returns `LogSearchResult`. Both `SearchConsoleLogAsync` and `SearchFileAsync` call it.
- **Added `SearchFileAsync`** to HelixService: downloads file via `DownloadFilesAsync(exact fileName)`, checks binary (null byte in first 8KB), enforces `MaxSearchFileSizeBytes` (50MB), delegates to `SearchLines`, cleans up temp files in finally. Returns `FileContentSearchResult(FileName, Matches, TotalLines, Truncated, IsBinary)`.
- **Added config toggle**: `IsFileSearchDisabled` static property checks `HLX_DISABLE_FILE_SEARCH=true` env var. Both `SearchConsoleLogAsync` and `SearchFileAsync` throw `InvalidOperationException` when disabled. MCP tools `SearchLog` and `SearchFile` return JSON error instead.
- **Added `hlx_search_file` MCP tool** in HelixMcpTools.cs following the exact `SearchLog` pattern — `TryResolveJobAndWorkItem` URL resolution, config toggle check, binary detection returns JSON error.
- **Added `search-file` CLI command** in Program.cs following the `search-log` pattern — positional args (jobId, workItem, fileName, pattern), context highlighting with ConsoleColor.Yellow.
- **Constants/records added**: `MaxSearchFileSizeBytes`, `IsFileSearchDisabled`, `FileContentSearchResult` record.
- Pattern: `DownloadFilesAsync` with exact fileName filter works as a single-file download since `MatchesPattern` does substring match — passing exact name matches only that file.


📌 Team update (2026-02-13): HLX_DISABLE_FILE_SEARCH config toggle added as security safeguard for disabling file content search operations — decided by Larry Ewing (via Copilot)

## 2025-07-23: US-32 — hlx_test_results TRX Parsing Implementation

- **Added `ParseTrxFile` private static method** to HelixService.cs. Parses TRX XML using secure `XmlReaderSettings` (`DtdProcessing.Prohibit`, `XmlResolver=null`, `MaxCharactersFromEntities=0`, `MaxCharactersInDocument=50M`). Extracts `UnitTestResult` elements from TRX namespace `http://microsoft.com/schemas/VisualStudio/TeamTest/2010`. Truncates error messages at 500 chars, stack traces at 1000 chars.
- **Added `ParseTrxResultsAsync` public method** to HelixService.cs. Downloads TRX files via `DownloadFilesAsync`, checks `IsFileSearchDisabled` and `MaxSearchFileSizeBytes`, parses each file, cleans up temp files in finally block. Auto-discovers all `.trx` files when no specific fileName is provided.
- **Added records**: `TrxTestResult(TestName, Outcome, Duration, ComputerName, ErrorMessage, StackTrace)` and `TrxParseResult(FileName, TotalTests, Passed, Failed, Skipped, Results)`.
- **Added `s_trxReaderSettings`** as `private static readonly XmlReaderSettings` field — follows the `s_jsonOptions` naming pattern.
- **Added `hlx_test_results` MCP tool** in HelixMcpTools.cs — follows SearchFile pattern with URL resolution, config toggle check, structured JSON output with per-file summary + results.
- **Added `test-results` CLI command** in Program.cs — positional args (jobId, workItem), color-coded output (red for FAIL, green for PASS, yellow for skipped).
- **Key patterns**: `using System.Xml` and `using System.Xml.Linq` added to HelixService.cs. XmlReader wraps FileStream for secure parsing. Filter logic: failed tests always included, non-pass/non-fail always included, passed only when `includePassed=true`.

## 2025-07-23: Status filter refactor (boolean → enum string)

- **MCP tool (`hlx_status`):** Replaced `bool includePassed = false` with `string filter = "failed"`. Three values: `"failed"` (default, shows only failures), `"passed"` (shows only passed, failed=null), `"all"` (both populated). Validation throws `ArgumentException` for invalid values. Uses `StringComparison.OrdinalIgnoreCase`.
- **CLI command (`status`):** Replaced `bool all = false` with `[Argument] string filter = "failed"` as second positional arg. Same three-way filter logic. Hint text updated from `(use --all to show)` to `(use 'hlx status <jobId> all' to show)`. Help text updated to `hlx status <jobId> [failed|passed|all]`.
- **Breaking change:** `--all` and `includePassed` no longer exist. Callers must use `filter="all"`.

📌 Team update (2025-07-23): Status filter refactored from boolean (--all/includePassed) to enum-style string (failed|passed|all) in both MCP tool and CLI command — decided by Ripley

## 2025-07-23: CI version validation

- Added "Validate version consistency" step to `.github/workflows/publish.yml` that checks csproj `<Version>`, server.json top-level `version`, and server.json `packages[0].version` all match the git tag. Fails with clear expected-vs-actual messages on mismatch.
- Updated Pack step to pass `/p:Version=${{ steps.tag_name.outputs.current_version }}` so the NuGet package version always matches the tag regardless of csproj content.
- Belt-and-suspenders: validation catches developer mistakes early, `/p:Version=` override ensures correctness even if validation is bypassed.


📌 Team update (2026-02-15): Cache security expectations documented in README (cached data subsection, auth isolation model, hlx cache clear recommendation) — decided by Kane
📌 Team update (2026-02-15): README v0.1.3 comprehensive update — llmstxt in Program.cs needs sync (missing hlx_search_file, hlx_test_results, search-file, test-results) — decided by Kane
📌 Team update (2026-02-15): DownloadFilesAsync temp dirs now per-invocation (helix-{id}-{Guid}) to prevent cross-process races — decided by Ripley
📌 Team update (2026-02-15): CI version validation added to publish workflow — tag is source of truth, csproj+server.json must match — decided by Ripley
## Summarized History (through 2026-02-11) — archived 2026-03-08

**Architecture & DI (P0):** Implemented IHelixApiClient interface with projection interfaces for Helix SDK types, HelixApiClient wrapper, HelixException, constructor injection on HelixService, CancellationToken on all methods, input validation (D1-D10). DI for CLI via `ConsoleApp.ServiceProvider`, for MCP via `builder.Services.AddSingleton<>()`.

**Key patterns established:**
- Helix SDK types are concrete — mockable via projection interfaces (IJobDetails, IWorkItemSummary, IWorkItemDetails, IWorkItemFile)
- `TaskCanceledException`: use `cancellationToken.IsCancellationRequested` to distinguish timeout vs cancellation
- Program.cs has UTF-8 BOM — use `UTF8Encoding($true)` when writing
- `FormatDuration` duplicated in CLI/MCP — extract to Core if third consumer appears
- HelixMcpTools.cs duplicated in HelixTool and HelixTool.Mcp — both must be updated together
- Two DI containers in CLI: one for commands, separate `Host.CreateApplicationBuilder()` for `hlx mcp`

**Features implemented:**
- US-1 (positional args), US-5 (dotnet tool packaging v0.1.0), US-11 (--json flag on status/files)
- US-17 (namespace cleanup: HelixTool.Core, HelixTool.Mcp), US-18 (removed unused Spectre.Console)
- US-20 (rich status: State, ExitCode, Duration, MachineName), US-24 (download by URL)
- US-25 (ConsoleLogUrl on WorkItemResult), US-29 (MCP URL parsing for optional workItem)
- US-30 (structured JSON: grouped files, jobId+helixUrl in status)
- US-10 (WorkItemDetail + work-item command + hlx_work_item MCP tool)
- US-23 (BatchJobSummary + batch-status command + hlx_batch_status MCP tool, SemaphoreSlim(5) throttling)
- Stdio MCP transport via `hlx mcp` subcommand

**Team updates received:**
- Architecture review, caching strategy, cache TTL policy, requirements backlog (30 US), docs fixes (Kane), auth design (US-4), MCP test strategy — all in decisions.md

**2025-07-23 session (archived):**
- STRIDE threat model action items: E1 URL scheme validation in DownloadFromUrlAsync, D1 batch size guard in GetBatchStatusAsync
- P1 security fixes implemented: URL scheme check (http/https only, throws ArgumentException), MaxBatchSize=50 const with guard in GetBatchStatusAsync, MCP tool description updated
- US-31 hlx_search_file: extracted SearchLines helper, added SearchFileAsync (binary detection, 50MB cap, config toggle), MCP tool + CLI command
- US-32 hlx_test_results: TRX parsing with secure XmlReaderSettings (DtdProcessing.Prohibit), ParseTrxResultsAsync, auto-discovery of .trx files, MCP tool + CLI command
- Status filter refactor: bool includePassed → string filter (failed|passed|all), case-insensitive, breaking change
- CI version validation: publish workflow validates csproj+server.json match git tag, /p:Version= override

## Archived 2026-03-09: Learnings (HelixTool.Mcp.Tools extraction)

- **HelixTool.Mcp.Tools project:** New class library at `src/HelixTool.Mcp.Tools/` containing MCP tool definitions (HelixMcpTools, AzdoMcpTools) and MCP result DTOs (McpToolResults.cs). No NuGet packaging metadata.
- **Namespace: `HelixTool.Mcp.Tools`:** All three moved files use this namespace. AzdoMcpTools was previously `HelixTool.Core.AzDO`.
- **ModelContextProtocol package removed from Core:** Core no longer references `ModelContextProtocol` — that dependency now lives in `HelixTool.Mcp.Tools`.
- **`IsFileSearchDisabled` promoted to public:** Was `internal static` on `HelixService`, had to become `public` for separate assembly.
- **`WithToolsFromAssembly` assembly reference:** Both CLI and HTTP server use `typeof(HelixMcpTools).Assembly`.
- **Test project references all three:** `HelixTool.Tests.csproj` now references Core, Mcp, and Mcp.Tools projects. Six test files needed `using HelixTool.Mcp.Tools;`.
- **git mv preserves history:** Used `git mv` for all three file moves.

## Archived 2026-03-09: Learnings (azdo_search_log implementation)

- **TextSearchHelper extraction:** `SearchLines()` moved from `HelixService` (private static) to `TextSearchHelper` (public static) in `HelixTool.Core`. Records (LogMatch, LogSearchResult, FileContentSearchResult) promoted to top-level in Core namespace.
- **Default parameter values matter:** Added `contextLines = 0, maxMatches = 50` defaults to `TextSearchHelper.SearchLines()` — existing tests relied on calling with fewer args.
- **AzDO log fetching already supports full content:** `AzdoApiClient.GetBuildLogAsync` returns the complete log; for search, pass `tailLines: null` to get full content.
- **IsFileSearchDisabled dual-check pattern:** MCP tool layer (throws McpException) and service layer (throws InvalidOperationException) both check.
- **CLI search output pattern:** Context lines displayed with `>>>` prefix for matching line and `   ` prefix for context. Line numbers right-aligned in 6-char column.

📌 Team update (2026-03-08): AzDO search gap analysis consolidated — CI-analysis skill study validated `azdo_search_log` as P0, confirmed `SearchLines()` extraction approach. New P1 ideas: `azdo_search_timeline`, `azdo_search_log_across_steps`. — analyzed by Ash

## Archived 2026-03-09: Learnings (PR #10 review fixes)

- **CLI line-number calculation:** Derive `startLine` from `m.LineNumber - contextLines` (clamped to 0) instead of TakeWhile. TakeWhile breaks with duplicate line text.
- **CRLF normalization in AzDO logs:** Normalize `\r\n` → `\n`, `\r` → `\n` before `Split('\n')`, trim trailing empty entry.
- **MCP result field naming honesty:** Name property to reflect broader type when field accepts ID or URL.
- **McpException wrapping pattern:** Wrap service calls for expected exceptions (InvalidOperationException, HttpRequestException) and rethrow as McpException.

## Archived 2026-03-09: Learnings (azdo_search_timeline implementation)

- **Domain types in Core, not MCP.Tools:** `TimelineSearchMatch` and `TimelineSearchResult` live in `HelixTool.Core.AzDO` (AzdoModels.cs). MCP tools return Core types directly.
- **`[JsonIgnore]` for raw record access:** `TimelineSearchMatch.Record` exposes underlying `AzdoTimelineRecord` with `[JsonIgnore]`.
- **Duration formatting in service layer:** Service computes formatted duration strings. `FormatDuration` is private to `AzdoService`.
- **Null timeline → InvalidOperationException:** MCP layer catches and wraps as McpException.
- **Pre-existing tests drove API shape:** Aligning service return type to match test expectations.

## Archived 2026-03-09: Learnings (PR #11 review fixes)

- **CLI-side validation before service calls:** Validate at CLI layer so error messages reference CLI option names. Check valid values with `string.Equals(OrdinalIgnoreCase)`, throw `ArgumentException` with `nameof(cliParam)`.
- **Doc accuracy for 'failed' filter semantics:** `result="failed"` means "non-succeeded OR has timeline issues", not just result=failed.

> Entries archived on 2026-03-09.

## Learnings (PR #13 review fixes + performance, 2025-07-18)

- **Integer overflow guard:** Cast to `(long)` for user-controlled arithmetic before narrowing to `int`. Falls back to full fetch on overflow.
- **Cache-layer range must match server semantics:** Return `null` for out-of-range, don't clamp.
- **Allocation-free string line ops:** `content.AsSpan().Count('\n')` for CountLines; `IndexOf('\n')` loop + slice for ExtractRange. Critical for large log delta-refresh.
- **SearchValues<char> for line-break scanning:** `SearchValues.Create("\r\n")` + `IndexOfAny` handles all line endings in one pass without pre-normalizing.
- **Shared StringHelpers.TailLines:** Reverse-scan `LastIndexOf('\n')` in Core. Guard empty string (pos+1 exceeds length).
- **SearchConsoleLogAsync refactor safe:** Only download feature and SearchFileAsync use disk path. Search can stream directly.
- **Cache format migration via sentinel prefix:** `raw:` prefix with `JsonSerializer.Deserialize<string>()` fallback for legacy entries. Zero-downtime, natural TTL expiry handles transition.
- **Test assertions must match serialization format:** Tests asserting exact stored values need updating on format changes; `Arg.Any<string>()` is resilient.
- **Key perf anti-patterns:** Chained `.Replace()` (span enumerator instead), Split+Join for tail (reverse-scan slice), `pattern[1..]` substring in loops (span EndsWith), triple-iteration file categorization (single-pass), disk round-trip for search (stream directly), JSON-serialized log strings (plain text + marker).

📌 Team update (2025-07-18): Perf review (17 issues) + 8 fixes implemented (3 P0, 5 P1), all 864 tests passing — decided/implemented by Ripley

📌 Team update (2026-03-09): 3 perf decisions merged — raw: cache prefix, SearchConsoleLog decoupling, shared StringHelpers — decided by Ripley

## Learnings (PR #14 review fixes, 2025-07-18)

- **CRLF in streamed content:** When bypassing `File.ReadAllLinesAsync` and splitting raw content, always normalize `\r\n`/`\r` to `\n` before splitting — otherwise `\r` leaks into line strings and breaks pattern matching.
- **Trailing empty element on split:** `"content\n".Split('\n')` yields a trailing empty string. Drop it only when `lines.Count > 1` to preserve semantics for single-newline inputs like `"\n"` → `[""]`.
- **Cache sentinel collision:** Plain-text prefixes like `"raw:"` can collide with legitimate log content. Use NUL byte prefixes (`"\0raw\n"`) — NUL can't appear in valid log text, making collision impossible.

📌 Team update (2025-07-18): Fixed 3 PR #14 review comments — CRLF handling in SearchConsoleLogAsync, NormalizeAndSplit edge case for single-newline, cache sentinel collision via NUL prefix — implemented by Ripley
- **Integer overflow in tail optimization:** `tailLines.Value * 2` computed as `int` can overflow for large user-controlled values. Fix: cast to `(long)tailLines.Value * 2` for the comparison, and guard `startLine` arithmetic with bounds check (`> 0 && <= int.MaxValue`) before the `(int)` cast. Falls back to full fetch if values don't fit. Lesson: always consider overflow on user-controlled arithmetic, especially before narrowing casts.
- **ExtractRange clamping vs server semantics:** The original `ExtractRange` clamped out-of-bounds `startLine` to the last line, which differs from AzDO API behavior (returns empty/404 for out-of-range). Fix: return `null` when `start >= lines.Length`, `start < 0`, or `end < start`. Lesson: cache-layer range extraction must match server semantics to avoid behavioral divergence between cached and uncached paths.
- **Allocation-free string line operations:** Replace `string.Split('\n')` with span-based approaches for hot-path methods. `CountLines`: use `content.AsSpan().Count('\n')` (MemoryExtensions.Count) — zero allocation. `ExtractRange`: scan for Nth newline via `IndexOf('\n')` in a loop to find character offsets, then slice with `content[start..end]` — avoids allocating an array of all lines just to extract a small range. Critical for large AzDO logs on delta-refresh paths. Note: must handle content without trailing `\n` (add +1 to count).

## Learnings (performance code review, 2025-07-18)

- **Chained Replace is the #1 perf pattern to watch:** `NormalizeAndSplit()` in AzdoService does `.Replace("\r\n", "\n").Replace("\r", "\n")` creating two intermediate full-size strings before `.Split('\n')` creates a third. In cross-step search this runs up to 30× per request on multi-MB logs. Fix: span-based line enumerator that handles all line ending types in one pass.
- **Split+Join for tail trimming is wasteful:** Both `AzdoService.GetBuildLogAsync` and `HelixService.GetConsoleLogContentAsync` split the entire log into a string[] just to get the last N lines, then Join them back. Fix: reverse-scan for Nth `\n` from end using `ReadOnlySpan<char>.LastIndexOf('\n')`, then slice — zero array allocation.
- **MatchesPattern allocates a substring on every call:** `pattern[1..]` in `MatchesPattern` and `MatchesTestResultPattern` creates a new string. Called per-file in loops (FindFiles scans 30 work items × N files each). Fix: `name.AsSpan().EndsWith(pattern.AsSpan(1), ...)`.
- **Triple-iteration anti-pattern in HelixMcpTools.Files:** Three `.Where().Select().ToList()` chains iterate the file list 3 times with redundant `MatchesPattern`/`IsTestResultFile` calls. Fix: single-pass categorization loop.
- **SearchConsoleLogAsync does disk round-trip unnecessarily:** Downloads log to temp file, then reads it all back with `File.ReadAllLinesAsync`. Could stream directly into memory, avoiding double I/O.
- **CachingAzdoApiClient serializes log content as JSON strings:** `JsonSerializer.Serialize<string>()` on multi-MB log content escapes every special character and wraps in quotes. `Deserialize<string>()` on cache hit re-parses it all. For large logs, store as plain text with a content-type marker instead.
- **Static array allocations on every call:** `knownTrailingSegments` in `HelixIdResolver.TryResolveJobAndWorkItem` allocates `string[]` per invocation. Should be `static readonly`.

📌 Team update (2025-07-18): Perf review identified 17 allocation issues — decided by Ripley

## Learnings (version bump 0.3.0, 2025-07-18)

- Version bump to 0.3.0 for the release containing AzDO integration, perf optimizations, and incremental log support
📌 Team update (2026-03-09): Test quality guidelines established — no layer duplication in tests, passthrough methods get ≤1 smoke test, interface compliance tests are redundant. ~20 tests cleaned up (PR #15). — decided by Dallas, actioned by Lambert

## 2026-03-13: Archived from history.md during summarization

## Learnings (MCP tool description updates with CI knowledge)

- **5 tool descriptions updated** with repo-specific CI knowledge (helix_test_results, helix_search_log, azdo_test_runs, azdo_test_results, azdo_timeline).
- **warn-before-fail pattern:** helix_test_results warns that 4/6 repos don't upload TRX, directing to azdo_test_runs/results.
- **Repo-specific search patterns:** runtime=`[FAIL]`, aspnetcore/efcore=`  Failed`, sdk=`error MSB`, roslyn=`aborted`/`Process exited`.
- **azdo_test_runs:** failedTests=0 can lie — always drill into results. azdo_timeline includes Helix task name mapping per repo.

## Learnings (CiKnowledgeService enrichment — 9-repo knowledge base)

- **CiRepoProfile expanded** with 9 new properties (PipelineNames, OrgProject, ExitCodeMeanings, etc.). Init-only defaults for backward compat. 3 new repos: maui (3 pipelines), macios (devdiv, NUnit), android (devdiv, NUnit+xUnit). Total: 9.
- **devdiv org repos need ⚠️ warnings** — standard helix_*/ado-dnceng-* tools don't work for macios/android.
- **MAUI is unique:** 3 separate pipelines with different investigation approaches. Pipeline identity matters for tool selection.
- **FormatProfile/GetOverview enriched** with org, pipelines, gotchas, exit codes, investigation order columns.

📌 Team update (2026-03-10): CiKnowledgeService enrichment (9 repos, 9 new properties, 171 tests, PR #16). — Ripley

## Learnings (PR #16 review comment fixes)

- **Try-block indentation:** Re-indent body when wrapping in try. Caught in 4 HelixMcpTools methods.
- **McpException must include inner exception and "Failed to" prefix.** Three AzDO search catch blocks were dropping context.
- **Error messages: don't hardcode one repo's pattern.** Use multiple patterns + point to `helix_ci_guide`.
- **Bool→string for nuanced semantics.** `UploadsTestResultsToHelix: bool` → `HelixTestResultAvailability: string` (`"none"`, `"partial"`, `"varies"`).
- **When renaming record properties, update test assertions too.**

## Learnings (Option A folder restructuring)

- **Moved 9 Helix files** from `Core/` root to `Core/Helix/`: HelixService, HelixApiClient, IHelixApiClient, IHelixApiClientFactory, HelixIdResolver, HelixException, IHelixTokenAccessor, ChainedHelixTokenAccessor, plus CachingHelixApiClient from `Cache/`. Namespace: `HelixTool.Core` → `HelixTool.Core.Helix`.
- **Added `HelixTool.Core.Cache` namespace** to 6 cache infrastructure files in `Cache/`: SqliteCacheStore, ICacheStore, ICacheStoreFactory, CacheOptions, CacheSecurity, CacheStatus.
- **Extracted `MatchesPattern` and `IsFileSearchDisabled`** from HelixService to `StringHelpers.cs`. HelixService methods now delegate to StringHelpers. AzdoService, HelixMcpTools, and AzdoMcpTools updated to call StringHelpers directly — breaking the AzDO→Helix coupling.
- **Moved MCP tools**: HelixMcpTools → `Mcp.Tools/Helix/`, AzdoMcpTools → `Mcp.Tools/AzDO/`.
- **Moved 24 Helix-specific test files** to `Tests/Helix/`. Kept shared tests (cache, security, CI knowledge, text search, API middleware) at root.
- **HelixService.cs needed `using HelixTool.Core.Cache;`** — it references `CacheSecurity` for path validation. Initial build failed until this was added. Key learning: when splitting namespaces within the same project, intra-project cross-namespace references are easy to miss.
- **StringHelpers changed from `internal` to `public`** to support cross-project access (MCP tools project references it).
- **59 files touched**, 0 behavioral changes, all 1038 tests pass.

📌 Team update (2026-03-10): Option A folder restructuring executed — 9 Helix files moved to Core/Helix/, Cache namespace added, shared utils extracted from HelixService, Helix/AzDO subfolders in Mcp.Tools and Tests. 59 files, 1038 tests pass, zero behavioral changes. PR #17. — decided by Dallas (analysis), Ripley (execution)

## Learnings (security boundary and DI review fixes)

- **Path boundary checks:** For security-sensitive root containment, normalize both paths, preserve the root boundary with `Path.TrimEndingDirectorySeparator(...) + Path.DirectorySeparatorChar`, and compare with `StringComparison.Ordinal`; ignore-case prefix checks can admit case-variant sibling paths on case-sensitive filesystems.
- **HelixService constructor contract:** `HelixService` should require an injected `HttpClient` and null-guard both constructor dependencies instead of silently allocating a fallback transport.
- **User preference:** Code-review follow-up fixes should stay surgical, behavior-safe, and avoid unrelated refactoring.
- **Key file paths:** `src/HelixTool.Core/Cache/CacheSecurity.cs` contains cache/download path traversal guards. `src/HelixTool.Core/Helix/HelixService.cs` owns direct URL download behavior and now depends on caller-provided `HttpClient`. `src/HelixTool/Program.cs` and `src/HelixTool.Mcp/Program.cs` are the production DI registration points for `HelixService`.

📌 Team update (2026-03-10): Review-fix decisions merged — README now leads with value prop, shared caching, and context reduction; cache path containment uses exact Ordinal root-boundary checks; and HelixService requires an injected HttpClient with no implicit fallback. Validation confirmed current CLI/MCP DI sites already comply and focused plus full-suite coverage exists. — decided by Kane, Lambert, Ripley

📌 Team update (2026-03-10): Knowledgebase refresh guidance merged — treat the knowledgebase as a living document aligned to current file state, not a static snapshot; earlier README/cache-security/HelixService review findings are resolved knowledge, and only residual follow-up should stay active (discoverability plus documentation/tool-description synchronization). — requested by Larry Ewing, refreshed by Ash

## Learnings (discoverability routing pass)

- **Behavioral routing beats vague warnings:** For tool-selection surfaces, state when a tool works, when to skip it, and the exact fallback path instead of saying it may fail.
- **Guide ordering pattern:** A short `Start Here` section before gotchas and inventory makes repo-specific workflow choice discoverable without reading the entire CI profile.
- **User preference:** Keep discoverability improvements incremental; do not add composite tools or new parameters when wording/order changes can solve the workflow gap.
- **Key file paths:** `src/HelixTool.Mcp.Tools/Helix/HelixMcpTools.cs` holds MCP tool descriptions, `src/HelixTool.Core/Helix/HelixService.cs` owns `helix_test_results` fallback messaging, `src/HelixTool.Core/CiKnowledgeService.cs` formats repo-specific CI guides, `src/HelixTool.Mcp.Tools/CiKnowledgeTool.cs` describes `helix_ci_guide`, and `src/HelixTool/Program.cs` mirrors MCP guidance in llms-txt/help output.

📌 Team update (2026-03-10): Discoverability routing decisions merged — keep the current tool surface, route repo-specific workflow selection through `helix_ci_guide(repo)`, treat `helix_test_results` as structured Helix-hosted parsing rather than a universal first step, and keep `helix_search_log`/docs/help guidance synchronized across surfaces. — decided by Dallas, Kane, Ripley

## 2026-03-13: Archived older learnings from history.md

📌 Team update (2025-07-24): Test quality review — ~17 redundant tests deleted, no layer duplication rule. — Dallas

## Archived from history.md (2026-03-13 auth-remaining summarization)

### 2026-03-08 through 2026-03-13 (pre-PR-28)
- AzDO search/log work established the 4-bucket ranking, concurrent metadata fetch, and the `McpException` wrapping pattern for MCP-visible failures.
- CI-routing guidance converged on short behavioral tool descriptions, repo-specific detail in `helix_ci_guide`, and scope-accurate MCP names (`helix_search`, `helix_parse_uploaded_trx`).
- Option A restructuring moved Helix code under `Core/Helix`, split cache infrastructure into `HelixTool.Core.Cache`, and extracted shared helpers into `StringHelpers`.
- Security hardening locked exact Ordinal path-boundary checks, required injected `HttpClient` for `HelixService`, and added explicit truncation metadata for capped MCP responses.
- Early AzDO auth hardening established the narrow chain, scheme-aware `AzdoCredential` metadata, explicit `AZDO_TOKEN_TYPE` override, and redaction of unexpected error snippets.

# Ripley — History

## Project Learnings (from import)
- **Project:** hlx — Helix Test Infrastructure CLI & MCP Server
- **User:** Larry Ewing
- **Stack:** C# .NET 10, ConsoleAppFramework, ModelContextProtocol, Microsoft.DotNet.Helix.Client
- **Structure:** Four projects — HelixTool.Core (shared library), HelixTool (CLI), HelixTool.Mcp (HTTP MCP server), HelixTool.Mcp.Tools (MCP tool definitions)
- **Key service methods:** GetJobStatusAsync, GetWorkItemFilesAsync, DownloadConsoleLogAsync, GetConsoleLogContentAsync, FindBinlogsAsync, DownloadFilesAsync, GetWorkItemDetailAsync, GetBatchStatusAsync, DownloadFromUrlAsync
- **HelixIdResolver:** Handles bare GUIDs, full Helix URLs, and `TryResolveJobAndWorkItem` for URL-based jobId+workItem extraction
- **MatchesPattern:** Simple glob — `*` matches all, `*.ext` matches suffix, else substring match

## Core Context

- **Implementation layout:** service code lives under `src/HelixTool.Core/Helix/` and `src/HelixTool.Core/AzDO/`; MCP tool definitions live under `src/HelixTool.Mcp.Tools/Helix/` and `src/HelixTool.Mcp.Tools/AzDO/`.
- **Cache/search primitives:** `CachingHelixApiClient`, `CachingAzdoApiClient`, `StringHelpers`, and `TextSearchHelper` are the shared implementation seams; `HLX_CACHE_MAX_SIZE_MB=0` disables caching and `HLX_DISABLE_FILE_SEARCH` disables file-content search.
- **Wire-format conventions:** structured MCP tools use `UseStructuredContent=true`, camelCase JSON/property names remain stable, and descriptions stay behavior-first with repo-specific guidance routed to `helix_ci_guide`.
- **Hot-path log rules:** keep large-log search/tail work span-based, overflow-safe, and semantically aligned with server behavior; when tagging raw cached payloads, prefer collision-proof sentinels over human-readable prefixes.
- **Test/routing defaults:** avoid layer-duplicate or passthrough-only tests, and keep repo-specific workflow guidance in `helix_ci_guide` instead of bloating always-loaded tool descriptions.
- **Auth/runtime:** Helix auth remains env-var based, while AzDO auth now uses the narrow chain `AZDO_TOKEN` → `AzureCliCredential` → az CLI → anonymous with metadata carried by `AzdoCredential`.

## Summarized History (2026-03-08 through 2026-03-13 pre-PR-28) — archived 2026-03-13

Detailed notes for AzDO search/log ranking, MCP error surfacing, CI-knowledge description churn, Option A restructuring, review-fix security hardening, discoverability routing, Helix MCP renames, truncation metadata, and early AzDO auth-chain hardening moved to `history-archive.md`.

- **Durable routing/doc rules:** keep repo-specific CI guidance in `helix_ci_guide`, keep tool descriptions short and behavior-first, and keep scope-accurate MCP names like `helix_search` / `helix_parse_uploaded_trx`.
- **Architecture/security rules:** shared code lives under `HelixTool.Core/{Helix,AzDO,Cache}` and `HelixTool.Mcp.Tools/{Helix,AzDO}`; shared utilities belong in `StringHelpers`; security-sensitive path containment uses normalized full-path + Ordinal root-boundary checks; `HelixService` requires injected `HttpClient`.
- **UX/runtime rules:** capped MCP list/search responses should surface explicit truncation metadata (`LimitedResults<T>`, `LogSearchResult.Truncated`), and AzDO auth stays on the narrow chain `AZDO_TOKEN` → `AzureCliCredential` → az CLI → anonymous with scheme-aware `AzdoCredential` metadata and sanitized error snippets.

## Learnings (remaining AzDO auth threat-model follow-ups)

- **CLI schema generation now respects explicit JSON property names:** `src/HelixTool.Core/CliSchema/SchemaGenerator.cs` uses `JsonPropertyNameAttribute` values when present, so `--schema` matches the actual `System.Text.Json` wire names instead of always reflecting raw CLR property names.
- **Historical CLI JSON casing is per-field, not per-command:** for `status`, `files`, and `work-item`, the old anonymous-object field names in `src/HelixTool/Program.cs` are the wire contract, so typed DTOs should add `[JsonPropertyName]` only on members that were explicitly lowercase/camelCase (`job`, `jobId`, `totalWorkItems`, `failedCount`, `passedCount`, `duration`, `failureCategory`, `binlogs`, `testResults`, `other`, `files`) and leave CLR-default PascalCase members alone.
- **CLI-only JSON DTOs should keep serializer defaults unless the external wire format truly requires overrides:** the private `status`, `files`, and `work-item` DTOs in `src/HelixTool/Program.cs` are meant to preserve the CLI's exact historical output, which may mix CLR-default PascalCase with explicit camelCase field names, so only members that were explicitly renamed in the old anonymous-object payloads should carry `[JsonPropertyName]`.

- **Refreshable fallback credentials now carry expiry metadata:** `AzCliAzdoTokenAccessor` caches Azure CLI and `az`-subprocess credentials with a refresh deadline (`min(expiresOn - 5m, now + 45m)`) instead of pinning the first token for the full process lifetime; 401/403 responses now invalidate that cached fallback so the next request re-resolves auth.
- **Stable cache isolation uses source identity, not token bytes:** `AzdoCredential.CacheIdentity` is derived from the auth path plus stable JWT claims (`tid`/`oid`/`appid`/`sub`) when available, and `AzdoApiClient` stores `CacheOptions.AuthTokenHash` from that stable identity after the first successful authenticated AzDO response. `CachingAzdoApiClient` then prefixes all AzDO cache/state keys with that hash when present.
- **Auth visibility is now first-class metadata:** `IAzdoTokenAccessor.AuthStatusAsync` reports the resolved path, credential source, expiry signal, and warnings without making an AzDO request; `hlx azdo auth-status` prints or serializes that safe metadata for operators.
- **Key file paths:** `src/HelixTool.Core/AzDO/IAzdoTokenAccessor.cs` now owns fallback-token refresh, cache identity derivation, and auth-status reporting. `src/HelixTool.Core/AzDO/AzdoApiClient.cs` now invalidates stale fallback auth on 401/403 and seeds `CacheOptions.AuthTokenHash` after successful auth. `src/HelixTool.Core/AzDO/CachingAzdoApiClient.cs` applies the auth hash to AzDO cache keys, while `src/HelixTool/Program.cs` exposes `azdo auth-status` and the CLI/MCP hosts both pass shared `CacheOptions` into `AzdoApiClient`.
- **PAT cache identities now get a secret-safe fingerprint fallback:** when `AzdoCredential.BuildCacheIdentity` cannot extract JWT claims, it appends the first 8 hex chars of a SHA256 over the token so PAT-backed AzDO contexts stay isolated without persisting the raw secret.
- **AzDO key partitioning is now established before cache reads and kept separate from cache-root partitioning:** `AzdoApiClient` seeds the mutable AzDO auth hash immediately after credential resolution, `CachingAzdoApiClient` also pre-resolves that hash before its first cache lookup, and `CacheOptions` now keeps stable cache-root partitioning in `CacheRootHash` instead of reusing `AuthTokenHash`.
- **Cache-key hygiene and CLI auth-status exits were tightened together:** `CachingAzdoApiClient` sanitizes the auth-hash segment before composing cache/state keys, and `hlx azdo auth-status` now sets a non-zero exit code for anonymous status even on the `--json` path.
- **JWT cache identities now fall back to fingerprints when stable claims are missing:** `AzdoCredential.BuildCacheIdentity` only returns claim-based identities when at least one of `tid`, `oid`, `appid`, or `sub` is present; otherwise JWTs use the same SHA256 suffix path as non-JWT tokens so cache partitioning stays per-principal instead of collapsing to the bare prefix.
- **AzDO token env-var tests are now serialized:** the `AzdoTokenEnv` xUnit collection now has `DisableParallelization = true`, which keeps PATH-mutation tests from racing unrelated tests that spawn external processes.

## Learnings (MCP Timeline Analysis — 2026-07-18)

- **`azdo_timeline` filter='failed' does MCP-layer record filtering but still returns raw `AzdoTimelineRecord` objects with all fields:** The MCP layer (lines 68–122 of `AzdoMcpTools.cs`) fetches the full timeline from the API client (`_client.GetTimelineAsync`), then filters client-side to non-succeeded records plus their parent chain. For large VMR builds (~600+ records), the raw AzdoTimeline JSON can be 400–600KB even after filtering to failures, because each record carries issues arrays, previousAttempts, log references, workerName, etc.
- **`azdo_timeline` filter='failed' does server-side record filtering but still returns raw `AzdoTimelineRecord` objects with all fields:** The MCP layer (lines 68–122 of `AzdoMcpTools.cs`) fetches the full timeline from the API client (`_client.GetTimelineAsync`), then filters client-side to non-succeeded records plus their parent chain. For large VMR builds (~600+ records), the raw AzdoTimeline JSON can be 400–600KB even after filtering to failures, because each record carries issues arrays, previousAttempts, log references, workerName, etc.
- **`azdo_search_timeline` solves the large-output problem by returning only pattern-matched records with a compact DTO:** `SearchTimelineAsync` (lines 182–290 of `AzdoService.cs`) fetches the same full timeline but then applies text search on record names and issue messages, returning `TimelineSearchMatch` objects that include only recordId, name, type, state, result, duration, logId, matchedIssues, and parentName — dramatically smaller than raw records.
- **The ci_guide workflow instructions never mention `azdo_search_timeline`:** All 9 repo profiles in `CiKnowledgeService.cs` recommend `azdo_timeline(buildId, filter='failed')` as the first investigation step, but none mention `azdo_search_timeline` as an alternative. The generic overview also omits it. This is the discoverability gap that caused the gist scenario.
- **Intended AzDO investigation workflow:** `azdo_timeline` (identify failed steps) → `azdo_log` (read specific step log by logId) or `azdo_search_log` (search across logs by pattern). `azdo_search_timeline` is a shortcut that combines timeline retrieval + filtering + text search in one call, ideal when you know what error text you're looking for.
- **MCP tool layer does significant post-processing over raw API:** `AzdoService` orchestrates URL resolution via `AzdoIdResolver`, parallel metadata fetches, ranked log search with early termination, and result shaping — it's not a thin pass-through. The API client (`AzdoApiClient`) handles auth and HTTP; the service layer handles business logic.

📌 Team update (2026-03-14): helix-cli skill docs must reflect shipped CLI behavior: use `hlx llms-txt` for CLI discovery, note no `hlx ci-guide` command yet, and keep `hlx search-log` CLI docs text-only. — decided by Kane
- **Added CLI schema discovery for JSON commands:** `src/HelixTool.Core/CliSchema/SchemaGenerator.cs` now builds pretty-printed JSON skeletons from reflected public types, including placeholder scalars, single-item collections, nested object recursion, enum value summaries, and circular/depth protection. `src/HelixTool/Program.cs` now exposes `TryPrintSchema<T>` and wires `--schema` into the 14 CLI query commands that already support `--json`.
- **Preserved existing Helix JSON wire shapes while introducing named schema types:** the `status`, `files`, and `work-item` CLI commands now serialize private DTOs in `src/HelixTool/Program.cs` with `[JsonPropertyName]` attributes so their runtime `--json` payloads stay stable even though schema discovery now reflects named types instead of anonymous objects.
- **`azdo search-log --schema` follows the active JSON mode:** when `--log-id` is supplied, the CLI prints `LogSearchResult`; otherwise it prints `CrossStepSearchResult`, matching the command's two existing JSON response shapes without redesigning the command surface.
- **CLI describe metadata is now generator-driven:** `src/HelixTool.Generators/DescribeGenerator.cs` reads MCP `[Description]` metadata from the referenced `HelixTool.Mcp.Tools` assembly, joins it with CLI `[McpEquivalent]` + `[Command]` metadata from `src/HelixTool/Program.cs`, and emits `HelixTool.Generated.CommandRegistry` so MCP descriptions stay the single source of truth.
- **CLI/MCP parity markers live in Core:** `src/HelixTool.Core/McpEquivalentAttribute.cs` is the shared marker attribute that CLI command methods use to opt into the generated registry without introducing a shared constants file for descriptions.
- **Key file paths:** `src/HelixTool.Generators/HelixTool.Generators.csproj` is the netstandard2.0 analyzer project referenced from `src/HelixTool/HelixTool.csproj`, and `hlx describe` summary/detail rendering now lives in `src/HelixTool/Program.cs`.

## Learnings (Timeline Truncation & Discoverability — 2026-07-18)

- **Partial response pattern for large MCP outputs:** When `azdo_timeline` returns >200 records, the MCP layer now truncates to the first 100 records and includes a `note` field with a ⚠️ message pointing the agent to `azdo_search_timeline`. The `TimelineResponse` record (in `AzdoMcpTools.cs`) wraps the timeline data with `truncated`, `totalRecords`, and `note` fields — using `JsonIgnore(Condition = WhenWritingDefault/WhenWritingNull)` so non-truncated responses stay clean.
- **Larry's preference: partial response > silent drop.** When output is too large, return what fits with a continuation indicator rather than dropping data silently. This applies to any MCP tool that might produce oversized responses.
- **ci_guide now recommends azdo_search_timeline as the preferred first investigation step** for repos that previously recommended `azdo_timeline(buildId, filter='failed')` first. Updated: runtime, sdk, roslyn, efcore, VMR profiles, plus the generic overview and non-Helix summary sections. Profiles that don't start with azdo_timeline (aspnetcore, maui, macios, android) were left as-is since their workflows start differently.
- **Threshold constants live in the MCP tool class:** `MaxTimelineRecords = 200` and `TruncatedTimelineBudget = 100` are private consts in `AzdoMcpTools` — the truncation is a presentation concern, not core business logic.

## Learnings (Auth Error UX — 2025-07-25)

- **MCP tool classes now take token accessors in constructors:** `AzdoMcpTools(AzdoService, IAzdoTokenAccessor)` and `HelixMcpTools(HelixService, IHelixTokenAccessor)` — both are injected by DI; tests pass `Substitute.For<...>()`.
- **`azdo_builds` and `azdo_test_attachments` are now URL-aware:** `TryExtractOrgProjectFromUrl` in `AzdoMcpTools` detects when the `org` param contains `dev.azure.com`, `visualstudio.com`, or `://` and auto-extracts org/project via `AzdoIdResolver.TryResolve`. Other tools already get this for free via their `buildId` param going through `AzdoIdResolver.Resolve`.
- **404 hint pattern for internal builds:** `AppendNotFoundHint` in `AzdoMcpTools` checks if the "not found" error mentions the default org/project (`dnceng-public`/`public`), and if so appends a hint that the build might be internal — directing the agent to pass the full URL or set `org='dnceng'` and `project='internal'`.
- **Two new auth-status MCP tools shipped:** `azdo_auth_status` delegates to `IAzdoTokenAccessor.AuthStatusAsync()` (returns `AzdoAuthStatus` record). `helix_auth_status` checks `IHelixTokenAccessor.GetAccessToken()` and resolves `TokenSource` from `ChainedHelixTokenAccessor` when available, returning a `HelixAuthStatus` record.
- **Key file paths:** `src/HelixTool.Mcp.Tools/AzDO/AzdoMcpTools.cs` (TryExtractOrgProjectFromUrl, AppendNotFoundHint, azdo_auth_status), `src/HelixTool.Mcp.Tools/Helix/HelixMcpTools.cs` (helix_auth_status, HelixAuthStatus record).

📌 Team update (2026-05-08): MCP SDK v1.0.0 → v1.3.0 upgrade pending — parallel research (Ash) and inventory (Dallas) complete; recommendation to upgrade to v1.3.0 (low risk, no code changes required); awaiting decision to spawn Ripley for upgrade PR. See `.squad/decisions/inbox/*` and `.squad/log/2026-05-08T20-29-00Z-mcp-sdk-upgrade-research.md` for research details and drift items flagged.

## 2025 — MCP SDK 1.3.0 upgrade + Central Package Management migration
- **Branch:** `squad/mcp-sdk-1.3.0-upgrade`
- **CPM migration pattern (reusable):**
  1. Create `Directory.Packages.props` at repo root with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`.
  2. Aggregate every `PackageReference Version=` from every csproj into `<PackageVersion Include= Version= />` entries.
  3. Strip `Version=` from each `PackageReference` (keep `Include=` and any `PrivateAssets`/`OutputItemType` attrs).
  4. If any project uses floating ranges (`*`, `*-*`), add `<CentralPackageFloatingVersionsEnabled>true</CentralPackageFloatingVersionsEnabled>` — otherwise `NU1011` blocks restore.
  5. CPM surfaces pre-existing `NU1507` (multiple NuGet sources w/o package source mapping) — note as separate cleanup, don't conflate.
- **Version-source pattern (replaces hardcoded `ServerInfo.Version`):**
  ```csharp
  var serverVersion = System.Reflection.Assembly.GetExecutingAssembly()
      .GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion ?? "0.0.0";
  options.ServerInfo = new() { Name = "hlx", Version = serverVersion };
  ```
  Works in both top-level Program.cs (`HelixTool.Mcp`) and class-based commands (`HelixTool`). HelixTool csproj's `<Version>0.5.4</Version>` flows through automatically; HelixTool.Mcp has no `<Version>` so it gets `1.0.0.0` (assembly default), which is fine for an unpacked aspnetcore host.
- **stdio resources bug:** the `hlx mcp` subcommand registered `WithToolsFromAssembly` but not `WithResourcesFromAssembly` — meaning `ci://profiles` and `ci://profiles/{repo}` resources were silently invisible to stdio clients (Claude Desktop, Cline, etc.) while HTTP clients saw them. Always mirror `WithToolsFromAssembly` and `WithResourcesFromAssembly` calls across hosts; treat divergence as a bug.
- **MCP SDK 1.0.0 → 1.3.0:** zero code changes required for our usage (root-endpoint HTTP transport, no manual `RequestContext` construction). Per Ash's research, the 1.2.0 breaking changes (legacy SSE off-by-default, RequestContext ctor obsolete) don't affect us.

## 2026 — MCP progress notifications (issue #43)
- **Branch:** `squad/mcp-progress-notifications` (off main; PR pending).
- **MCP SDK 1.3.0 server-side progress API — verified surface:**
  - **The pattern is auto-injection.** Any tool method parameter typed
    `IProgress<ProgressNotificationValue>` is bound by the SDK at invocation
    time. If the client included a `_meta.progressToken` in `tools/call`, the
    SDK manufactures a forwarding sink whose `Report` calls turn into
    `notifications/progress` JSON-RPC messages with that token. **If the client
    did NOT include a progress token, the parameter is still injected but
    bound to a no-op sink** — so emitting progress is always safe.
  - The parameter is omitted from the tool's JSON schema (clients don't see it).
  - `ProgressNotificationValue` has init-only `float Progress`, `float? Total`,
    `string? Message`. Use object-initializer syntax.
  - Lower-level escape hatch: `McpSession.NotifyProgressAsync(ProgressToken, ProgressNotificationValue, …)` exists for code outside the auto-injection path
    (e.g. when you have a raw `RequestContext`). We did not need it.
- **Architecture decision:** Service layer (`HelixService`, `AzdoService`) stays
  MCP-free. Added `HelixTool.Core/ProgressUpdate.cs` (record struct
  `(double Current, double? Total, string? Message)`) + `ProgressReporter`
  static helpers. The MCP tool layer owns a tiny `McpProgressAdapter` that
  translates `IProgress<ProgressNotificationValue>` → `IProgress<ProgressUpdate>`.
  Adapter returns `null` when the inner sink is `null` so the no-op fast path
  doesn't even allocate.
- **Granularity rule:** ~5–10 emits per long run. `ProgressReporter.ItemStep(total)`
  returns `max(1, total/10)`. `CopyToWithProgressAsync` emits every 10% of total
  (or every 1 MiB when `Content-Length` is missing) plus a 250ms throttle so a
  small file doesn't spam.
- **Tools instrumented:**
  - `helix_download` → per-file ("Downloaded N of M files: <name>") for the
    work-item path; chunked bytes ("Downloaded 42 of 128 MB") for the URL path.
  - `helix_find_files` → "Scanned N of M work items (K matches)" every ~10%.
  - `azdo_search_log` → "Searched N of M log steps (K matches)" every ~10%.
  - `azdo_log` skipped — confirmed it's a single fetch, no streaming.
- **Smoke test (file-based C# app under throwaway `scratch/` dir, deleted after):**
  Spun up an in-proc HttpListener serving 4 MiB in 256 KiB chunks with 40ms
  delays; called `ProgressReporter.CopyToWithProgressAsync` directly; confirmed
  4 events fired (0%, ~45%, ~89%, 100% — the 250ms throttle suppressed
  intermediate emits, which is correct behavior). Output bytes match input.
- **Backward compat:** All public service signatures got an *optional*
  `IProgress<ProgressUpdate>?` parameter inserted **before** the
  `CancellationToken`. This is a binary-compatible source change for callers
  using named args, but **breaks any caller that passed `cancellationToken`
  positionally as the next argument**. Fixed three internal call sites
  (`FindBinlogsAsync`, two `DownloadFilesAsync` calls inside HelixService,
  one test in `DownloadTests.cs`) by passing `progress: null,` explicitly.
  Lesson: append-at-end is gentler; insert-before-CT only works because we
  control all callers. **Future progress params should still go before CT to
  keep CT visually last.**
- **Working-tree gotcha (one-time):** Mid-task the working tree was switched
  out from under me to `squad/mcp-tool-annotations-and-cleanup` (the parallel
  branch) — likely the parallel agent grabbed the same checkout. My MCP tool
  edits got blown away. Recovered by salvaging diffs to a `.salvage/` dir,
  switching back to `squad/mcp-progress-notifications`, and reapplying. **In
  shared workspaces, `git worktree list` early so each squad member can spawn
  a separate worktree per branch.**
## Learnings — Release process (v0.6.0, 2026-05-08)
- **Version lives in two files only** — bump both:
  - `src/HelixTool/HelixTool.csproj` → `<Version>X.Y.Z</Version>`
  - `src/HelixTool/.mcp/server.json` → both `.version` (top-level) and `.packages[0].version`
- **No `CHANGELOG.md` convention.** Prior releases (v0.5.0 onward) used the release-commit message and the GitHub Release body as the changelog. Don't invent one.
- **Release notes drafts** now live under `.squad/release-notes/vX.Y.Z.md` (introduced this release for use with `gh release create --notes-file`). Reuse this path going forward.
- **Tag format:** `vMAJOR.MINOR.PATCH` (e.g. `v0.6.0`). Tag triggers `.github/workflows/publish.yml` (`on: push: tags: v*`) which validates that the tag version matches both `HelixTool.csproj` `<Version>` and the two `server.json` versions before publishing — any mismatch fails the publish job.
- **Release commit subject style:** `release: vX.Y.Z — <one-line scope>`.
- **Release branch convention:** `squad/release-vX.Y.Z` PR'd into `main`. Tag is created on the merge commit AFTER the PR lands, never pre-tag.
- **Co-author trailer** required on the release commit: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`.
- **CI workflows of interest:** `ci.yml` (build/test on PR), `publish.yml` (tag → NuGet/MCP registry publish), `squad-release.yml` (squad-side coordination — not inspected this round).

## Learnings — Pagination Standardization Implementation (2026-05-20, commit 1a2e1d0)

**Files changed:**
- `src/HelixTool.Mcp.Tools/AzDO/AzdoMcpTools.cs` — wrapped azdo_changes and azdo_test_runs in CreateLimitedResults()
- `src/HelixTool.Mcp.Tools/McpToolResults.cs` — added truncated+note fields to 5 Helix result types
- `src/HelixTool.Core/AzDO/AzdoModels.cs` — added truncated+note fields to 3 AzDO result types (HelixJobsFromBuildResult, TimelineSearchResult, BuildAnalysisResult)
- `src/HelixTool.Core/Helix/HelixService.cs` — added FindFilesResults wrapper record, updated FindFilesAsync to return it with truncation metadata
- `src/HelixTool.Mcp.Tools/Helix/HelixMcpTools.cs` — wired truncation logic for helix_find_files
- `src/HelixTool/Program.cs` — updated CLI find-files command to show truncation warning
- `src/HelixTool.Tests/Helix/WorkItemDetailTests.cs` — fixed test to match new FindFilesResults shape

**Build:** ✅ Succeeded (dotnet build passed clean)  
**Branch:** `squad/pagination-standardize` (created from main)  
**Commit:** `1a2e1d0` — "Standardize pagination across MCP tools (Phase 1+2)"

**Deviations from Dallas's spec:** None — all 2 🔴 tools wrapped, all 8 🟡 bespoke result types updated with truncated+note fields (helix_search and azdo_search_log already had truncation, as noted in spec).

**Pattern learned:** When changing service-layer return types, remember to:
1. Update the record definition (or add new wrapper)
2. Update the service method signature + implementation
3. Update all tool/CLI call sites
4. Update tests that directly call the service
5. Clean build to avoid stale reference errors

📌 Team update (2026-05-21): Pagination Phase 1+2 implemented — wrapped `azdo_changes`/`azdo_test_runs` in `LimitedResults<T>`, added `truncated`/`note` to 8 result types. Build clean (0 warnings, 0 errors). Commit 0a82e58. Full suite: 1180/1180 passing. ⚠️ BRANCH-HYGIENE: committed to local main instead of squad/pagination-standardize per manifest instruction; Larry will handle branch/push decision.

## Learnings — RollForward policy for global tool (2026-05-21)

- Set `<RollForward>Major</RollForward>` only in `src/HelixTool/HelixTool.csproj` so the generated `HelixTool.runtimeconfig.json` for the `hlx` global tool can run on the next installed major runtime when `net10.0` is missing.
- Do not add `RollForward` to library projects; this startup policy is only consumed by the executable entry point/runtimeconfig.
