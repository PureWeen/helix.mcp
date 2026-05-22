# Skill: Codebase Slop Audit (C#)

**Domain:** Code quality / technical debt discovery  
**Language:** C# (but method generalizes to any language)  
**Effort:** 2–4 hours for mid-size codebase (10K–50K LOC)  
**Output:** Markdown report with prioritized findings, refactor recommendations, risk assessments

## Definition of Slop (C#-specific)

Slop ≠ bugs or style violations. Slop = **low-value, redundant code patterns that add maintenance burden without functional benefit:**

- **Copy-paste duplication:** Near-identical methods/catch blocks across 2+ files (same logic, cosmetic variation)
- **Boilerplate that should be a helper:** Repeated try/catch+log, exception wrapping, JSON serialization
- **Dead code:** Unused `public` methods, commented-out blocks, orphaned test methods
- **Pseudo-abstraction:** Wrappers that delegate without adding value; interfaces with single implementations that don't aid testability
- **Schema drift:** DTOs/records that mirror each other with small differences (common in API → internal model → CLI model pipelines)
- **String-template slop:** Repeated error message patterns (`$"Failed to {action}: {ex.Message}"`)
- **Over-broad catches:** `catch (Exception)` masking specific failures; shadow catching with poor re-throw
- **Magic number/string repetition:** Same literal in 5+ call sites (should be `const` or helper)
- **Long methods screaming for extraction:** 100+ LOC with clear logical phases
- **Test slop:** Repeated arrange blocks; copy-paste test methods varying by 1 parameter (should be `[Theory]`/`[InlineData]`)

## Audit Method (Systematic)

### Phase 1: Baseline & Survey (30 min)

1. **Count LOC by directory:**
   ```bash
   find src -name '*.cs' -not -path '*/obj/*' -not -path '*/bin/*' \
     -not -name '*.g.cs' -type f -exec wc -l {} + | sort -rn | head -20
   ```
   - Establish baseline (total LOC).
   - Identify largest files (slop often correlates with size).

2. **List all source files by size:**
   ```bash
   find src -name '*.cs' -type f ! -path '*/obj/*' | xargs wc -l | sort -rn
   ```
   - Top 10–20 files merit deep inspection.

3. **Mark priority tiers:**
   - **Tier 1:** API/tool layers (highest duplication risk—many similar operations)
   - **Tier 2:** Service/business logic (moderate duplication—shared patterns)
   - **Tier 3:** Models/DTOs (drift risk—schema evolution)
   - **Tier 4:** Tests (arrange-block slop, theory candidates)

---

### Phase 2: Pattern Hunting (1–2 hours)

**For each Tier 1 file:**

1. **Grep for structural patterns:**
   - Exception handlers: `grep -n "catch.*when\|catch (Exception" <file>`
   - Repeated methods: `grep -n "private\|public.*(" <file> | sort`
   - JSON decorators: `grep -c "\[JsonPropertyName\|[Serializable\|[DataMember" <file>`
   - Error messages: `grep "throw new.*Exception\|Error:\|Failed" <file> | sort | uniq -c | sort -rn`
   - Repeated strings: `grep "const.*=\|readonly.*=" <file>` (identify what *should* be `const`)

2. **Cross-file diff for sibling types:**
   - If you see `*Tools.cs` and `*Service.cs`, compare public method signatures.
   - If you see `*Result.cs` and `*JsonResult.cs`, diff property lists.
   - If you see `*Summary.cs` and `*Response.cs`, look for schema duplication.

3. **Identify long methods (>80 LOC):**
   ```bash
   grep -n "^[[:space:]]*\(public\|private\|internal\).*{" <file> | \
     while read line; do
       linenum=$(echo "$line" | cut -d: -f1)
       nextline=$(tail -n +$linenum <file> | head -1)
       # Rough heuristic: count LOC until next method or closing brace
     done
   ```

4. **Scan for dead code markers:**
   ```bash
   grep -n "// TODO\|// FIXME\|// HACK\|throw new NotImplementedException\|[Obsolete" <file>
   ```

---

### Phase 3: Categorize & Rank (30 min)

For each finding, assess:

| Attribute | Assess | Example |
|---|---|---|
| **Type** | Copy-paste, boilerplate, drift, dead, pseudo-abstraction, slop string | "Copy-paste catch handlers" |
| **Count** | How many sites? | 16 catch blocks across 2 files |
| **LOC/Impact** | Duplicated lines? (multiply by site count) | 1 LOC per catch × 16 = 16 LOC, but high maintenance cost |
| **Severity** | HIGH (10+ lines × 3+ sites OR 50+ lines × 2 sites), MEDIUM (3–10 lines × 3+ sites), LOW (cosmetic) | HIGH: 6 DTO classes × 60 LOC each duplicated |
| **Risk** | Low (mechanical refactor), Medium (touches public API/tests), High (changes behavior) | Low: Moving DTOs is schema consolidation only |
| **Effort** | Time to fix (min) | 60 min for DTO consolidation |

