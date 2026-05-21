# History Archive — Lambert

📌 Team update (2026-02-11): US-10 (GetWorkItemDetailAsync) and US-23 (GetBatchStatusAsync) implemented — new CLI commands work-item and batch-status, MCP tools hlx_work_item and hlx_batch_status added. — decided by Ripley

📌 Team update (2026-02-11): US-21 failure categorization implemented — FailureCategory enum + ClassifyFailure heuristic classifier added to HelixService. WorkItemResult/WorkItemDetail records expanded. — decided by Ripley

## Sessions (2026-02-12)

**Cache tests (L-CACHE-1 through L-CACHE-10):** 56 tests across 3 files. CachingHelixApiClientTests (26 unit), SqliteCacheStoreTests (18 integration), CacheOptionsTests (12 unit). Key patterns: temp dirs with GUID for SQLite integration tests, sequential `.Returns()` for cache miss→hit flow, private DTOs for JSON round-tripping. Test count 126 → 182.

**Cache security tests:** 24 tests in CacheSecurityTests.cs. ValidatePathWithinRoot (7), SanitizePathSegment (6), SanitizeCacheKeySegment (5), SqliteCacheStoreSecurityTests (2 integration), CachingHelixApiClientSecurityTests (3 integration). DB tampering test requires dispose→WAL checkpoint→reopen cycle. Test count 182 → 206.

**HTTP/SSE auth tests (L-HTTP-1 through L-HTTP-5):** 46 tests across 5 files. HelixTokenAccessorTests (5), HelixApiClientFactoryTests (5), CacheStoreFactoryTests (8 incl. thread safety), SqliteCacheStoreConcurrencyTests (10), HttpContextHelixTokenAccessorTests (17). Total: 252 tests, all passing.

## Learnings

- CachingHelixApiClient constructor: 3-arg `(IHelixApiClient, ICacheStore, CacheOptions)`. `_enabled = options.MaxSizeBytes > 0`.
- Console log cache miss: decorator calls inner, stores via SetArtifactAsync, disposes original, returns GetArtifactAsync. Mock needs `.Returns(null, stream)`.
- CacheStoreFactory: IDisposable, ConcurrentDictionary, GetOrAdd with key = AuthTokenHash ?? "public"
- HttpContextHelixTokenAccessor tests: IDisposable pattern saves/restores HELIX_ACCESS_TOKEN env var per test
- NSubstitute gotcha: `GetMetadataAsync` default return is empty string (not null) — must explicitly return `Task.FromResult<string?>(null)` for cache miss


📌 Team update (2026-02-13): HTTP/SSE multi-client auth architecture decided — scoped DI with IHelixTokenAccessor, IHelixApiClientFactory, ICacheStoreFactory. Affects test infrastructure for auth-related tests. — decided by Dallas
📌 Team update (2026-02-13): Multi-auth support deferred — single-token-per-process model retained. No additional multi-auth test coverage needed. — decided by Dallas

- US-6 DownloadTests: 46 tests written in `DownloadTests.cs` across 4 test classes. Test count 252 → 298.
- DownloadFilesTests (27 tests): happy path single/multi-file download, pattern matching (*.binlog, *.trx, *, specific name, case-insensitive), empty results (no match, no files), correct temp dir placement, path traversal protection (forward slash, backslash, `..`), empty file streams, binary content preservation, same-name file overwrite, URL-based job ID resolution, input validation (null/empty/whitespace jobId and workItem), error handling (404, 401, 403, server error, timeout, cancellation).
- DownloadFromUrlParsingTests (5 tests): argument validation (null, empty, whitespace), invalid/relative URL format, URL-encoded character parsing. Cannot mock static HttpClient — tests verify argument validation and URI parsing only.
- DownloadSanitizationTests (6 tests): normal filename preserved, forward slash sanitized, `..` sanitized, path traversal stays within outDir, spaces preserved, unicode preserved.
- DownloadPatternTests (8 tests): Theory with 4 InlineData for extension/wildcard/substring patterns, default pattern downloads all, case-insensitive extension matching, case-insensitive substring matching.
- Key pattern: Each test class that writes to disk uses a UNIQUE ValidJobId constant (different GUID) to avoid temp directory collisions during parallel xUnit execution. File contention was observed when all classes shared the same GUID — `helix-{idPrefix}` dir was shared.
- DownloadFilesAsync flow: ListWorkItemFilesAsync → filter with MatchesPattern → create `helix-{id[..8]}` temp dir → foreach file: GetFileAsync → SanitizePathSegment(Path.GetFileName(name)) → ValidatePathWithinRoot → File.Create → CopyToAsync.
- DownloadFromUrlAsync uses static `s_httpClient` — only testable for argument validation and URI parsing. HTTP errors (401/403/404/timeout) cannot be tested without an HTTP mock or test server.
- NSubstitute lambda pattern for streams: `.Returns(_ => new MemoryStream(...))` — lambda needed so each call gets a fresh stream instance. Sequential `.Returns(first, second)` works for overwrite tests.

📌 Team update (2026-02-13): US-9 script removability analysis complete — 100% core API coverage, Phase 1 migration can proceed with zero blockers — decided by Ash

📌 Team update (2026-02-13): Requirements audit complete — 25/30 stories implemented, US-22 structured test failure parsing is only remaining P2 gap — audited by Ash
📌 Team update (2026-02-13): MCP API design review — 6 actionable improvements identified (P0: batch_status array fix, P1: add hlx_list_work_items, P2: naming, P3: response envelope) — reviewed by Dallas
📌 Team update (2026-02-13): Generalize hlx_find_binlogs to hlx_find_files with pattern parameter — update existing FindBinlogsAsync tests, add FindFilesAsync tests with various patterns — decided by Dallas

- MCP camelCase migration: `s_jsonOptions` now uses `PropertyNamingPolicy = JsonNamingPolicy.CamelCase` — all JSON property assertions in tests must use camelCase (`name`, `uri`, `exitCode`, `state`, `machineName`) not PascalCase
- `FindBinlogs` MCP tool delegates to `FindFiles` — JSON output uses `"files"` key (not `"binlogs"`) and includes `"pattern"` field
- `FileEntry` simplified to `(string Name, string Uri)` — no more `IsBinlog`/`IsTestResults` boolean tags; classification done at MCP layer via `MatchesPattern`
- `BatchStatus` MCP tool accepts `string[]` not comma-separated string — test with `new[] { id1, id2 }`
- `Status` parameter renamed from `all` to `includePassed` — update all test call sites accordingly
- Test count 298 → 304: fixed 8 camelCase failures, added 6 new tests (FindFiles pattern/wildcard, FindBinlogs delegation, BatchStatus array, GetWorkItemFiles simple FileEntry, FindFilesAsync pattern filtering)

## 2026-02-15: Cross-agent note from Scribe

- **Decision merged:** "camelCase JSON assertion convention" (Lambert, 2026-02-13) — all MCP test assertions must use camelCase property names.
- **Decision merged:** "MCP API Batch — Tests Need CamelCase Update" (Ripley, 2026-02-15) — tests referencing PascalCase JSON props or `binlogs` key need updating to camelCase and `files`.

