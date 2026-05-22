# Ash — History (Summarized)

## Current Session (2026-05-22)

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
