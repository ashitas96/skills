# Planning module and migration-spec.yaml 

- Status: proposed
- Date: 2026-09-22
- Issue: [#53](https://github.com/quarkusio/skills/issues/53)

## Context and Problem Statement

[ADR-0002](https://github.com/quarkusio/skills/pull/77/) introduced
the `planning` module as one of the new modules and defined its responsibilities at a
high level. [ADR-0003](https://github.com/quarkusio/skills/pull/81/) defined the full
schema of `migration-spec.yaml`.

This ADR specifies how the planning module collects user decisions, the two-file
model governing `.quarkus-migration.yml` and `migration-spec.yaml`, the selective
feature flag approach, and the gate dependency between planning and the prerequisite
module.

The `migrate-spring-to-quarkus` skill currently has no module that captures critical
features requiring frequent lookup and user transformation preferences in interactive
mode. Strategy, Quarkus version, and Java version are resolved ad-hoc inline inside
`SKILL.md` Step 1, then the skill immediately begins executing modules. This means:

- Decisions are not machine-readable and can get lost on session interruption.
- Every module either re-scans the source independently (wasting tokens) or relies on
  implicit prompt context.
- There is no durable artifact in the target directory recording what was found and
  what was decided before transformation began.

## Decision Drivers

- **Single decision point.** Strategy, Quarkus version, Java version, persistence
  choice, messaging transport, and view technology should be asked once and recorded
  once. Downstream modules read the record rather than re-asking or re-inferring.
- **Token efficiency.** A structured scan written at the start of the run avoids each
  module independently re-reading the source project to detect the same features.
- **Traceability.** Every decision and its source (`argument | config-file | user`)
  must be recoverable from the target directory after the run.

## Considered Options

### Keep decisions inline in `SKILL.md` (status quo)

Strategy and version are asked as part of Step 1 and carried implicitly in prompt
context. No file is written.

Simple, no new files. But decisions are not machine-readable, every new module must
re-derive them, and there is no persistent record in the target directory.

### Dedicated `modules/planning/planning.md` module writing `migration-spec.yaml`

A new module runs after `prerequisite` and before the transformation modules. It scans
the source project, collects decisions from the user (or auto-selects in non-interactive
mode), and writes `migration-spec.yaml` into the target directory. All downstream
modules read from this file.

More structure up front, but produces a durable artifact that serves as the shared
contract for all current and future modules (discovery #55, reporting #59, resumable
state #40).

## Decision

Introduce `modules/planning/planning.md` that runs **ALWAYS**, after `prerequisite`
checks pass, and writes `migration-spec.yaml` into `<target>/migration-spec.yaml`.

### Gate condition

**ALWAYS** — provided the `prerequisite` module has passed. If any hard prerequisite
fails (missing JDK, missing build tool), the migration is aborted before planning runs.

### Two-file model: `.quarkus-migration.yml` and `migration-spec.yaml`

These two files serve different purposes. The overlap between them is intentional:

| | `.quarkus-migration.yml` | `migration-spec.yaml` |
|---|---|---|
| Written by | The user, manually, before the run | The planning module, automatically |
| When | Before the migration starts | During the planning phase |
| Where | Source project root | Target directory (`<target>/migration-spec.yaml`) |
| Purpose | Pre-declare choices to skip interactive questions | Full record of everything detected and decided |
| Lifecycle | Unchanged across runs | Regenerated each run |

The resolution order for any decision is:
**skill argument → `.quarkus-migration.yml` → ask user (interactive) / auto-select (non-interactive)**

The resolved value is written into `migration-spec.yaml` with a `strategy_source` field
recording where it came from (`argument | config-file | user | agent-selected`).

**Downstream modules always read from `migration-spec.yaml` only.** No module reads
`.quarkus-migration.yml` directly. There is always a single place to look for the
current run's decisions, regardless of how they were originally provided.

### User decisions

The planning module collects decisions in two stages. Stage 1 is always collected
before Stage 2 questions are shown. In non-interactive mode, defaults are applied
without asking; all chosen values and their rationale are still written to
`decisions[]` in `migration-spec.yaml`.

**Stage 1 — always collect:**

| # | Decision | Interactive | Non-interactive default |
|---|---|---|---|
| 1 | Target Quarkus version | Ask: latest stable (resolved via Maven Central) or specify | Latest stable |
| 2 | Target Java version | Ask: 17 (LTS) or 21 (LTS, virtual threads) | 17 |
| 3 | Migration strategy | Ask: `full-quarkus` or `spring-compat` | `full-quarkus` |

> Non-interactive defaults for Java version, Quarkus version, and strategy are
> subject to confirmation — see [issue #53](https://github.com/quarkusio/skills/issues/53).

**Stage 2 — conditional on Stage 1 answers and detected features:**

| # | Decision | Condition | Options |
|---|---|---|---|
| 4 | Persistence strategy | `full-quarkus` + JPA detected | Panache active record / Panache repository / Hibernate ORM standard |
| 5 | REST framework | `full-quarkus` + web detected | Quarkus REST — RESTEasy Reactive (recommended) / RESTEasy Classic |
| 6 | Messaging transport | Messaging detected (any strategy) | `kafka` / `amqp` / `artemis-jms` |
| 7 | View technology | View layer detected (any strategy) | Qute (recommended) / MyFaces |
| 8 | Security approach | `full-quarkus` + Spring Security detected | OIDC / Basic / JWT / None |

If a Stage 2 condition is not met, the question is skipped silently and the
corresponding `migration_strategy` field is set to `none`.

### Selective detected feature flags

The planning module detects the following features during its source scan. This list
is **selective** — it covers features relevant to the current module set. It grows
as new modules are added; each new module declares the flag(s) it depends on.

| Flag | Detected when |
|---|---|
| `spring_web` | `@RestController`, `@Controller` in Java sources |
| `spring_data_jpa` | `JpaRepository`, `@Entity` in Java sources |
| `spring_security` | `SecurityConfig`, `@EnableWebSecurity` in Java sources |
| `spring_kafka` | `@KafkaListener`, `KafkaTemplate` in Java sources |
| `spring_rabbitmq` | `@RabbitListener`, `RabbitTemplate` in Java sources |
| `spring_jms` | `@JmsListener` in Java sources |
| `spring_scheduled` | `@Scheduled` in Java sources |
| `spring_cache` | `@Cacheable`, `@CacheEvict` in Java sources |
| `view_layer` | Thymeleaf/JSP/FreeMarker/JSF templates in `templates/` or `WEB-INF/` |

The full `detected_features` schema (including flags written by the `discovery` module
once that module is introduced) is defined in
[ADR-0003](0003-file-schemas-for-new-migration-modules.md).

### `migration-spec.yaml` as shared contract

`migration-spec.yaml` is written by `planning` and read by all downstream modules.
The full schema is defined in ADR-0003. The planning module's specific contribution is:

- `project.*` — source and target paths, artifact name
- `source_technology.*` — Spring Boot version, Java version, build tool
- `target_technology.*` — resolved Quarkus version, Java version, extensions list
- `detected_features.*` — selective boolean flags from the planning scan (superseded
  by the `discovery` module's richer scan once that module is introduced)
- `migration_strategy.*` — all user decisions with `strategy_source`
- `metadata.complexity` — `low` / `medium` / `high` based on component count
- `metadata.generatedAt` — ISO-8601 timestamp
- `decisions[]` — append-only log of every decision and its reason

**Complexity estimate** — based on total component count (controllers + services +
repositories + entities): `low` (< 10), `medium` (10–50), `high` (> 50).

### Changes to `SKILL.md`

- Step 1 **Analyze & Choose Strategy** is replaced by a delegation to
  `modules/planning/planning.md` — the inline scan and strategy question move into the
  module.
- The Decision Gate Table gains a `planning` **ALWAYS** row, positioned after
  `prerequisite` and before all transformation modules.
- The Execution Protocol `FOR module IN [...]` list is updated accordingly.

### How downstream modules use `migration-spec.yaml`

| Module | What it reads |
|---|---|
| `modules/build/` | `quarkus_version`, `java_version` — writes to `pom.xml` / `build.gradle` |
| `modules/code/code.md` | `migration_mode`, `persistence` — branches between strategies |
| `modules/frontend/frontend.md` | `view_layer` — chooses Qute vs. MyFaces path |
| Prerequisite module (#55) | `target_technology.java_version` — validates JDK minimum |
| Discovery module (#55) | Writes richer `detected_features` flags into the spec |
| Reporting module (#59) | `decisions[]`, `complexity`, feature flags — final report |

## Consequences

### Positive

- Decisions are asked once, recorded durably, and available to every module without
  re-scanning. Reduces both token consumption and non-determinism.
- The target directory contains a complete, human-readable record of every migration
  run.
- New modules can be added without touching `SKILL.md` — they declare the
  `detected_features` flags they depend on and read their decisions from the spec.
- The planning module's scan is a stepping stone: once `discovery` (#55) is
  introduced, its richer structured output replaces the planning-time scan, and
  planning becomes purely a decision-collection and spec-writing module.

### Negative

- Adds a mandatory module before all transformation work, even for trivial projects.
- The planning gate must be **ALWAYS** with no skip condition — if it is skipped or
  fails, all downstream modules lose their source of truth.
- `modules/build/build.md`, `maven.md`, and `gradle.md` must be updated to read
  version values from the spec rather than resolving them independently.
- The existing inline scan and strategy question in `SKILL.md` Step 1 must be removed
  to avoid duplication.
