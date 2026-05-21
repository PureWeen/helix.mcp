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

📌 Team update (2026-05-21): Pagination contract tests — wrote 13 tests (333 LOC) for Phase 1+2 pagination spec in src/HelixTool.Tests/AzDO/PaginationContractTests.cs. All 13/13 passing; full suite 1180/1180 passing. Commits 181ff5b + d5fde34. ⚠️ BRANCH-HYGIENE: committed to local main instead of squad/pagination-standardize per manifest instruction; Larry will handle branch/push decision.

📌 Cross-agent heads-up (2026-05-21T17:22Z, from Ripley dependency audit): xunit v3 migration held for v0.8.0+ (not v0.7.1); Roslyn 5.3.0 bump also held (requires generator verification). FYI for test framework planning.

# Summary (archived 19 older entries)

See history-archive.md for detailed history.
