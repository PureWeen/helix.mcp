---
name: "dependency-audit"
description: "How to audit NuGet dependencies in this repo for updates, deprecations, and vulnerabilities"
domain: "dependencies, maintenance"
confidence: "high"
source: "earned"
tools: []
---

## Context

Run periodically before patch releases or when a dependency deprecation/vulnerability is flagged. This repo uses Central Package Management (CPM) — all versions live in `Directory.Packages.props`.

## Patterns

### 1. Gather the full picture

```bash
# From repo root — covers all projects in one pass
dotnet list package --outdated --include-transitive
dotnet list package --deprecated
dotnet list package --vulnerable
```

Focus on **Top-level Package** rows (direct deps) from the `--outdated` output — transitive rows are informational unless they're security or the direct dep controls them via a floating range.

### 2. Classify the gap

| Gap type | Signal | Default stance |
|----------|--------|----------------|
| Patch (x.y.Z) | Servicing only | 🟢 Bump immediately |
| Minor (x.Y.z) | New features, typically additive | 🟢 Bump unless API breakage notes |
| Major (X.y.z) — .NET versioning | Follows runtime version cadence | 🟢 Bump when targeting the matching runtime |
| Major (X.y.z) — third-party | May break API surface | 🟡 Check release notes first |
| Preview-only available | Upstream not stable | 🔴 Hold unless existing pin is preview |

### 3. Check deprecation flags

- **"Other" with no alternative** → NuGet/author marking old minor versions to push users to latest of same package. Safe to update to latest stable.
- **"Legacy" with named alternative** (e.g., `xunit` → `xunit.v3`) → This is a migration, not a bump. Plan separately.
- **"Security"** → Treat as urgent.

### 4. Azure SDK packages

Azure SDK packages (`Azure.*`) move in step. Updating `Azure.Identity` typically also bumps `Azure.Core` and `System.ClientModel` transitively. The Azure SDK changelog is at: `https://github.com/Azure/azure-sdk-for-net/blob/main/sdk/{area}/{PackageId}/CHANGELOG.md`

### 5. .NET versioning lockstep

`Microsoft.Data.Sqlite`, `Microsoft.Extensions.*`, `Microsoft.AspNetCore.*` follow the .NET runtime version (e.g., 9.x for net9, 10.x for net10). When the target framework is net10.0, bumping these from 9.x → 10.x is a natural alignment move, not a risky major.

### 6. dnceng beta packages

`Microsoft.DotNet.Helix.Client` is published only to the `dnceng/public` feed, often as beta builds. `dotnet list package --outdated` reports "Not found at the sources" for the latest. Check the feed directly if an update is needed, or leave pinned to the build you depend on.

### 7. Source generator packages (Microsoft.CodeAnalysis.CSharp)

Roslyn version bumps for the analyzer/generator project (`HelixTool.Generators`, netstandard2.0) should be validated with a compile after bumping — run `dotnet build src/HelixTool.Generators` to confirm no API breakage before merging.

## Anti-Patterns

- **Don't bump transitive-only packages directly** — they're resolved by the owning direct dep; bumping them in CPM can create inconsistencies or conflict warnings.
- **Don't upgrade test framework packages (xunit, NUnit) as a "bump"** — these are migrations; treat separately.
- **Don't bump `Microsoft.DotNet.Helix.Client` speculatively** — it's a beta feed package; only bump if a new beta is known to be needed.
- **Don't conflate .NET major bumps (e.g., 9→10 for EF/Sqlite) with third-party major bumps** — .NET versioning conventions make the former low-risk.
