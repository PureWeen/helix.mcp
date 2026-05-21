
**2026-05-21 10:33Z:** Ash audit found helix_ci_guide needs exception wrapping (5-line fix). See decisions.md for details.

**2026-05-21 12:22Z:** Ran full dependency audit against Directory.Packages.props. Key findings:
- Azure.Identity 1.13.2 is NuGet-deprecated ("Other") — update to 1.21.0 clears the flag; types type-forwarded to Azure.Core in 1.21, non-breaking for our `AzureCliCredential` usage.
- Microsoft.Data.Sqlite 9.0.7 → 10.0.8 is a natural net10 alignment bump.
- Microsoft.Extensions.* (DI/Hosting/Http) at 10.0.0–10.0.3 can be patched to 10.0.8 safely (servicing releases).
- Microsoft.CodeAnalysis.CSharp 4.12.0 → 5.3.0 is a major version for the generator project; warrants a compile check before shipping.
- ConsoleAppFramework 5.7.13 and ModelContextProtocol 1.3.0 are at latest.
- Microsoft.DotNet.Helix.Client reports "Not found at sources" — only updates from the dnceng feed, so leave pinned.
- xunit 2.x is marked Legacy; test framework migration is out of scope for a patch.
- No vulnerable packages found. Recommended ship 5 🟢 packages (Azure.Identity, Microsoft.Data.Sqlite, 3x Microsoft.Extensions.*) as v0.7.1.

## Learnings — Pagination Standardization Implementation (2026-05-20, commit 1a2e1d0)

**Pattern learned:** When changing service-layer return types:
1. Update the record definition (or add new wrapper)
2. Update the service method signature + implementation
3. Update all tool/CLI call sites
4. Update tests that directly call the service
5. Clean build to avoid stale reference errors

📌 Team update (2026-05-21): Pagination Phase 1+2 implemented — wrapped `azdo_changes`/`azdo_test_runs` in `LimitedResults<T>`, added `truncated`/`note` to 8 result types. Build clean. Commit 0a82e58. Full suite: 1180/1180 passing.

## Learnings — RollForward policy for global tool (2026-05-21)

- Set `<RollForward>Major</RollForward>` only in `src/HelixTool/HelixTool.csproj` for the generated `HelixTool.runtimeconfig.json`.
- Do not add `RollForward` to library projects; this startup policy is only consumed by the executable entry point/runtimeconfig.

## Learnings — MCP exception cleanup (2026-05-21)

- Confirmed the MCP tool exception pattern is `catch (Exception ex) when (...)` followed by `throw new McpException($"Failed to {action}: {ex.Message}", ex);`, preserving the original exception as `InnerException` for debugging.
- `azdo_auth_status` is **not** sync-safe in its current shape: `AzCliAzdoTokenAccessor.AuthStatusAsync()` can await `_resolutionLock.WaitAsync(...)` and perform fallback credential resolution through `AzureCliCredential` or `az account get-access-token` on cache miss.
- PR #53 tracks the `helix_ci_guide` exception wrap and the auth-status audit follow-up.

# Summary (archived older history before 2026-05-20)

See history-archive.md for complete history including AzDO auth patterns, MCP SDK upgrades, CLI schema generation, release conventions, and earlier learnings from 2025-03 through 2026-03.
