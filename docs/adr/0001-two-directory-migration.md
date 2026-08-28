# ADR 0001: Two-directory migration model (source → target)

## Status

Proposed

## Context

The `migrate-spring-to-quarkus` skill currently migrates projects **in place**: the agent transforms the Spring Boot project inside its own directory, 
and the migration result replaces the original code.

Working with IBM Research on merging their `spring2quarkus` skill (PR #49, issue #58) surfaced an alternative: migrating into a **separate target directory**, keeping the source project untouched. 
The main drivers are:

1. **Deterministic validators.** The Java validators contributed by IBM (issue #39) verify migration correctness by comparing metadata extracted from the Spring code against metadata extracted from the Quarkus code (entities, REST endpoints, messaging channels, configuration). 
This comparison requires both versions of the code to exist at the same time. 
With in-place migration the original code is gone once a module runs, so the validators cannot work without git gymnastics.
2. **Cleaner reasoning for agents.** With a read-only source, the agent cannot accidentally destroy unmigrated code, and "what is left to migrate" is always answerable by comparing the two trees. 
The target project follows Quarkus conventions from scratch instead of inheriting the legacy layout.
3. **Auditability.** The diff between source and target is a complete, permanent record of the migration.

The trade-offs considered: in-place migration is simpler to explain, works naturally with the existing git workflow (branch per run on the same repo), and does not duplicate disk usage.

## Decision

The skill adopts a **two-directory migration model**:

- **Source directory**: the existing Spring project. Read-only, with one exception: the skill may write extraction metadata describing the source app (repo metadata, dependency analysis, code metadata, messaging and config extractions) under `<source>/migration-metadata/`. 
These files describe the source and are reusable across migration runs against the same project, so they live with it. 
All source-side files go under `<source>/migration-metadata/`; nothing is ever written at the source root or anywhere else in the source tree.
- **Target directory**: the generated Quarkus project. All migration artifacts other than the source-side extractions (reports, spec, summary, progress state) live here.
- **Target resolution**:
  - Autonomous mode: default to a sibling directory named `<source-name>-quarkus`. If the default target already exists: resume the migration when it contains `migration-metadata/migration-context.json`, otherwise stop with an error asking for an explicit target.
  - Interactive mode: propose the same default and ask for confirmation together with the strategy question, in a single interaction.
- **After the migration completes**: `migration-summary.md` is meant to be kept as the record of the migration. The remaining artifacts (reports, metadata, spec, progress state) are working files: the skill adds `.gitignore` entries for them in the target project so they never end up committed, and they can be deleted once the migration is accepted.

## Consequences

- The deterministic validators (issue #39) can run as designed, comparing source and target extractions.
- `SKILL.md` and the modules must resolve and thread two paths instead of assuming the working directory is the project (build, code, frontend, testing, cleanup).
- The final verification checks run against the target directory.
- The git workflow module must be redesigned: the target is a new repository, so the branch-per-run model on the original repo no longer applies as-is.
- Documentation (skill README, repo README) must describe the two-directory usage.
- Migrating the same source repeatedly (benchmark runs) can reuse the source-side extractions in `<source>/migration-metadata/`.
- Resuming an interrupted migration requires knowing the target directory, since the progress state (`migration-context.json`) lives there. The default sibling naming makes it discoverable from the source path; with a custom target the user must provide the path again.
- Disk usage doubles per migration run; acceptable for the project sizes targeted.

### Changes to the test sub-project (`./tests`)

Today the harness copies `tests/projects/<name>/source/` into a workdir and the agent migrates it in place; all checks run against that single directory. With this decision:

- The runners create a separate target directory per run (e.g. `target/workdirs/<name>/` as read-only source copy and `target/workdirs/<name>-quarkus/` as migration target) and pass both paths in the prompt.
- All automated checks (build, tests pass, no Spring deps, has Quarkus, starts up, smoke tests) point at the target directory.
- Each project under `tests/projects/` keeps, next to `source/`, a **reference migrated project** (e.g. `migrated/`). This reference allows:
  - a user to verify and compare a migration run against a known-good result,
  - documenting the migration process step by step,
  - a CI job to compile and test both the source and the migrated reference,
  - supporting several source versions over time (Spring Boot 3.x, 4.x, ...) with their corresponding references.

## Open questions

- **Target-side layout.** The migration artifacts currently land as three entries at the target root (`migration-reports/`, `migration-metadata/`, plus `migration-spec.yaml` and `migration-summary.md`). Grouping everything under a single directory (e.g. `.migration/`) would keep the generated project root clean and make the `.gitignore` a single line. To be discussed in #58; does not change the decision above.