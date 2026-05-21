# Dallas — History Archive
> Archived on 2026-03-09. See history.md for current context.

## 2026-03-07: Decision — AzdoMcpTools returns model types directly
If we later need to reshape AzDO output differently from the API models, we'd add wrapper types then. For now, direct return is simpler and correct.

## 2026-03-08: AzDO Security Review — Learnings
- **Query parameter injection is easy to miss.** `prNumber` was not escaped or validated as an integer while `branch` and `statusFilter` were properly escaped with `Uri.EscapeDataString`. Enforce convention: all user-provided string values interpolated into URLs must be either type-validated (e.g., `int.TryParse` for numeric IDs) or `Uri.EscapeDataString`-escaped. No exceptions.
- **`BuildUrl` hardcoding `https://dev.azure.com/` is the right SSRF mitigation.** Combined with `Uri.EscapeDataString` on org/project, this makes SSRF structurally impossible regardless of input. This pattern should be preserved — never allow user input to influence the URL base/authority.
- **Singleton `AzCliAzdoTokenAccessor` with `_resolved` flag doesn't handle token expiry.** For long-running MCP servers, az CLI tokens (~1h lifetime) will expire. Not a security bug (fails closed with 401), but an operational gap. Future work: track JWT expiry or use `AZDO_TOKEN` with external rotation.
- **`CacheSecurity.SanitizeCacheKeySegment` is the established pattern for cache key hygiene.** AzDO caching correctly reuses it. Any new cacheable subsystem must use it too.
- **Security review convention established:** For new API integrations, review all 7 focus areas (command injection, SSRF, token leakage, input validation, cache isolation, HTTP security, pattern consistency). Use SEC-{N} IDs with severity levels.

## 2026-03-13: Archived from history.md during summarization

### 2026-03-09: azdo_search_log_across_steps design spec

**Spec:** `.ai-team/decisions/inbox/dallas-azdo-search-across-steps.md`

Complements `azdo_search_log`. Two-phase: metadata → incremental search with early termination. Ranking by failure likelihood (4 buckets). New `GetBuildLogsListAsync` on `IAzdoApiClient`. `NormalizeAndSplit` extracted. Result types in Core. Safety: maxLogs=30, minLines=5, maxMatches=50. ~19 tests.

Key files: IAzdoApiClient.cs, AzdoApiClient.cs, AzdoModels.cs, AzdoService.cs, AzdoMcpTools.cs, Program.cs.

### 2026-03-09: Append-on-expire caching (D-6)

**Spec:** `.ai-team/decisions/inbox/dallas-incremental-log-fetching.md`

Freshness marker pattern: content key (4h) + sentinel (15s). Delta-append via CountLines. Uses `IsBuildCompletedAsync` (Option B). Range requests on stale caches delta-refresh first. 12 new tests (C-10–C-21).

### 2026-03-09: Code review — Incremental log fetching (Phase 1 + Phase 2)

**APPROVED** with P0 follow-up. All spec items D-1–D-6 verified correct. 32 tests passing (A-1..A-5, C-1..C-21, S-1..S-6).

**P0 — CountLines off-by-one:** `Split('\n').Length` overcounts by 1 with trailing `\n`. Fix: subtract 1 when content ends with `\n`. Ripley: fix. Lambert: update C-18, C-19, delta tests.