## 2026-02-15: Security Validation Tests (P1 Threat Model)

**SecurityValidationTests.cs** — 18 tests covering threat-model findings E1 and D1:

- **URL scheme validation (E1):** 10 tests — HTTPS/HTTP accepted (no ArgumentException), file:///ftp://data:/javascript:/ssh:// all throw ArgumentException, null/empty throw, no-scheme throws (UriFormatException or ArgumentException both acceptable).
- **Batch size limit (D1):** 5 tests — MaxBatchSize const = 50 verified, single job accepted, 50 boundary accepted, 51 throws ArgumentException, 200 throws, empty throws.
- **MCP tool enforcement:** 2 tests — hlx_batch_status rejects 51 IDs, accepts 50 IDs with correct JSON output.
- **Total test count:** 304 → 322 (all passing).

## Learnings

- URL scheme validation in `DownloadFromUrlAsync` runs after `new Uri(url)` — a schemeless string throws `UriFormatException` from the Uri constructor before scheme validation can run. Tests for "no scheme" should accept both `ArgumentException` and `UriFormatException`.
- `HelixService.MaxBatchSize` is `internal const int` — accessible from tests via `InternalsVisibleTo`.
- Security test pattern: for HTTP scheme acceptance tests, use `Record.ExceptionAsync` + `Assert.IsNotType<ArgumentException>` rather than asserting no exception (the method will still fail with network errors, which is fine — we only care that it wasn't rejected at the validation layer).
- Batch boundary tests need mock setup for all N job IDs — use `Enumerable.Range` with formatted GUID strings (`$"{i:x8}-0000-0000-0000-000000000000"`) for bulk generation.

📌 Team update (2026-02-13): P1 security fixes E1+D1 implemented (URL scheme validation, batch size cap, MCP description) — decided by Ripley


📌 Team update (2026-02-13): Remote search design — 2 new tools (hlx_search_file, hlx_test_results) designed with 8 decisions pending Larry's review. US-31/US-32 created. Lambert to write tests for search logic and TRX parsing (including XXE, oversized, malformed .trx files) — decided by Dallas

## 2026-02-15: US-31 SearchFileAsync Tests (Phase 1)

**SearchFileTests.cs** — 17 tests covering SearchFileAsync and config toggle:

- **Input validation (3 tests, Theory with 3 InlineData each):** null/empty/whitespace jobId, workItem, fileName all throw ArgumentException
- **Config toggle (2 tests):** SearchFileAsync and SearchConsoleLogAsync both throw InvalidOperationException when HLX_DISABLE_FILE_SEARCH=true. Env var set/reset in try/finally.
- **Binary file detection (1 test):** file content with null bytes → IsBinary=true, empty matches
- **Basic search (3 tests):** simple pattern match (correct line numbers, 1-based), case-insensitive matching ("ERROR" matches "error"), context lines (1 before + match + 1 after)
- **Max matches (1 test):** maxMatches=3 limits results, Truncated=true
- **No matches (1 test):** pattern not found → empty matches, IsBinary=false, Truncated=false
- **Total test count:** 322 → 339 (all passing)

## Learnings

- SearchFileAsync mock setup: requires both `ListWorkItemFilesAsync` (return IWorkItemFile list) and `GetFileAsync` (return stream). The DownloadFilesAsync flow filters by MatchesPattern, so the mock file name must match the fileName parameter exactly.
- Binary detection: null byte (0x00) anywhere in first 8KB triggers IsBinary=true. Use raw `byte[]` with `SetupFileBytes` helper.
- Config toggle test pattern: set env var before call, reset in finally block. Both SearchFileAsync and SearchConsoleLogAsync check `IsFileSearchDisabled` before argument validation.
- Truncated flag: set when `Matches.Count >= maxMatches` — tests can verify this by providing fewer maxMatches than matching lines.
- Each test class uses a UNIQUE ValidJobId GUID to avoid temp directory collisions during parallel xUnit execution (established pattern from DownloadTests).


📌 Team update (2026-02-13): HLX_DISABLE_FILE_SEARCH config toggle added as security safeguard for disabling file content search operations — decided by Larry Ewing (via Copilot)

📌 Team update (2026-02-13): US-31 hlx_search_file Phase 1 implemented (SearchFileAsync, MCP tool, CLI command, config toggle) — decided by Ripley

## 2026-02-15: US-32 TRX Parsing Tests

**TrxParsingTests.cs** — 15 tests covering ParseTrxResultsAsync (TRX file parsing):

- **Input validation (2 tests, Theory with 3 InlineData each):** null/empty/whitespace jobId and workItem both throw ArgumentException
- **Config toggle (1 test):** ParseTrxResultsAsync throws InvalidOperationException when HLX_DISABLE_FILE_SEARCH=true
- **Basic TRX parsing (3 tests):** mixed results with correct Passed/Failed/Skipped counts, failed tests include ErrorMessage+StackTrace, default (includePassed=false) excludes passed tests from Results list
- **Include passed (1 test):** includePassed=true returns all 3 results (Passed, Failed, NotExecuted)
- **Max results (1 test):** maxResults=1 limits output to single result
- **Error truncation (1 test):** ErrorMessage >500 chars truncated with "... (truncated)" suffix, StackTrace >1000 chars truncated similarly
- **No TRX files (1 test):** when no .trx files found, throws HelixException
- **XXE prevention (1 test):** DTD declaration in TRX XML causes XmlException (DtdProcessing.Prohibit)
- **Total test count:** 349 → 364 (all passing)

## Learnings

- ParseTrxResultsAsync uses DownloadFilesAsync internally — mock setup same as SearchFileTests: ListWorkItemFilesAsync (return IWorkItemFile list with .trx name) + GetFileAsync (return MemoryStream with TRX XML). The DownloadFilesAsync flow writes to disk, ParseTrxFile reads from disk files.
- Ripley's TRX implementation landed before tests — proactive test writing pattern still works, just needed minor confirmation that signatures matched spec.
- TRX outcome classification: "Passed" → passed++, "Failed" → failed++, everything else (including "NotExecuted") → skipped++. Tests for non-pass/non-fail outcomes always included in Results regardless of includePassed flag.
- Error truncation in ParseTrxFile: only extracts ErrorInfo for outcome="Failed" (case-insensitive). Truncation adds "... (truncated)" suffix — total length is limit + 15 chars for suffix.
- XmlReaderSettings includes `MaxCharactersInDocument = 50_000_000` and `Async = true` beyond the DTD/resolver settings. XmlException thrown by `XDocument.Load(reader)` when DTD encountered.

## 2026-02-15: Status API Filter Migration Tests

**HelixMcpToolsTests.cs** — Updated 2 existing tests and added 5 new tests for `filter: string` parameter migration:

- **Renamed:** `Status_AllFalse_PassedIsNull` → `Status_FilterFailed_PassedIsNull` (uses `filter: "failed"`)
- **Renamed:** `Status_AllTrue_PassedIncludesItems` → `Status_FilterAll_PassedIncludesItems` (uses `filter: "all"`)
- **New:** `Status_DefaultFilter_ShowsOnlyFailed` — verifies default (no filter arg) shows only failed, passed is null
- **New:** `Status_FilterPassed_FailedIsNull` — verifies `filter: "passed"` nulls out failed, populates passed
- **New:** `Status_FilterPassed_IncludesPassedItems` — verifies passed items have expected structure (name, exitCode, state, machineName)
- **New:** `Status_FilterCaseInsensitive` — verifies `filter: "ALL"` (uppercase) populates both failed and passed
- **New:** `Status_InvalidFilter_ThrowsArgumentException` — verifies invalid filter value throws ArgumentException
- **Total test count:** 364 → 369 (15 status tests total, all passing). 1 pre-existing failure in SearchConsoleLogAsync unrelated to changes.

## Learnings

- Status API `filter` parameter accepts "failed" (default), "passed", "all" — case-insensitive. Invalid values throw ArgumentException.
- `filter: "passed"` nulls `failed` array (mirrors `filter: "failed"` nulling `passed` array). `filter: "all"` populates both.
- Proactive test writing pattern continues to work: wrote tests against new `filter` API spec before Ripley's code landed, waited for build to succeed.

## Cache Concurrency Audit (2026-02-15)

### Production Code Concurrency Patterns (SqliteCacheStore.cs)
- **Connection-per-operation:** Each method opens and closes its own `SqliteConnection` via `OpenConnection()`. No shared connection → inherently thread-safe at the .NET level.
- **WAL mode:** `PRAGMA journal_mode=WAL;` set during `InitializeSchema()` (line 73). Enables concurrent reads across processes, single writer at a time.
- **busy_timeout:** `PRAGMA busy_timeout=5000;` set per-connection in `OpenConnection()` (line 44). SQLite retries for 5 seconds when encountering a write lock.
- **Cache=Shared:** Connection string includes `Cache=Shared` (line 31). Multiple in-process connections share a single SQLite page cache.
- **Atomic artifact writes:** `SetArtifactAsync` uses write-to-temp-then-rename pattern (lines 195-207). Temp file has GUID suffix to avoid collisions. `File.Move(temp, target, overwrite: true)` is atomic on most filesystems. Fallback on Windows `IOException`/`UnauthorizedAccessException` deletes temp and tolerates failure.
- **Artifact read sharing:** `GetArtifactAsync` opens `FileStream` with `FileShare.ReadWrite | FileShare.Delete` (line 177). Allows concurrent readers and allows eviction (deletion) while readers hold the file open.
- **LRU eviction:** `EvictLruIfOverCapAsync` is called at the end of `SetArtifactAsync` (line 227). It reads the full artifact list, selects LRU candidates, then deletes files and rows one by one in `DeleteArtifactRows`. File deletion uses `File.Delete` with `IOException` catch.
- **No transaction wrapping on eviction:** `DeleteArtifactRows` deletes file, then deletes SQLite row, with no transaction. A crash between these two steps leaves an orphan row (stale row cleanup exists in `GetArtifactAsync` via `File.Exists` check).
- **No lock between size-check and eviction:** `EvictLruIfOverCapAsync` reads total size, then evicts. A concurrent writer can insert between these two steps, meaning the cache can temporarily exceed `MaxSizeBytes`.

### Key File Paths
- `src/HelixTool.Core/Cache/SqliteCacheStore.cs` — Main cache store, all concurrency patterns
- `src/HelixTool.Core/Cache/CachingHelixApiClient.cs` — Decorator calling SetArtifactAsync/GetArtifactAsync, drives the download-then-cache flow
- `src/HelixTool.Core/Cache/ICacheStoreFactory.cs` — ConcurrentDictionary-based factory, one SqliteCacheStore per auth token hash
- `src/HelixTool.Core/HelixService.cs:321-374` — DownloadFilesAsync: downloads to temp dir (not cache), uses File.Create (not atomic)
- `src/HelixTool.Tests/SqliteCacheStoreConcurrencyTests.cs` — 14 tests: 10 original multi-thread concurrency + 4 new gap tests (stale row cleanup, eviction-during-read, concurrent eviction+write integrity, same-key race)
- `src/HelixTool.Tests/SqliteCacheStoreTests.cs` — 18 CRUD/eviction tests, single-threaded
- `src/HelixTool.Tests/CacheStoreFactoryTests.cs` — 8 tests: factory thread safety, instance identity

## Cache Concurrency Gap Tests (2026-02-15)

**SqliteCacheStoreConcurrencyTests.cs** — 4 new tests added (10 → 14 total):

- **StaleRowCleanup_FileDeletedFromDisk_ReturnsNullAndCleansUp:** Verifies orphan SQLite row cleanup when artifact file is deleted externally. First `GetArtifactAsync` detects missing file, deletes orphan row, returns null. Second call confirms row is gone (fast null).
- **EvictionDuringRead_OpenStreamRemainsReadable:** Verifies `FileShare.Delete` behavior — opens a read stream, triggers LRU eviction via small `MaxSizeBytes`, confirms the already-opened stream reads complete, uncorrupted 1KB data.
- **ConcurrentEvictionAndWrite_ArtifactIntegrity:** Stress test with 2048-byte cap, 20 concurrent 256-byte writes triggering frequent LRU eviction plus 20 concurrent reads. Tolerates `FileNotFoundException` (known race between `File.Exists` and `FileStream` open in `GetArtifactAsync`). All successful reads verified for fill-byte integrity.
- **ConcurrentCachingClientSimulation_SameKey:** Two concurrent `SetArtifactAsync` on same key with different fill bytes ('A' vs 'B'). Verifies result is exactly 128 bytes of one consistent fill byte — no partial/mixed writes.

## Learnings

- `CacheOptions.GetEffectiveCacheRoot()` appends `/public` (no auth) or `/cache-{hash}` (auth) under `CacheRoot`. Tests that reference the artifacts directory must use `_opts.GetEffectiveCacheRoot()` not `_tempDir` directly.
- Known production race in `GetArtifactAsync`: `File.Exists` check (line 160) and `FileStream` open (line 177) are not atomic. Under concurrent eviction, the file can be deleted between these two calls, causing `FileNotFoundException`. Concurrency tests must tolerate this as a known gap (catch `FileNotFoundException`).
- `DeleteArtifactRows` catches `IOException` on `File.Delete` but not `UnauthorizedAccessException`. Under concurrent eviction + read on Windows, `File.Delete` can throw `UnauthorizedAccessException` when another thread holds the file open. Concurrency stress tests must tolerate both `IOException` and `UnauthorizedAccessException`.
- Tests using small `MaxSizeBytes` to trigger LRU eviction must create their own `CacheOptions`/`SqliteCacheStore` instances (not modify shared `_opts`/`_store`) to avoid interfering with other tests running in parallel.
- Write-to-temp-then-rename pattern in `SetArtifactAsync` ensures same-key concurrent writes produce complete, uncorrupted artifacts — the atomic `File.Move(overwrite: true)` guarantees one writer wins cleanly.


📌 Team update (2026-02-15): DownloadFilesAsync temp dirs now per-invocation (helix-{id}-{Guid}) to prevent cross-process races — decided by Ripley
📌 Team update (2026-02-15): CI version validation added to publish workflow — tag is source of truth for package version — decided by Ripley

## Archived from history.md (2026-03-08 summarization)

### Old team updates (2026-02-11 through 2026-02-15)
📌 Team update (2026-02-11): US-10/US-23 implemented — decided by Ripley
📌 Team update (2026-02-11): US-21 failure categorization — decided by Ripley
📌 Team update (2026-02-13): HTTP/SSE multi-client auth — decided by Dallas
📌 Team update (2026-02-13): Multi-auth deferred — decided by Dallas
📌 Team update (2026-02-13): US-9 script removability — decided by Ash
📌 Team update (2026-02-13): Requirements audit — audited by Ash
📌 Team update (2026-02-13): MCP API design review — reviewed by Dallas
📌 Team update (2026-02-13): hlx_find_files generalization — decided by Dallas
📌 Team update (2026-02-13): P1 security fixes E1+D1 — decided by Ripley
📌 Team update (2026-02-13): Remote search design — decided by Dallas
📌 Team update (2026-02-13): HLX_DISABLE_FILE_SEARCH toggle — decided by Larry Ewing
📌 Team update (2026-02-13): US-31 hlx_search_file — decided by Ripley
📌 Team update (2026-02-13): Status filter changed — decided by Larry/Ripley
📌 Team update (2026-02-15): DownloadFilesAsync per-invocation temp dirs — decided by Ripley
�� Team update (2026-02-15): CI version validation — decided by Ripley

### Old learnings (pre 2026-02-22)
- **ParseTrxResultsAsync auto-discovery:** Production code now tries `*.trx` first, falls back to `*.xml`, then throws `HelixException` with work-item name in the message. Two error paths: "No test result files found" (no files at all) and "Found XML files but none were in a recognized format" (files found but unrecognizable).
- **SetupMultipleFiles mock pitfall:** Files with `null` content don't configure `GetFileAsync`. If `DownloadFilesAsync` matches them by pattern, the null stream causes `NullReferenceException`. Always use non-matching extensions (`.binlog`, `.log`) for "no files found" tests, or provide actual content for downloadable files.
- **MCP error surfacing pattern:** `HelixMcpTools.TestResults` currently lets `HelixException` propagate uncaught. Use `Record.ExceptionAsync` + message assertions (not exception type) to write tests that pass both before and after a try/catch wrapper is added. Comment out `Assert.IsType<McpException>` as a contract marker.


## Archived from history.md (2026-03-09)

### 2026-03-07: AzDO Security Tests (63 tests)
- **AzdoSecurityTests** in `src/HelixTool.Tests/AzDO/AzdoSecurityTests.cs` — 63 tests across 5 categories:
  - AzdoIdResolver malicious URL inputs (embedded credentials, non-AzDO hosts/SSRF, path traversal, query injection, unicode, long URLs, scheme attacks, integer overflow)
  - AzCliAzdoTokenAccessor command injection safety (no shell execute, env var passthrough, CLI failure resilience)
  - AzdoApiClient request construction (SSRF prevention via host assertion, token leakage in errors, special chars in org/project, null/empty token)
  - CachingAzdoApiClient cache isolation (org/project key separation, azdo: prefix, no tokens in cached data, cache key poisoning via path traversal/colons, disabled cache)
  - AzdoService end-to-end (malicious URLs rejected before API call, null/empty/invalid inputs)
- **Security test patterns:** Token leakage (DoesNotContain on error), SSRF (host assertion), Cache isolation (different keys for org/project), No-API-call guard (DidNotReceive after rejection)
- **Edge cases:** HttpUtility.ParseQueryString comma concat safe via int.TryParse, Uri credential parsing safe (Host/Path only), Uri normalizes traversal, long.MaxValue fails int.TryParse safely, newlines in AZDO_TOKEN mitigated by AuthenticationHeaderValue
- **Total:** 594 tests (531 + 63 new).

### 2026-03-08: AzDO Artifact & Attachment Tests (33 tests)
- **AzdoArtifactTests** in `src/HelixTool.Tests/AzDO/AzdoArtifactTests.cs` — 33 tests: API Client (artifacts/attachments), Service Layer (URL resolution, top param), Caching (miss/hit, TTL 4h artifacts/1h attachments, azdo: prefix), MCP Tools (list/URL/empty), Edge Cases (invalid input, 2GB file, JSON round-trip)
- CamelCase JSON: `root.GetProperty("camelCaseName").GetXxx()` avoids xUnit2002
- Artifacts use ImmutableTtl (4h), TestAttachments use TestTtl (1h)
- **Total:** 700 tests (667 + 33 new).

### 2026-03-08: Proactive Tests for SEC-2/3/4 and AzDO CLI (53 tests)
- **HttpClientConfigurationTests** (13): null-guard, timeout range validation, timeout vs cancellation behavior, IHttpClientFactory pattern
- **StreamingBehaviorTests** (18): empty/large streams, tailLines edges, connection errors, stream disposal, special chars, input validation
- **AzdoCliCommandTests** (22): build summary, timeline, build log, changes, test runs/results, list builds, artifacts
- NSubstitute: `.Returns<Stream>(_ => throw new Ex())` for exception testing; init-only properties need object initializer
- AzdoBuildChange.Author is AzdoChangeAuthor (not AzdoIdentityRef)
- **Total:** 753 tests (700 + 53 new).

### 2026-03-08: AzDO Search Log & TextSearchHelper Tests (41 tests)
- **TextSearchHelperTests** (20): basic matching, context lines, case insensitivity (Theory), edge cases, max matches, overlapping context, large content (10K lines), special chars (literal not regex)
- **AzdoSearchLogTests** (21): happy path, no matches, context, max matches, large log, special chars, URL resolution, case insensitivity, input validation, null log, search disabled env var, result identifier
- TextSearchHelper: pure static class, 5 required params, no defaults
- AzdoService.SearchBuildLogAsync delegates to TextSearchHelper, uses IsFileSearchDisabled guard
- Env var test pattern: save/set/try-finally-restore for HLX_DISABLE_FILE_SEARCH
- **Total:** 791 tests (750 + 41 new).

### PR #10 Review Fix: Test Parallelism for Env Var Tests
- Added `[Collection("FileSearchConfig")]` to AzdoSearchLogTests for env var mutation safety
- Added FileSearchConfigCollection.cs with `[CollectionDefinition("FileSearchConfig", DisableParallelization = true)]`
- Convention: all classes mutating HLX_DISABLE_FILE_SEARCH must use this collection

### AzDO Search Timeline Tests (19 tests)
- **AzdoSearchTimelineTests** (19): name/issue matching, record type filtering, result filtering (all/failed/default), empty/null handling, input validation, parent name resolution, duration formatting, edge cases
- SearchTimelineAsync returns TimelineSearchResult (in AzdoModels.cs), null timeline throws InvalidOperationException
- Default resultFilter is "failed"; FormatDuration: >1h "Xh Ym", >1m "Xm Ys", else "Xs"
- Tests written in parallel with Ripley's implementation, adapted from tuple to final TimelineSearchResult class

## 2026-03-13: Archived from history.md during summarization

### Redundant test cleanup (PR #15)
- **Deleted `AzdoCliCommandTests.cs`** (22 tests → 19 removed, 3 rescued): The file was written proactively for CLI subcommands that were never implemented. 19 of 22 tests were near-identical duplicates of `AzdoServiceTests` — same mock setup, same assertions, just different variable names. Rescued 3 unique tests (artifact default/pattern filtering, changes with top parameter) into `AzdoServiceTests.cs`.
- **Removed 3 "ImplementsInterface" / "Constructor_Accepts" tests**: `HelixApiClientFactoryTests.ImplementsIHelixApiClientFactory`, `HttpContextHelixTokenAccessorTests.ImplementsIHelixTokenAccessor`, `HelixMcpToolsTests.Constructor_AcceptsHelixService`. These are compile-time guarantees — if the class doesn't implement the interface, the project won't build.
- **Merged 2 overlapping filter tests** in `HelixMcpToolsTests`: `Status_FilterFailed_PassedIsNull` and `Status_DefaultFilter_ShowsOnlyFailed` tested the same behavior (default filter is "failed"). Combined into one test that verifies both the default and explicit "failed" filter.
- **Pattern observed**: Proactive test files written before production code tends to produce near-duplicates of the actual test file once it lands. Worth catching during PR review.
- **Test count**: 864 → 844 (net -20 tests removed). All 844 pass.

📌 Team updates (2026-03-09 – 2026-03-10 summary): CI profile analysis — 14 tool description/error message recommendations (Ash). Test quality review — net -17 tests, zero coverage loss, prune proactive tests when real tests land (Dallas). CiKnowledgeService expanded to 9 repos, 5 tool descriptions updated (Ripley).

### CiKnowledgeService enrichment tests (2025-07-25)
- **Expanded `CiKnowledgeServiceTests.cs`** from ~23 tests (14 [Fact] + 9 [Theory] cases) to 57 test methods with 159 InlineData entries covering all 9 repos.
- **New repo coverage:** maui, macios, android — profile lookup by short name, full path (`dotnet/maui`, `xamarin/macios`, `xamarin/android`), case insensitivity (`MAUI`, `Macios`, `ANDROID`).
- **Enriched property tests (all 9 repos via Theory):** TestFramework, TestRunnerModel, WorkItemNamingPattern, KnownGotchas, RecommendedInvestigationOrder, PipelineNames, UploadedFiles, CommonFailureCategories — all verified non-empty.
- **OrgProject correctness:** devdiv/DevDiv for macios + android, dnceng-public/public for the other 7.
- **UsesHelix matrix:** Theory covering all 9 repos with expected bool values.
- **ExitCodeMeanings split:** non-empty for Helix repos + vmr, empty for macios/android (no Helix = no exit codes).
- **Edge cases:** maui has 3 pipelines verified, macios/android KnownGotchas warn about devdiv, android mentions fork PRs, roslyn has empty HelixTaskNames, efcore has lowercase 'Send job to helix'.
- **FormatProfile rendering:** KnownGotchas section renders for new repos, ExitCodes section omitted when empty, OrgProject/TestFramework rendered, Maui guide lists all 3 pipelines.
- **GetOverview:** 9 repos in table, devdiv warning present, OrgProject column has both orgs, Quick Reference table format verified.
- **DisplayName correctness:** xamarin/macios, dotnet/android, dotnet/dotnet (VMR) — verifies non-dotnet org display names.
- **Key patterns:** [Theory] with all 9 repos for property-existence tests, [Fact] for repo-specific behavioral assertions. No mocking needed — CiKnowledgeService is pure static data.
- **Test count:** 1038 total (was ~1020 before enrichment, net +~18 test methods but many more test cases via InlineData).

📌 Team update (2026-03-10): CiKnowledgeService expanded from 6 stubs to 9 full repo profiles with 9 new properties. 5 MCP tool descriptions updated with repo-specific CI knowledge. Future test work should cover the enriched CiRepoProfile fields. — decided by Ripley
# Lambert — History

## Project Learnings (from import)
- **Project:** hlx — Helix Test Infrastructure CLI & MCP Server
- **User:** Larry Ewing
- **Stack:** C# .NET 10, ConsoleAppFramework, ModelContextProtocol, Microsoft.DotNet.Helix.Client
- **Test project:** `src/HelixTool.Tests/HelixTool.Tests.csproj` — xUnit, net10.0, references HelixTool.Core and HelixTool.Mcp
- **Testable units:** HelixIdResolver (pure functions), MatchesPattern (internal static via InternalsVisibleTo), HelixService (via NSubstitute mocks of IHelixApiClient), HelixMcpTools (through HelixService)

## Core Context

- **Test stack:** `src/HelixTool.Tests/HelixTool.Tests.csproj` targets net10.0 with xUnit + NSubstitute; Helix tests live under `src/HelixTool.Tests/Helix/`, AzDO tests under `src/HelixTool.Tests/AzDO/`, and shared coverage stays at the test-project root.
- **Assertion conventions:** MCP-surface tests assert camelCase JSON names, env-var mutation tests use `[Collection("FileSearchConfig")]`, and disk-writing tests use unique GUID-based temp roots/job IDs to avoid parallel contention.
- **Mocking seams:** mock `IHelixApiClient` / `IAzdoApiClient` plus their projection interfaces, use fresh-stream lambdas for file/download tests, and prefer focused test runs before the full suite when reviewing changes.
- **High-value file paths:** `src/HelixTool.Tests/Helix/HelixMcpToolsTests.cs`, `src/HelixTool.Tests/CiKnowledgeServiceTests.cs`, `src/HelixTool.Tests/CacheSecurityTests.cs`, and `src/HelixTool.Tests/Helix/HelixServiceDITests.cs` are the main regression seams for current architecture decisions.

## Learnings

**Archive refresh (2026-03-13):** Detailed PR #15 cleanup, 9-repo `CiKnowledgeServiceTests` expansion, and the linked 2026-03-10 CI-knowledge update moved to `history-archive.md`. Durable takeaways: delete proactive duplicate tests once real coverage lands, and use broad Theory matrices for static CI-profile data.

- Tests for AzdoMcpTools should assert against `[JsonPropertyName]` names (camelCase). No separate MCP result wrappers for AzDO tools.
- **SearchBuildLogAcrossSteps (21 tests):** 3 categories — Unit (T-1–T-11: ranking, early termination, orphans, normalization), Validation (V-1–V-6: argument checks), MCP (M-1–M-2: exception remapping). Key: `SetupTimeline()`/`SetupLogsList()`/`SetupLogContent()` helpers. `LogsSkipped` tracks cap-limited logs, not minLines-filtered. `stoppedEarly` = budget exhausted OR eligible logs remain. Test count after: 812.
- Discoverability copy has two strong regression seams: reflect `DescriptionAttribute` text on MCP tool methods to lock routing promises, and assert rendered CI-guide section ordering with `IndexOf`/section slicing so “use AzDO first” guidance stays visible before pattern inventories.
- `helix_test_results` false-confidence regressions are best caught through MCP-layer exception assertions in `src/HelixTool.Tests/Helix/HelixMcpToolsTests.cs`; high-value cases are no structured-result files, empty uploads, and crash-artifact uploads, all of which should route callers toward `azdo_test_runs`/`azdo_test_results`, `helix_search_log`, and `helix_ci_guide`.
- Key file paths for discoverability coverage: `src/HelixTool.Tests/Helix/HelixMcpToolsTests.cs` now holds MCP description + fallback-routing assertions, `src/HelixTool.Tests/CiKnowledgeServiceTests.cs` locks guide wording/order for aspnetcore/runtime, `src/HelixTool.Mcp.Tools/Helix/HelixMcpTools.cs` contains the live tool descriptions, and `src/HelixTool.Core/CiKnowledgeService.cs` renders the repo-specific CI guide text.
- User preference reinforced again: for review-driven test changes, run focused tests first to debug wording/assertion mismatches quickly, then run the full `src/HelixTool.Tests/HelixTool.Tests.csproj` suite before concluding the regression coverage is complete.

📌 Team updates (2026-03-09 – 2026-03-10 summary): CI profile analysis — 14 tool description/error message recommendations (Ash). Test quality review — net -17 tests, zero coverage loss, prune proactive tests when real tests land (Dallas). CiKnowledgeService expanded to 9 repos, 5 tool descriptions updated (Ripley).

📌 Team update (2026-03-10): Option A folder restructuring executed — 9 Helix files moved to Core/Helix/, Cache namespace added, shared utils extracted from HelixService, Helix/AzDO subfolders in Mcp.Tools and Tests. 59 files, 1038 tests pass, zero behavioral changes. PR #17. — decided by Dallas (analysis), Ripley (execution)

- Cache path-boundary hardening is now regression-covered in `src/HelixTool.Tests/CacheSecurityTests.cs`; the important edge case is a sibling path that differs only by casing (`test-root` vs `TEST-ROOT`), which must be rejected to avoid false containment on case-sensitive filesystems.
- `HelixService` no longer supports a null/implicit `HttpClient`; `src/HelixTool.Tests/Helix/HelixServiceDITests.cs` covers both `ArgumentNullException` branches, and focused/full-suite runs confirmed current CLI/MCP construction sites already inject `IHttpClientFactory` clients.
- Key file paths for this review: `src/HelixTool.Core/Cache/CacheSecurity.cs` contains the Ordinal child-boundary check, `src/HelixTool.Core/Helix/HelixService.cs` owns the strict constructor requirement, `src/HelixTool/Program.cs` and `src/HelixTool.Mcp/Program.cs` wire named `HelixDownload` clients, and `src/HelixTool.Tests/HttpClientConfigurationTests.cs` exercises timeout/cancellation behavior with explicit `HttpClient` injection.
- User preference reinforced: validate review fixes with targeted tests first, then run the full `src/HelixTool.Tests/HelixTool.Tests.csproj` suite before concluding coverage is sufficient.

📌 Team update (2026-03-10): Review-fix decisions merged — README now leads with value prop, shared caching, and context reduction; cache path containment uses exact Ordinal root-boundary checks; and HelixService requires an injected HttpClient with no implicit fallback. Validation confirmed current CLI/MCP DI sites already comply and focused plus full-suite coverage exists. — decided by Kane, Lambert, Ripley

📌 Team update (2026-03-10): Knowledgebase refresh guidance merged — treat the knowledgebase as a living document aligned to current file state, not a static snapshot; earlier README/cache-security/HelixService review findings are resolved knowledge, and only residual follow-up should stay active (discoverability plus documentation/tool-description synchronization). — requested by Larry Ewing, refreshed by Ash

📌 Team update (2026-03-10): Discoverability routing decisions merged — keep the current tool surface, route repo-specific workflow selection through `helix_ci_guide(repo)`, treat `helix_test_results` as structured Helix-hosted parsing rather than a universal first step, and keep `helix_search_log`/docs/help guidance synchronized across surfaces. — decided by Dallas, Kane, Ripley

### Idempotent annotation sweep (2025-07-25)
- **What:** Added `Idempotent = true` to all 22 `[McpServerTool]` attributes that had `ReadOnly = true` across 3 files: `AzdoMcpTools.cs` (12 tools), `HelixMcpTools.cs` (9 tools), `CiKnowledgeTool.cs` (1 tool).
- **Why:** MCP best practices (Anthropic, OpenAI, AWS, arxiv 2602.14878) recommend safety annotations on all tools. `Idempotent = true` signals to clients that these tools are safe to retry and cache, complementing the existing `ReadOnly = true`.
- **Verification:** `helix_download` and `helix_download_url` correctly have `Idempotent = true` WITHOUT `ReadOnly = true` — they write files to disk, so they're idempotent but not read-only. No tools were found missing `ReadOnly = true`.
- **Key files:** `src/HelixTool.Mcp.Tools/AzDO/AzdoMcpTools.cs`, `src/HelixTool.Mcp.Tools/Helix/HelixMcpTools.cs`, `src/HelixTool.Mcp.Tools/CiKnowledgeTool.cs`
- **Test count:** 1047 (1046 pass, 1 pre-existing flaky: `AzdoTokenAccessorTests.ConcurrentCallsWithoutEnvVar`).

📌 Team update (2026-03-13): Scribe merged decision inbox items covering `dotnet` as the VMR profile key, `helix_search`/`helix_parse_uploaded_trx` naming, tighter MCP descriptions, and explicit truncation metadata (`truncated`, `LimitedResults<T>`). README/docs now also call out `ci://profiles` resources and idempotent annotations.
- AzDO auth now centers on `AzdoCredential` instead of raw strings: `Token` is the wire value, `DisplayToken` preserves the original PAT/JWT for assertions and messages, and implicit string conversion returns `DisplayToken`, which keeps older mock patterns readable while still allowing scheme-aware auth tests.
- `AzCliAzdoTokenAccessor` checks `AZDO_TOKEN` on every call but only caches the fallback chain (`AzureCliCredential`/`az` CLI). High-value regression tests should lock both behaviors: env tokens short-circuit without marking fallback state resolved, while a resolved fallback returns the cached credential on later calls.

📌 Team update (2026-03-13): AzDO auth is now the narrow chain `AZDO_TOKEN` → `AzureCliCredential` → az CLI → anonymous, with scheme-aware `AzdoCredential` metadata and `DisplayToken` kept separate from the wire token. — decided by Dallas, Ripley

📌 Team update (2026-03-13): MCP-facing Helix names/descriptions should stay scope-accurate and low-context: use `helix_parse_uploaded_trx`, `helix_search`, and keep repo-specific routing in `helix_ci_guide`. — decided by Ripley
- For private or compile-time auth seams, prefer narrow seam tests: drive `AzdoApiClient` redaction through public 500-response behavior, and use reflection only for `AzdoCredential` API-surface assertions or `TryGetEnvCredential` env-only behavior.
- Edge case: `GetAccessTokenAsync()` does not return null when only `AZDO_TOKEN_TYPE` is set, because the accessor still falls through to `AzureCliCredential`/`az` fallback; the null assertion belongs at the env-resolution seam, not the full accessor chain.
- Edge case: redaction regexes are intentionally selective — `token|key|password|secret=` values, JWT-shaped triples, and 41+ char base64-like blobs redact independently, while ordinary `name=value` text such as `reason=timeout` should remain visible in exception snippets.
- Key file paths: `src/HelixTool.Tests/AzDO/AzdoApiClientRedactionTests.cs` covers error-body redaction via `GetBuildAsync`, and `src/HelixTool.Tests/AzDO/AzdoTokenAccessorTests.cs` now locks invalid override fallback, missing-token env behavior, and `AzdoCredential` operator metadata.

📌 Team update (2026-03-13): PR #28 merged the remaining AzDO auth quick wins — fallback Azure CLI/`az` credentials now refresh on deadline/401, cache isolation keys off stable auth-source identity instead of raw token bytes, and `hlx azdo auth-status` exposes safe auth-path metadata. High-value regression coverage should lock refresh invalidation, cache partitioning, and auth-status output. — decided by Ripley
- Test patterns established: for `AzCliAzdoTokenAccessor`, deterministic fallback-cache tests can be done by seeding the private cached resolution via reflection while forcing a PATH with no `az`; that cleanly distinguishes “fresh cached fallback returned” from “expired fallback re-resolved to anonymous” without depending on developer machine credentials.
- Edge cases discovered: JWT helpers should treat non-JWT text, missing `exp`, and malformed base64url payloads as non-fatal nulls; auth-status on env PATs should surface `environment variable` + PAT expiry warning, while anonymous fallback must still report a user-facing "No AzDO credentials resolved" warning.

📌 Team update (2026-03-13): Cache roots now stay stable via `CacheRootHash` while mutable `AuthTokenHash` partitions AzDO entries, and AzDO auth hashes are seeded before cached AzDO reads. — decided by Ripley

📌 Team update (2026-03-14): helix-cli skill docs must reflect shipped CLI behavior: use `hlx llms-txt` for CLI discovery, note no `hlx ci-guide` command yet, and keep `hlx search-log` CLI docs text-only. — decided by Kane
- Added TDD-style schema coverage in `src/HelixTool.Tests/CliSchema/SchemaGeneratorTests.cs` for the planned `HelixTool.Core.CliSchema.SchemaGenerator` API: primitives, DateTime/DateTimeOffset, Guid, enums, flat/nested POCOs, collections, nullable unwraps, circular references, max-depth cutoff, `AzdoAuthStatus` smoke coverage, and generic-vs-Type overload parity.
- Key pattern for pre-implementation testability: call `SchemaGenerator.GenerateSchema<T>()` / `GenerateSchema(Type)` via reflection, then parse the returned JSON with `JsonDocument` and assert placeholder values and array/object shape. That keeps `dotnet build src/HelixTool.Tests/HelixTool.Tests.csproj --nologo` green before the production type exists while still giving Ripley runtime-red TDD coverage once the implementation lands.

📌 Team update (2026-03-14): `hlx azdo search-log --schema` must mirror the active JSON payload: `LogSearchResult` with `--log-id`, `CrossStepSearchResult` otherwise. — decided by Ripley
- Test patterns for source-generated code: exercise generated registries through the consuming assembly (`HelixTool.Generated.CommandRegistry`) instead of the generator project, and sync metadata tests by reflecting the runtime MCP assembly for `[McpServerTool]` + `[Description]` attributes before comparing routes, descriptions, categories, and parameter defaults.
- Key file paths for describe coverage: `src/HelixTool.Tests/CliSchema/DescribeTests.cs`, `src/HelixTool/Program.cs`, `src/HelixTool.Generators/DescribeGenerator.cs`, and `src/HelixTool.Mcp.Tools/{Helix/HelixMcpTools.cs,AzDO/AzdoMcpTools.cs,CiKnowledgeTool.cs}`.
- `src/HelixTool.Tests/CliSchema/SchemaGeneratorTests.cs` should call `SchemaGenerator.GenerateSchema<T>()` / `GenerateSchema(Type)` directly now that the API is public; schema assertions must follow JSON contract names, so `[JsonPropertyName]` coverage belongs in both a local DTO test and a real-model smoke test such as `AzdoBuild`.

📌 Team update (2026-03-16): MCP timeline truncation implementation complete — `TimelineResponse` record type introduced for `azdo_timeline` return value when output exceeds 200 records (first 100 returned + truncation metadata). May need test updates if assuming `AzdoTimeline?` return type. See `.squad/decisions/decisions.md` (section "Timeline Truncation Implementation") for details and design trade-offs. — implemented by Ripley
📌 Team update (2026-03-16): MCP timeline truncation implementation complete — `TimelineResponse` record type introduced for `azdo_timeline` return value when output exceeds 200 records (first 100 returned + truncation metadata). May need test updates if assuming `AzdoTimeline?` return type. See `.squad/decisions/decisions.md#timeline-truncation` for details and design trade-offs. — implemented by Ripley

📌 Team update (2026-05-20): Pagination standardization audit complete — Dallas audit found 2 🔴 tools (`azdo_changes`, `azdo_test_runs`) returning raw lists with no truncation metadata, 10 🟡 tools with bespoke response shapes. Phase 1 wraps reds in `CreateLimitedResults()` (~30 min). Phase 2 adds `truncated` field to yellows (~2-3 hours). Testing target: verify truncated flag set correctly when results exceed `top` parameter. See `.squad/decisions.md#dallas--pagination-architecture` for inventory table and upstream API reality.

### Pagination contract tests (2026-05-20)
- **What:** Added 13 contract tests (333 LOC) in `src/HelixTool.Tests/AzDO/PaginationContractTests.cs` verifying pagination standardization per Dallas's spec.
- **Coverage areas:**
  1. **CreateLimitedResults helper** (5 tests): truncation logic when count == top, count < top, top=0, empty results, count > top edge case
  2. **Phase 1 tools** (4 tests): `azdo_changes` and `azdo_test_runs` now return `LimitedResults<T>` with correct truncation flag
  3. **Default parameters** (4 tests): `azdo_builds`, `azdo_test_results`, `azdo_changes`, `azdo_test_runs` use reasonable defaults (20/50/20/50)
- **Branch coordination:** Ripley is implementing Phase 1 changes in parallel (modified `AzdoMcpTools.cs`, `HelixMcpTools.cs`, `AzdoModels.cs`). Ripley's work is WIP (incomplete Helix changes causing build errors), but Phase 1 changes to `azdo_changes` and `azdo_test_runs` match the test expectations exactly.
- **Test status:** Tests compile against `HelixTool.Core`. Full solution build blocked by Ripley's incomplete Helix work — expected and normal for parallel development. Tests will verify Ripley's implementation once complete.
- **Key file:** `src/HelixTool.Tests/AzDO/PaginationContractTests.cs`
- **Test count:** 1167 existing + 13 new = 1180 tests (when Ripley completes implementation).


📌 Team update (2026-05-08): MCP SDK v1.0.0 → v1.3.0 upgrade pending — parallel research (Ash) and inventory (Dallas) complete; recommendation to upgrade to v1.3.0 (low risk, no code changes required). Drift items flagged by Dallas: hardcoded ServerInfo.Version, stdio host missing WithResourcesFromAssembly, no Directory.Packages.props. May need test updates if SDK changes affect existing test coverage. See `.squad/decisions/inbox/*` for details.

📌 Team update (2026-05-08): MCP SDK 1.3.0 upgrade — Ripley shipped branch `squad/mcp-sdk-1.3.0-upgrade` with MCP SDK 1.0.0 → 1.3.0, Central Package Management adoption (Directory.Packages.props), stdio host resource visibility fix, and ServerInfo.Version de-hardcoding. Validation: dotnet restore ✅, dotnet build ✅ (0 errors, 6 NU1507 warnings pre-existing). **Test verification pending** — Lambert owns full suite validation and sign-off before merge.


### MCP SDK 1.0.0 → 1.3.0 upgrade verification (2026-05-08)
- **Branch:** `squad/mcp-sdk-1.3.0-upgrade`, commit 80bf9f2 by Ripley.
- **Test command:** `dotnet test` from repo root → **1167 passed, 0 failed, 0 skipped, ~3 s** on net10.0. No regressions; no test changes needed for the SDK bump itself.
- **Stdio smoke:** `dotnet run --project src/HelixTool -- mcp --help` exits cleanly; `dotnet run --project src/HelixTool -- --version` returns **0.5.4**, proving `AssemblyInformationalVersionAttribute` lookup replaces the old hardcoded "0.1.2".
- **HTTP smoke:** `dotnet src/HelixTool.Mcp/bin/Debug/net10.0/HelixTool.Mcp.dll --urls http://127.0.0.1:18765` boots, binds, logs `Application started.` with zero DI resolve errors. Killed cleanly.
- **Resource scan parity:** Both `src/HelixTool/Program.cs:944` and `src/HelixTool.Mcp/Program.cs:89` now call `.WithResourcesFromAssembly(typeof(HelixMcpTools).Assembly)` — stdio fix matches HTTP host so `ci://profiles` is reachable from CLI/stdio clients.
- **Version source:** both Program.cs files use `Assembly.GetExecutingAssembly().GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion ?? "0.0.0"` (stdio L939, http L84).
- **Gotcha:** Pre-existing `NU1507` warning (nuget.org + dotnet-eng without source mapping) is now surfaced under CPM. Out of scope but worth a follow-up. Verdict file: `.squad/decisions/inbox/lambert-mcp-sdk-upgrade-verdict.md`.
- **Verdict:** ✅ APPROVE.

📌 Team update (2026-05-08): MCP SDK 1.3.0 follow-ups — verification pending

**Context:** Ripley completed two PRs:
1. **PR #47** — `[AllowedValues]` sweep, `OpenWorld` annotations, `<packageSourceMapping>` (1167 tests pass ✅)
2. **PR #48** — Progress notifications on long-running tools (1167 tests pass ✅)

**Verification status:**
- Both PRs passed local test suite (1167/1167)
- Smoke tests confirmed (build clean, MCP help, version resolution)
- Awaiting sequential verification per squad orchestration

**Follow-ups from Ripley:**
- **PR #48:** Add unit tests for `ProgressReporter.CopyToWithProgressAsync` (event count band, monotonic progress, null sink no-op) + one end-to-end MCP-stdio test using SDK client's `WithProgress`
- **General:** Verify `[AllowedValues]` options match actual enum values in MCP schema generation

**Test count:** 1167 baseline maintained across both PRs.

### PR #47 (annotations + NU1507) and PR #48 (MCP progress notifications) verification (2026-05-08)
- **PR #47 verdict:** ✅ APPROVE. Build clean (0 warnings — NU1507 gone), 1167/1167 tests pass. Reflection probe over `HelixTool.Mcp.Tools.dll` confirms 25 `[McpServerTool]` methods with `OpenWorld` set explicitly on every one (22 true / 3 false: `azdo_auth_status`, `helix_auth_status`, `helix_ci_guide`). 10 params carry `[AllowedValues]`. Generated descriptors via `McpServerTool.Create()` show `enum` arrays in JSON schema (e.g. `azdo_builds.status.enum = ["all","cancelling","completed","inProgress","none","notStarted","postponed"]`) and `annotations.openWorldHint` in tool annotations.
- **PR #48 verdict:** ✅ APPROVE. Build clean, 1167/1167 baseline tests pass. Wrote a temporary `src/HelixTool.Tests/ProgressNotificationsSmokeTests.cs` (7 tests) to verify (1) `Download`, `FindFiles`, `SearchLog` declare a default-null `IProgress<ProgressNotificationValue>?` parameter with no `[Description]`, (2) `McpProgressAdapter.Wrap(null) == null` (fast path), (3) `Wrap(sink)` forwards `ProgressUpdate → ProgressNotificationValue` with float coercion, (4) `ProgressReporter.CopyToWithProgressAsync` emits initial+final updates over a 1 MiB stream. With smokes: 1174/1174 pass. Smokes were removed before returning to main (Ripley owns the branch); body preserved in the verdict file for re-use.
- **Pattern:** `McpProgressAdapter` is `internal` and `HelixTool.Mcp.Tools` has no `InternalsVisibleTo("HelixTool.Tests")`, so any future test of internal seams there must use reflection or land an IVT first. Recommended follow-up: ask Dallas/Ripley to add `[assembly: InternalsVisibleTo("HelixTool.Tests")]` to `HelixTool.Mcp.Tools`.
- **Probe pattern reused:** building a tiny `Probe` console csproj that `ProjectReference`s `HelixTool.Mcp.Tools` is the cleanest way to inspect MCP tool descriptors out-of-band — `McpServerTool.Create(method, target)` requires a non-null `target` for instance methods (use `RuntimeHelpers.GetUninitializedObject(method.DeclaringType)` when you don't want to construct the real DI graph).
- **Gotcha:** `HelixMcpTools` and `AzdoMcpTools` live in the flat `HelixTool.Mcp.Tools` namespace despite being under `Helix/` and `AzDO/` folders — don't add `using HelixTool.Mcp.Tools.Helix;`-style usings, they will not compile.
- **Branch sequencing observation:** during this verification, an external process (likely Scribe) checked out main and committed in the same working tree while Lambert was mid-verify. Sequential single-tree verification is workable but fragile — re-confirm the active branch with `git rev-parse --abbrev-ref HEAD` between stages, or move to git worktrees per the standing recommendation in the SDK 1.3.0 verdict.

