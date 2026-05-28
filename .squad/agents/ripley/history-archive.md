# Ripley History Archive

Archived on 2026-05-28 (history.md exceeded 15,360 bytes; archiving entries before 2026-05-21 for size management).

## Recent Activity Summary (2026-05-22)

### v0.7.3 Release (2026-05-22)
- Bumped 3 version stamps (HelixTool.csproj + server.json)
- Build: 0 errors, 0 warnings (10.58s)
- Tests: 1292/1292 passing
- Commit: 73e65fd (`release: v0.7.3`)
- Tag: `v0.7.3` pushed; publish workflow completed in 41s
- Asset: `lewing.helix.mcp.0.7.3.nupkg` on nuget.org

### DTO Consolidation PR #58 (2026-05-22)
- Implemented result DTO consolidation per Dallas triage verdict
- Consolidated 6 duplicate classes from Program.cs → McpToolResults.cs
- +93/-87 LOC (net -4 LOC); Commit 311a571
- [JsonPropertyName] attributes standardized
- JSON wire-format verified (status, files, work-item outputs)
- 1292/1292 tests passing; no breaking changes
- **Status:** Merged to main

## Pattern Learnings (Archive: See history-archive.md)

**Release Flow (confirmed 3× consecutive releases):**
- Version stamps: HelixTool.csproj (v), server.json (top-level + packages array)
- Build gate: 0 errors, tests 100% passing
- Commit → Push main → Tag → Push tag (triggers publish workflow)
- Publish workflow auto-creates GitHub Release + NuGet push
- Never manual `gh release create` (causes 422, skips NuGet)

**Release Recipe Metrics:**
- v0.7.1: 33s workflow time (consistent)
- v0.7.2: 38s workflow time (consistent)
- v0.7.3: 41s workflow time (consistent)
- **Three consecutive releases with zero deviations**

**Dependency Management (CPM):**
- Central version control: `Directory.Packages.props` only
- No `.csproj` changes needed for version bumps
- Azure.Identity 1.13.2 → 1.21.0 (deprecated, type-forwarded, non-breaking)
- Microsoft.Data.Sqlite 9.0.7 → 10.0.8 (major cross safe for net10.0)

**Code Changes (2026-05-21):**
- PR #54: CPM bump (6 packages) v0.7.1
- PR #55: WorkItemSummary exit-code field surfacing (v0.7.2)
- PR #56: AzDO timeline filter presets (97 unit tests, main)
- PR #57: MCP description tightening (8 tools, 229→93 words, main)
- PR #58: DTO consolidation (6 classes, main)

**MCP Tool Patterns:**
- Exception handling: `catch (Exception ex) when (...) { throw new McpException(...); }`
- Description rubric: Verb-led, ~20 words, defaults in parameters
- `azdo_auth_status` is NOT sync-safe (can await on cache miss)

**AzDO Timeline Filters:**
- Aliases: `inProgress`, `in-progress`, `active`, `notStarted`, `not-started`
- `azdo_helix_jobs` gate relaxed for `running`/`pending`/`incomplete`
- Filter helpers: `NormalizeFilter()`, `ValidateFilter()` on `AzdoService`

---

### Earlier History (2026-03 through 2026-05-20)

- Confirmed the MCP tool exception pattern is `catch (Exception ex) when (...)` followed by `throw new McpException($"Failed to {action}: {ex.Message}", ex);`, preserving the original exception as `InnerException` for debugging.
- `azdo_auth_status` is **not** sync-safe in its current shape: `AzCliAzdoTokenAccessor.AuthStatusAsync()` can await `_resolutionLock.WaitAsync(...)` and perform fallback credential resolution through `AzureCliCredential` or `az account get-access-token` on cache miss.
- PR #53 tracks the `helix_ci_guide` exception wrap and the auth-status audit follow-up.
See history-archive.md for complete history.

# Summary (archived older history before 2026-05-20)

See history-archive.md for complete history including AzDO auth patterns, MCP SDK upgrades, CLI schema generation, release conventions, and earlier learnings from 2025-03 through 2026-03.

## Learnings — dnceng feed & Helix.Client version format (2026-05-21)

**Feed URL pattern:** `https://pkgs.dev.azure.com/dnceng/public/_packaging/dotnet-eng/nuget/v3/flat2/{package-id}/index.json` for the flat index. The registration endpoint (for publish dates) is accessed via the GUID-based URL found in the index response's `items[].@id` fields.

**Version number format:** `11.0.0-beta.{YYMDD}.{build}` where `M` is the **single-digit month** (1–9 for Jan–Sep, 10–12 for Oct–Dec) and `DD` is zero-padded two-digit day. So `26110` = YY=26, M=1(Jan), DD=10 → **Feb 10, 2026** (the publish date, not necessarily the commit date — there is a pipeline delay of ~1 day). The embedded date reflects the **build pipeline date**, not the git commit date.

**Gotcha:** The task-prompt shorthand "26110 = 2026-01-10 build" is misleading — NuGet registration shows `26110.116` was actually published **2026-02-10**. The number encodes the Arcade daily build pipeline run date.