📌 Team updates (2026-03-09): Incremental log (PR #13), perf review, cache raw: prefix. — Ripley

### 2025-07-24: Test Quality Review — Tautological Test Audit

Reviewed 776 tests. ~40 problematic (5%), concentrated not systemic. Deleted ~17 tests (~350 lines), zero coverage loss. Key rules: no layer duplication, ≤1 passthrough smoke test, interface compliance tests are redundant. Gold standard patterns: AzdoSecurityTests, AzdoIdResolverTests, TextSearchHelperTests, CachingAzdoApiClientTests, AzdoServiceTailTests.

📌 Team update (2026-03-10): CiKnowledgeService expanded to 9 repos with full profiles. 5 tool descriptions updated. — Ripley

- **HelixTool.Core has asymmetric organization.** AzDO code lives in a clean `AzDO/` subfolder with `HelixTool.Core.AzDO` namespace. Helix-specific code (8 files, ~1,700 lines) is scattered at the project root alongside shared utilities (5 files, ~800 lines). CachingHelixApiClient is in `Cache/` but is Helix-specific.
- **Cache/ folder uses `HelixTool.Core` namespace, not `HelixTool.Core.Cache`.** All 7 cache files lack a sub-namespace, unlike AzDO which correctly uses `HelixTool.Core.AzDO`. This makes cache types indistinguishable from Helix types by namespace alone.
- **AzdoService depends on HelixService for shared utility methods.** `AzdoService.cs` calls `HelixService.MatchesPattern()` and `HelixService.IsFileSearchDisabled` — these are genuinely shared utilities stranded on a domain-specific class. Must extract before any structural reorganization.
- **HelixTool.Mcp.Tools has flat structure mixing domains.** HelixMcpTools.cs (483 lines) and AzdoMcpTools.cs (307 lines) sit side-by-side with no folder separation.
- **Program.cs (CLI) is 1,513 lines** — largest file in the repo, contains all Helix + AzDO commands in one file.
- **Option A (folder-level reorg) recommended over project splitting at current scale (~22K lines, ~80 files, ~770 tests, 1 team).** Create `Helix/` subfolders mirroring existing `AzDO/` subfolders. Decision spec: `.ai-team/decisions/inbox/dallas-helix-azdo-restructure.md`

## 2026-03-13: Archived durable learnings from history.md

- **Helix auth is opaque tokens only.** The Helix API uses `Authorization: token <TOKEN>` with server-generated opaque strings. No Entra/JWT/OAuth possible until the Helix service team adds support server-side. This is a hard constraint — don't revisit.
- **AzDO auth uses Azure Identity (Entra ID).** The scope is `499b84ac-1321-427f-aa17-267ca6975798/.default`. `AzureDeveloperCliCredential` (via `azd auth login`) is the primary credential for CLI tools. `Azure.Identity` handles token caching/refresh internally.
- **AzDO REST API is stable at v7.0.** The ci-analysis script uses `api-version=7.0` for all endpoints. The 7 endpoints we need (build, builds, timeline, log, changes, test runs, test results) are well-documented and unlikely to change.
- **Microsoft.TeamFoundationServer.Client SDK is too heavy for our use case.** It pulls 40+ transitive deps including Newtonsoft.Json and a CVE-affected System.Data.SqlClient. HttpClient + System.Text.Json is sufficient for 7 REST endpoints.
- **AzDO builds span two orgs: dnceng-public (PR builds) and dnceng (internal).** The API client must accept org/project as per-call parameters, not constructor-level config.
- **Timeline for in-progress builds must never be cached.** The timeline changes as jobs complete — the ci-analysis script explicitly skips cache writes for in-progress builds.
- **`git credential` is the right storage abstraction for CLI tools targeting .NET developers.** Zero new deps, cross-platform, delegates keychain management to the user's existing git credential helper. Same pattern `darc` uses.
- **IHelixTokenAccessor.GetAccessToken() is synchronous.** Changing to async would be a cross-cutting change affecting all 3 projects. For Phase 1, sync-over-async (.GetAwaiter().GetResult()) on the git credential call is acceptable — it runs once at startup and completes in <100ms.
- **Token resolution precedence: env var > stored credential.** Env var must win for backward compat and CI/CD override semantics. Never prompt during DI container setup.
- **HelixService.cs has 7 identical error message strings** for 401 handling. These should be extracted to a constant when updating the message text.
- **AzDO build logs are append-only.** Once a line is written to a build log, it never changes. This is a structural property of the AzDO logging pipeline, not just an observed behavior. Cached log content is always a valid prefix of the current log — only new lines get added at the end.
- **`ICacheStore` deletes expired entries — no stale reads.** `GetMetadataAsync` returns `null` for expired keys, not stale data. Any "keep-but-refresh" pattern must use long TTLs + separate freshness markers, not short TTLs on the content itself.
- **Freshness marker pattern for delta caching.** Two cache keys: content (long TTL) + freshness sentinel (short TTL). When sentinel expires, delta-fetch new data, append to content, reset sentinel. Avoids extending `ICacheStore` for stale-while-revalidate semantics. Applicable to any append-only data source.
- **`CountLines` must account for trailing newlines in AzDO log content.** `string.Split('\n').Length` overcounts by 1 when content ends with `\n` (common for AzDO logs). The correct count is the number of `\n` characters (for newline-terminated content) or `Split` count minus 1 when trailing `\n` exists. This matters for delta-fetch `startLine` computation — an off-by-one causes a missed boundary line on every delta cycle. P0 fix required.
- **Existing tests survive interface changes via `Arg.Any<T>()` for new optional params.** When adding optional parameters to an interface method (like `int? startLine = null`), all existing mock setups need `Arg.Any<int?>()` for the new params. NSubstitute won't match if the arg matchers don't cover the full signature.
# Dallas — History

## Project Learnings (from import)
- **Project:** hlx — Helix Test Infrastructure CLI & MCP Server
- **User:** Larry Ewing
- **Stack:** C# .NET 10, ConsoleAppFramework, Spectre.Console, ModelContextProtocol, Microsoft.DotNet.Helix.Client
- **Structure:** Three projects — HelixTool.Core (shared library), HelixTool (CLI), HelixTool.Mcp (HTTP MCP server)
- **Key files:** HelixService.cs (core ops), HelixIdResolver.cs (GUID/URL parsing), HelixMcpTools.cs (MCP tool definitions), Program.cs (CLI commands)

## Core Context

- **Architecture boundaries:** `src/HelixTool.Core/Helix/`, `src/HelixTool.Core/AzDO/`, `src/HelixTool.Core/Cache/`, and `src/HelixTool.Mcp.Tools/{Helix,AzDO}/` are the stable domain folders; composition roots remain `src/HelixTool/Program.cs` and `src/HelixTool.Mcp/Program.cs`.
- **Design defaults:** keep business logic in services, keep MCP tools thin, use decorators for caching, and prefer behavioral contracts in tool descriptions over implementation details.
- **Caching/auth:** running console logs are never cached, cache isolation is by auth-token hash, stdio remains the primary transport, and HTTP/SSE auth is a scoped per-request design rather than a CLI concern.
- **Protocol/runtime rules:** Helix auth remains opaque `Authorization: token` only, AzDO calls must continue accepting org/project per request, and in-progress AzDO timelines/logs should be treated as live append-only data rather than long-lived cache snapshots.
- **Cache/testing patterns:** append-only log refresh uses long-lived content plus short-lived freshness sentinels, `ICacheStore` never serves expired entries, and interface signature changes require updated `Arg.Any<T>()` coverage for every optional parameter in existing NSubstitute setups.
- **AzDO direction:** model types can be returned directly from MCP tools, `helix_ci_guide` owns repo-specific routing, and the current AzDO auth chain is `AZDO_TOKEN` → `AzureCliCredential` → az CLI → anonymous.

## Learnings

**Archive refresh (2026-03-13):** Detailed `azdo_search_log_across_steps`, append-on-expire caching, tautological-test-audit, and Helix/AzDO restructuring-analysis notes moved to `history-archive.md`. Keep the durable rules: ranked incremental log search, freshness-sentinel append caching, pruning layer-duplicate/passthrough tests, and preferring the low-risk folder-level split over premature project sprawl.


📌 Team updates (2026-03-07 – 2026-03-08 summary): Test result file discovery consolidated (Ripley). CacheStoreFactory Lazy<T> pattern (Ripley). AzDO edge cases documented (Lambert). Dynamic TTL caching strategy (Ripley). Context-limiting defaults for AzDO MCP tools (Ripley). AzDO artifact/attachment test patterns — 700 total tests (Lambert). AzDO docs subsections (Kane). IsFileSearchDisabled promoted to public (Ripley). AzDO search gap analysis — P0 azdo_search_log (Ash).

### 2026-03-09: azdo_search_log_across_steps design spec

**Spec:** `.ai-team/decisions/inbox/dallas-azdo-search-across-steps.md`

Complements `azdo_search_log`. Two-phase: metadata → incremental search with early termination. Ranking by failure likelihood (4 buckets). New `GetBuildLogsListAsync` on `IAzdoApiClient`. `NormalizeAndSplit` extracted. Result types in Core. Safety: maxLogs=30, minLines=5, maxMatches=50. ~19 tests.

Key files: IAzdoApiClient.cs, AzdoApiClient.cs, AzdoModels.cs, AzdoService.cs, AzdoMcpTools.cs, Program.cs.

### 2026-03-09: Append-on-expire caching (D-6)

**Spec:** `.ai-team/decisions/inbox/dallas-incremental-log-fetching.md`

Freshness marker pattern: content key (4h) + sentinel (15s). Delta-append via CountLines. Uses `IsBuildCompletedAsync` (Option B). Range requests on stale caches delta-refresh first. 12 new tests (C-10–C-21).

### 2026-03-09: Code review — Incremental log fetching (Phase 1 + Phase 2)

**APPROVED** with P0 follow-up. All spec items D-1–D-6 verified correct. 32 tests passing (A-1..A-5, C-1..C-21, S-1..S-6).

**P0 — CountLines off-by-one:** `Split('\n').Length` overcounts by 1 with trailing `\n`. Fix: subtract 1 when content ends with `\n`. Ripley: fix. Lambert: update C-18, C-19, delta tests.

📌 Team updates (2026-03-09): Incremental log (PR #13), perf review, cache raw: prefix. — Ripley

📌 Team update (2026-03-10): CiKnowledgeService expanded to 9 repos with full profiles. 5 tool descriptions updated. — Ripley

- **HelixTool.Core has asymmetric organization.** AzDO code lives in a clean `AzDO/` subfolder with `HelixTool.Core.AzDO` namespace. Helix-specific code (8 files, ~1,700 lines) is scattered at the project root alongside shared utilities (5 files, ~800 lines). CachingHelixApiClient is in `Cache/` but is Helix-specific.
- **Cache/ folder uses `HelixTool.Core` namespace, not `HelixTool.Core.Cache`.** All 7 cache files lack a sub-namespace, unlike AzDO which correctly uses `HelixTool.Core.AzDO`. This makes cache types indistinguishable from Helix types by namespace alone.
- **AzdoService depends on HelixService for shared utility methods.** `AzdoService.cs` calls `HelixService.MatchesPattern()` and `HelixService.IsFileSearchDisabled` — these are genuinely shared utilities stranded on a domain-specific class. Must extract before any structural reorganization.
- **HelixTool.Mcp.Tools has flat structure mixing domains.** HelixMcpTools.cs (483 lines) and AzdoMcpTools.cs (307 lines) sit side-by-side with no folder separation.
- **Program.cs (CLI) is 1,513 lines** — largest file in the repo, contains all Helix + AzDO commands in one file.
- **Option A (folder-level reorg) recommended over project splitting at current scale (~22K lines, ~80 files, ~770 tests, 1 team).** Create `Helix/` subfolders mirroring existing `AzDO/` subfolders. Decision spec: `.ai-team/decisions/inbox/dallas-helix-azdo-restructure.md`

📌 Team update (2026-03-10): Option A folder restructuring executed — 9 Helix files moved to Core/Helix/, Cache namespace added, shared utils extracted from HelixService, Helix/AzDO subfolders in Mcp.Tools and Tests. 59 files, 1038 tests pass, zero behavioral changes. PR #17. — decided by Dallas (analysis), Ripley (execution)

📌 Team update (2026-03-10): Review-fix decisions merged — README now leads with value prop, shared caching, and context reduction; cache path containment uses exact Ordinal root-boundary checks; and HelixService requires an injected HttpClient with no implicit fallback. Validation confirmed current CLI/MCP DI sites already comply and focused plus full-suite coverage exists. — decided by Kane, Lambert, Ripley

📌 Team update (2026-03-10): Knowledgebase refresh guidance merged — treat the knowledgebase as a living document aligned to current file state, not a static snapshot; earlier README/cache-security/HelixService review findings are resolved knowledge, and only residual follow-up should stay active (discoverability plus documentation/tool-description synchronization). — requested by Larry Ewing, refreshed by Ash

📌 Team update (2026-03-10): Discoverability routing decisions merged — keep the current tool surface, route repo-specific workflow selection through `helix_ci_guide(repo)`, treat `helix_test_results` as structured Helix-hosted parsing rather than a universal first step, and keep `helix_search_log`/docs/help guidance synchronized across surfaces. — decided by Dallas, Kane, Ripley

📌 Team update (2026-03-13): Scribe merged decision inbox items covering `dotnet` as the VMR profile key, `helix_search`/`helix_parse_uploaded_trx` naming, tighter MCP descriptions, and explicit truncation metadata (`truncated`, `LimitedResults<T>`). README/docs now also call out `ci://profiles` resources and idempotent annotations.
- **AzDO auth code and auth decisions are currently out of sync.** `decisions.md` and Dallas history record Azure Identity / `AzureDeveloperCliCredential` as the intended direction for AzDO, but the live implementation in `src/HelixTool.Core/AzDO/IAzdoTokenAccessor.cs` still explicitly avoids `Azure.Identity` and uses `AZDO_TOKEN` → `az account get-access-token` → anonymous because of WSL libsecret/D-Bus failures. Any future auth work must treat this as an intentional divergence to resolve, not a missing cleanup.
- **`HelixTool.Core` already carries Azure SDK plumbing transitively.** `Microsoft.DotNet.Helix.Client` already brings in `Azure.Core`, so adding `Azure.Identity` would not introduce the first Azure package; the incremental cost is the identity stack (Azure.Identity + MSAL packages and a few abstractions), not the Azure SDK foundation.
- **If AzDO adopts Azure.Identity, use an explicit narrow chain, not `DefaultAzureCredential`.** The recommended order is `AZDO_TOKEN` env var → targeted Azure.Identity credential(s) for cached developer auth → existing `az` CLI subprocess fallback → anonymous. This avoids `DefaultAzureCredential`'s broad probe surface and preserves the known-good WSL fallback path.
- **AzDO auth touchpoints are concentrated in four files.** `src/HelixTool.Core/AzDO/IAzdoTokenAccessor.cs` contains both the interface and current az CLI implementation, `src/HelixTool.Core/AzDO/AzdoApiClient.cs` conditionally adds the Bearer header, and both `src/HelixTool/Program.cs` and `src/HelixTool.Mcp/Program.cs` register the accessor as a singleton in the composition roots.

📌 Team update (2026-03-13): AzDO auth is now the narrow chain `AZDO_TOKEN` → `AzureCliCredential` → az CLI → anonymous, with scheme-aware `AzdoCredential` metadata and `DisplayToken` kept separate from the wire token. — decided by Dallas, Ripley

📌 Team update (2026-03-13): MCP-facing Helix names/descriptions should stay scope-accurate and low-context: use `helix_parse_uploaded_trx`, `helix_search`, and keep repo-specific routing in `helix_ci_guide`. — decided by Ripley

- **AzDO auth is server-scoped in HTTP MCP mode.** `IAzdoTokenAccessor` is registered as a singleton in `HelixTool.Mcp/Program.cs`, so remote MCP clients act through the server's shared AzDO identity even when Helix auth/cache isolation is per-request. Future security reviews should treat shared HTTP deployments as a privilege-concentration boundary.
- **Caching raw Azure tokens blocks refresh and extends memory residency.** The current accessor caches `AzdoCredential` strings, not credential-provider state, so `AzureCliCredential`/`az` bearer tokens persist for process lifetime and cannot refresh after expiry without explicit invalidation. Long-running MCP servers need expiry-aware refresh or strict guidance to use externally rotated `AZDO_TOKEN`.
- **AzDO cache isolation is weaker than the README currently implies.** `CachingAzdoApiClient` keys entries by org/project/suffix, while CLI/stdio composition roots initialize `CacheOptions.AuthTokenHash = null`; authenticated AzDO responses can therefore persist in the shared `public/` cache even though the auth chain itself never writes tokens to disk. Any future cache/auth work should key AzDO cache state by AzDO auth context or narrow the documentation claim.
- **`DisplayToken` remains the main token-leak footgun.** `ToString()` is redacted and current error paths are careful, but the implicit `AzdoCredential -> string` conversion still yields the raw display token. Preserve the current guarded call-site pattern and prefer removing or heavily warning this conversion if compatibility allows.

📌 Team update (2026-03-13): PR #28 merged the remaining AzDO auth quick wins — fallback Azure CLI/`az` credentials now refresh on deadline/401, cache isolation keys off stable auth-source identity instead of raw token bytes, and `hlx azdo auth-status` exposes safe auth-path metadata. — decided by Ripley

📌 Team update (2026-03-13): Cache roots now stay stable via `CacheRootHash` while mutable `AuthTokenHash` partitions AzDO entries, and AzDO auth hashes are seeded before cached AzDO reads. — decided by Ripley

📌 Team update (2026-03-14): helix-cli skill docs must reflect shipped CLI behavior: use `hlx llms-txt` for CLI discovery, note no `hlx ci-guide` command yet, and keep `hlx search-log` CLI docs text-only. — decided by Kane

📌 Team update (2026-03-16): MCP timeline truncation + CI guide improvements complete — `azdo_timeline` now implements partial response pattern (200-record threshold, 100 returned + truncation metadata); 5 core repo profiles in `CiKnowledgeService` updated to recommend `azdo_search_timeline` as first investigation step for large builds; cross-references added in tool descriptions. Wire format change from `AzdoTimeline?` to `TimelineResponse?` — review for CLI-side compatibility and threshold configurability. Build clean, 1127 tests pass. — implemented by Ripley

## 2026-XX-XX: MCP SDK usage inventory (for Larry, parallel with Ash's SDK research)
- Both `ModelContextProtocol` and `ModelContextProtocol.AspNetCore` are pinned to **1.0.0** (no Directory.Packages.props; versions live in csproj).
- Two transports in use: `WithStdioServerTransport()` in `HelixTool/Program.cs` (CLI `mcp` subcommand) and `WithHttpTransport()` + `MapMcp()` in `HelixTool.Mcp/Program.cs` (ASP.NET Core HTTP server).
- Tool registration is fully attribute-based (`[McpServerToolType]` + `[McpServerTool]`) discovered via `WithToolsFromAssembly(typeof(HelixMcpTools).Assembly)`. ~28 tools across 3 classes (Helix=11, AzDO=15, CiKnowledgeTool=1, +1 resource type with 2 resources).
- Resources: only registered in HTTP server (`WithResourcesFromAssembly`), not in stdio CLI server — divergence worth flagging if SDK upgrade changes resource semantics.
- No prompts. No source generators against MCP SDK (DescribeGenerator inspects `[McpServerTool]` syntax for CLI/MCP cross-reference doc but doesn't depend on SDK runtime types).
- SDK extension surface is shallow: `McpException` for tool-surface errors, `UseStructuredContent = true` on most tools for auto-generated output schemas, `[Description]` for params, `IHttpContextAccessor`-based per-request token plumbing (HTTP server), custom `ApiKeyMiddleware` runs before `MapMcp()`.
- Version-pinning rationale (decisions.md): chose 1.0.0 specifically for `UseStructuredContent` schema generation and `McpException` surfacing semantics. No "do not upgrade" guidance recorded.
- `.mcp/server.json` (packaged with the dotnet tool) ships separately and is version-validated against csproj/tag at release.

📌 Team update (2026-05-08): MCP SDK usage inventory for v1.0.0 → v1.3.0 upgrade assessment — ModelContextProtocol 1.0.0 + ModelContextProtocol.AspNetCore 1.0.0 across 3 csprojs; attribute-based tool registration (~27 tools); identified drift: hardcoded ServerInfo.Version, stdio host missing WithResourcesFromAssembly, no Directory.Packages.props; parallel Ash research recommends v1.3.0 upgrade (low risk, no code changes required).

📌 Team update (2026-05-08): MCP SDK 1.3.0 upgrade — Ripley shipped Central Package Management migration across 6 csprojs (new Directory.Packages.props), MCP SDK 1.0.0 → 1.3.0 (zero source changes), stdio host resource visibility fix (WithResourcesFromAssembly), ServerInfo.Version de-hardcoding (AssemblyInformationalVersionAttribute pattern). Build validates (0 errors, 6 NU1507 warnings pre-existing). Branch `squad/mcp-sdk-1.3.0-upgrade` ready for Lambert testing.


📌 Team update (2026-05-08): Parallel squad work — worktree recommendation

**Context:** Two Ripley branches (`squad/mcp-tool-annotations-and-cleanup` and `squad/mcp-progress-notifications`) ran in parallel in the same working tree, causing a checkout race mid-task.

**Recommendation:** Future parallel squad work should use separate `git worktree`s. Each agent gets isolated working state, eliminating checkout races entirely. Worktrees are lightweight (shared git dir, isolated working dirs). Recovery: `git worktree add /path/to/worktree-N branch-name` before spawning agents; cleanup with `git worktree remove` after.

**Owner:** Dallas (CI/coordination) — recommend baking into squad orchestration checklist for future parallel sprints.

📌 Team update (2026-05-21): Pagination Phase 1+2 implemented and tested — wrapped `azdo_changes` and `azdo_test_runs` in `LimitedResults<T>`; added `truncated`/`note` fields to 8 bespoke result types; 13 contract tests (333 LOC) verify truncation behavior. Full suite: 1180/1180 passing. Closes Dallas's pagination audit spec. — implemented by Ripley, tested by Lambert