---

### Phase 4: Report & Recommend (30 min)

**Mandatory sections:**

1. **Executive summary** (3–5 lines):
   - Total findings by severity.
   - Top 3 highest-impact findings (1-line description each).
   - Single recommended next step.

2. **Detailed findings** (per HIGH/MEDIUM finding):
   - Type, count, LOC impact.
   - Exact file + line ranges.
   - Refactor sketch (2–3 lines).
   - Risk assessment.

3. **Aggregate statistics table:**
   - Category | Count | LOC | Priority

4. **Codebase health grade:**
   - A (clean, <0.1% duplication), B (good, <0.5% duplication), C (moderate slop, needs cleanup), D (high tech debt)

5. **Recommendations:**
   - What to fix first (prioritized by effort/impact ratio).
   - What NOT to fix (e.g., intentional schema separation).
   - Effort estimate for each.

---

## Checks: False Positives to Exclude

**NOT slop:**
- **Intentional schema separation** (API model ≠ CLI model ≠ service model). This is justified for API stability.
- **Private test helper methods** repeated across test files. (Expected; testing often needs localized fixtures.)
- **Framework boilerplate** (e.g., test setup, dependency injection). Necessary.
- **Performance micro-patterns** (e.g., `string.IsNullOrEmpty()` checks repeated for safety). OK.
- **Legitimate overloads or parameterized methods** that vary only in optional parameters. Not duplication.

---

## Tools & Commands Reference

### C#-specific grep patterns:

```bash
# Exception handlers
grep -n "catch.*when" *.cs
grep -n "catch (Exception)" *.cs

# Repeated error messages
grep "throw new.*Exception" *.cs | sed 's/.*throw //' | sort | uniq -c | sort -rn

# JSON/serialization boilerplate
grep -c "\[JsonPropertyName\|\[Serializable\|JsonConverter" *.cs

# Unused public methods (requires semantic analysis; heuristic: public methods never called)
grep -n "^[[:space:]]*public.*(" *.cs

# TODO/FIXME dead code markers
grep -rn "// TODO\|// FIXME\|// HACK" .

# Test copy-paste candidates (repeated method names with numeric suffixes)
grep -n "Test.*1\|Test.*2\|Test.*_A\|Test.*_B" *Tests.cs
```

### Size analysis:

```bash
# Top 10 files by LOC
find src -name '*.cs' ! -path '*/obj/*' -type f -exec wc -l {} + | sort -rn | head -10

# Identify methods >80 LOC (rough heuristic using Cloc)
# In practice, rely on IDE or LSP to highlight long methods
```

---

## Applying to HelixTool (Case Study)

**Audit executed:** 2026-05-22  
**Key findings:**
1. Result DTO duplication (Program.cs vs McpToolResults.cs): 6 classes, 60 LOC
2. Catch-throw boilerplate: 16 identical handlers
3. JSON attribute placement inconsistency: 48 properties

**Follow-up slop audits recommended:** Quarterly, or after major feature additions.

---

## Generalization to Other Languages

| Language | Key patterns | Tool |
|---|---|---|
| TypeScript/JS | Duplicate interface definitions, repeated error handlers, unused exports | `grep`, ESLint (dead-code plugin) |
| Python | Repeated exception handlers, unused imports, copy-paste functions | `grep`, `pylint --disable-all --enable=W` |
| Java | Duplicate catch handlers, repeated null checks, unused methods | `grep`, Spotbugs (dead-code detector) |
| Go | Repeated `if err != nil { return err }`, unused exported functions | `grep`, `go vet` |

---

## Audit Frequency

- **Initial:** Quarterly (after major features or refactoring pushes)
- **Maintenance:** Annually (if codebase is stable)
- **Trigger:** After large consolidation (e.g., PR merging multiple tools) to ensure slop wasn't introduced

---

## Notes

- This audit method prioritizes **high-impact findings** (duplicated schema, repetitive boilerplate) over cosmetic linting (formatting, naming).
- Slop differs from **anti-patterns** (e.g., shared mutable state, circular dependencies). Use separate anti-pattern audit if needed.
- Slop differs from **over-engineering** (e.g., over-abstracted code). Slop is *under-refactored* duplication; over-engineering is unnecessary abstraction.