**Source repo:** `dotnet/arcade` (not `dotnet/arcade-services`). Helix.Client lives at `src/Microsoft.DotNet.Helix/Client/CSharp/`. Search commits with `gh api "repos/dotnet/arcade/commits?path=src/Microsoft.DotNet.Helix/Client&since=..."`.

**Multiple major versions:** The dnceng feed publishes 11.x, 10.x, 9.x, 8.x, 6.x simultaneously (different SDK stream branches). We are on 11.x which is the head/main stream. Stick to 11.x when bumping.

**Daily build cadence:** The feed produces multiple builds per day (build numbers 100–130 = internal official builds; 1–9 = PR/validation builds). The highest-numbered build of the day is typically the last official build.

## Learnings — CPM bump for v0.7.1 (2026-05-21)

**CPM bump workflow confirmed:** For a pure version-bump PR in this repo, the only file to touch is `Directory.Packages.props`. No `.csproj` changes needed — CPM resolves versions centrally. Restore picks up new versions automatically on first run after the props edit.

**Azure.Identity deprecation behavior:** NuGet marks 1.13.2 as deprecated with reason "Other" (not "HasVulnerability" or "CriticalBugs"). The deprecation is purely a "please upgrade" signal, not a security advisory. Upgrading to 1.21.0 clears it. The type-forwarding change (types moved from Azure.Identity to Azure.Core via `[TypeForwardedTo]`) is binary-compatible — our `AzureCliCredential` usage compiles and runs unchanged.

**Microsoft.Data.Sqlite major version cross (9→10):** Moving from 9.0.7 to 10.0.8 crosses a major version boundary but is safe here because we target `net10.0` and there are no API surface changes for our usage patterns (basic connection/command/reader). SQLitePCLRaw transitive dependencies bump automatically.

**PR #54:** `chore(deps): bump 6 packages for v0.7.1` — branch `chore/v0.7.1-deps`, all 6 bumps in one commit. Build green (0/0), tests green (1180/1180). Lewing will merge and tag v0.7.1 separately.

## Learnings — Release flow v0.7.1 (2026-05-21 12:55Z)

**Three version stamps to bump:** The workflow validates all three before creating a release:
1. `src/HelixTool/HelixTool.csproj` — `<Version>0.7.1</Version>`
2. `src/HelixTool/.mcp/server.json` — top-level `"version": "0.7.1"` and package `"version": "0.7.1"`

**Build & test gate:** Both must pass (0 errors, 0 warnings; 1180/1180 tests) before commit.

**Never manual `gh release create`:** The `publish.yml` workflow uses `ncipollo/release-action` and creates the GitHub Release automatically when the tag is pushed. Manually running `gh release create` causes a 422 error and **skips the NuGet push step** entirely.

**Release flow (in order):**
1. Verify all THREE version stamps match the tag (e.g., 0.7.1)
2. Build clean + all tests pass
3. Commit version bumps: `git commit -F -` with detailed changelog
4. Push main: `git push origin main`
5. Create annotated tag: `git tag -a v0.7.1 -m "Release v0.7.1..."`
6. Push tag: `git push origin v0.7.1` ← **workflow triggers here**
7. Workflow auto-creates GitHub Release with nupkg asset; no manual steps needed

**Timing:** Workflow runs in ~33s (validation, pack, create release, NuGet auth, push to NuGet). Watch with `gh run watch <id> --exit-status`.

**Verification:** After workflow completes, `gh release view v0.7.1 --json assets` confirms `lewing.helix.mcp.0.7.1.nupkg` is attached. Release URL: `https://github.com/lewing/helix.mcp/releases/tag/v0.7.1`

**All steps confirmed working 2026-05-21 v0.7.1 release.**

## Learnings — Strict Coordinator Dispatch Rule (2026-05-21 v0.7.1)

- Coordinator dispatched the entire v0.7.1 release workflow to Ripley (claude-haiku-4.5) per the strict role-to-model map, with no ad-hoc hand-offs or human intervention mid-stream.
- Ripley executed cleanly: merged PR #54, bumped 3 version stamps (HelixTool.csproj + server.json variants), pushed tag, watched publish.yml workflow complete (26243596534, 33s, all green), verified asset on nuget.org.
- No deviations from the established pattern (ship-after-merge, tag-based trigger, ncipollo/release-action auto-creation).
- **First release enforcing strict dispatch rule end-to-end.** Success signal for scaling Squad's role-based task routing.

## Learnings — v0.7.2 Design: Surface WorkItemSummary fields (2026-05-21 Dallas)

Dallas filed design proposal in `.squad/decisions/inbox/dallas-surface-workitem-fields.md` (Brady approved option B: surface + optimize `GetJobStatusAsync`). Proposal details interface changes to `IWorkItemSummary`, adapter wiring, ~95% API call reduction for jobs with mostly-passing items, test plan, and risks. Extracted reusable skill guidance to `.squad/skills/sdk-adapter-extension/` for future SDK field surfacing. Ripley to implement on branch `feat/workitem-summary-exit-code`.

