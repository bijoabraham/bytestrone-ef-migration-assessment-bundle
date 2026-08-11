# bytestrone-ef-migration-assessment-bundle (v1.1.6)

> **Type**: Read-Only Mining Codemod & Codemod Insights Metrics Emitter
> **Target**: .NET Framework 4.x / EF6 Solution Repositories

## Overview

`bytestrone-ef-migration-assessment-bundle` is a read-only **mining codemod**
that assesses Entity Framework 6 (EF6) to EF Core 8 migration readiness,
architectural risk, and modernization effort for a .NET repository, before a
project-management team commits to a migration plan or estimate.

It performs **zero code modifications**. It emits standard
[Codemod Insights](https://docs.codemod.com/platform/insights) metric events
by delegating its scan to the published
`bytestrone-ef-csharp-pattern-mining` package.
It does not generate a standalone HTML report — the metrics below are meant
to be viewed as Codemod Insights dashboard widgets, which is also where the
readiness/risk/effort composite scores are computed (see
[Dashboard formulas](#dashboard-formulas-readiness-risk-effort) below).

---

## Architecture

One workflow node, one `codemod` step, delegating to
`bytestrone-ef-csharp-pattern-mining`, which itself runs a single
`js-ast-grep` step whose `include` glob matches `**/*.cs` **and**
`**/*.csproj`, `packages.config`, `App.config`, `Web.config`,
`appsettings*.json` — the engine's native per-file walker invokes it
separately for every matched file, with no shared state. All mining
metrics — per-pattern usage, the unified blocker rollup, project inventory,
configuration surface area, and dependency risk — come from that one step,
so every dashboard widget binds to a single, consistent data source.

```
workflow.yaml (this package)
  └─ codemod step → bytestrone-ef-csharp-pattern-mining@1.4.1 (registry)
       └─ single js-ast-grep step, per-file dispatch → scripts/codemod.ts
```

`codemod.source` is pinned to `bytestrone-ef-csharp-pattern-mining@1.4.1` on
the registry. Bump the pin explicitly when the mining package publishes a
new version — this package does not float to `latest`. See the mining
package's own README for why v1.4.0 replaced an earlier lock-guarded
whole-repo `fs` walk design that failed on the hosted platform.

---

## Emitted metrics

These are emitted by the delegated mining package. Full cardinality details
live in its README; summarized here:

| Metric | Cardinality | What it's for |
| --- | --- | --- |
| `ef_objectcontext_usages`, `ef_idbset_usages`, `ef_entity_type_configuration_usages`, `ef_dbconfiguration_usages`, `ef_execute_sql_command_usages`, `ef_execute_sql_query_usages`, `ef_set_initializer_usages`, `ef_virtual_nav_props`, `ef_db_interception_usages` | `{filepath, linenumber, explanation}` | One metric per legacy EF6 pattern — detail/trend widgets |
| `ef_dbcontext_usages`, `ef_dbset_usages` | `{filepath, linenumber, explanation}` | Informational only — these names are already EF Core-compatible |
| `ef_migration_blocker` | `{blockerType, severity, file, linenumber}` | Unified rollup of every legacy-pattern hit above (excludes the two informational metrics) — the source for a single "all blockers" table/timeseries widget |
| `total_projects` | `{file}` | One row per `.csproj` found |
| `legacy_csproj_count` | `{file}` | Subset using the pre-SDK MSBuild project format |
| `ef_version` | `{packageId, version, file, source}` | Detected EF package references across `.csproj`/`packages.config` |
| `ef_config_surface` | `{configType, name, file}` | `connectionString` / `entityFrameworkSection` / `appsettingsConnectionStrings` hits in `App.config`/`Web.config`/`appsettings*.json` |
| `ef_dependency_risk` | `{packageId, version, source, file, riskTier, risk, targetVersion}` | Every NuGet/GAC package reference across `.csproj`/`packages.config`, classified into `supported` / `requires-upgrade` / `deprecated` / `unsupported` / `custom-binary` / `gac` |
| `ef_loc_inventory` | `{file, loc}` | Non-blank line count per `.cs` file — a codebase-size sizing signal for effort/cost formulas |

---

## Dashboard formulas: readiness, risk, effort

`codemod:metrics` only supports `increment()` on a per-file, per-step basis —
there is no gauge/set operation, and no way for the codemod script itself to
combine several different metrics into one computed value (see the mining
package's README for why). Composite scores like "EF Core 8 readiness %" are
therefore built as **Insights dashboard Formula widgets** that combine the
raw metric widgets above, not emitted by the script. This also means the
weights below are tunable per team from the dashboard UI, without
republishing the codemod.

Suggested widget queries (each `SUM`s the named metric, optionally filtered
by cardinality) and the formula referencing them by the platform's
alphabetical variable convention:

| Var | Query |
| --- | --- |
| A | `SUM(ef_migration_blocker WHERE severity="critical")` |
| B | `SUM(ef_migration_blocker WHERE severity="warning")` |
| C | `SUM(total_projects)` |
| D | `SUM(legacy_csproj_count)` |

**EF Migration Risk Score (0-100)** — weighted blocker density per project,
critical patterns (`ObjectContext`, `DbConfiguration`, `ExecuteSqlCommand`,
`SqlQuery`, `SetInitializer`, `DbInterception` — all have no direct EF Core
equivalent) weighted higher than warning patterns (`IDbSet`,
`EntityTypeConfiguration`, virtual nav props — mechanical renames/refactors):

```
ef_risk_score = MIN(100, ROUND((A * 10 + B * 4) / MAX(C, 1)))
```

**EF Core 8 Modernization Readiness Index (%)** — inverse of risk:

```
efcore8_readiness_score = 100 - ef_risk_score
```

**EF 6.5 Upgrade Readiness Index (%)** — a lighter-weight readiness measure
for staying on the EF6 family and just modernizing tooling (patch/point
upgrade), which mainly hinges on project-file format rather than API
patterns:

```
ef65_readiness_score = ROUND(100 * (C - D) / MAX(C, 1))
```

**Effort estimates** — a starting heuristic, not a calibrated model. Tune the
weights and hours-per-point to your team's actual velocity once you have a
few migrated files to compare against:

```
estimated_story_points = ROUND((A * 1.5 + B) / 10)
estimated_dev_hours   = estimated_story_points * 6
```

**Dependency risk** — `ef_dependency_risk` is a separate signal from the
code-pattern blockers above (it's about *what you depend on*, not *what your
code does*), so it's kept as its own widget rather than folded into
`ef_risk_score`:

```
E = SUM(ef_dependency_risk WHERE riskTier="unsupported" OR riskTier="gac")
F = SUM(ef_dependency_risk WHERE riskTier="requires-upgrade" OR riskTier="deprecated" OR riskTier="custom-binary")
```

A table widget grouped by `riskTier`/`packageId` (using `ef_dependency_risk`
directly) is usually more useful to a PM than a single collapsed number —
"3 unsupported dependencies, here's which ones" is more actionable than "E=3".
If you do want it folded into the overall risk score, add `E * 8 + F * 3` to
the `ef_risk_score` formula's numerator above.

---

## Testing

```bash
npm test    # validates workflow.yaml (this package has no scripts of its own)
```

The actual transform logic and its tests live in
`bytestrone-ef-csharp-pattern-mining` — run `npm test` there when changing
detection logic. To smoke-test the full delegated pipeline end-to-end:

```bash
npx codemod workflow run --workflow workflow.yaml \
  --target path/to/a/dotnet-repo --dry-run --allow-fs --allow-dirty --no-interactive
```

## Execution

```bash
# Run locally against a target .NET repo
npx codemod workflow run --workflow workflow.yaml --target path/to/your/dotnet-repo --allow-fs

# Publish to Codemod Registry
npx codemod publish
```
