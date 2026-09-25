# Module: Planning

Scan the source project, collect migration decisions, and generate `<target>/migration-spec.yaml` as the binding contract for all downstream modules.

## Preconditions

- Prerequisite checks (JDK version check) have passed.
- Source and target directory paths are known.

## Gate Condition

**ALWAYS** — runs before all transformation modules.

---

## Instructions

Follow the 4 steps below in sequence:

### Step 1: Scan Source Project & Detect Features

1. **Build Descriptor**:
   - Maven (`pom.xml`) or Gradle (`build.gradle` / `build.gradle.kts`).
   - Identify: `spring_boot_version`, source `java_version`, `build_tool` (`Maven` or `Gradle`).
2. **Java Sources**:
   - Inspect annotations, imports, and classes to evaluate selective feature flags:

| Flag | Detected when |
|---|---|
| `spring_web` | `@RestController`, `@Controller` in Java sources |
| `spring_data_jpa` | `JpaRepository`, `@Entity` in Java sources |
| `spring_security` | `SecurityConfig`, `@EnableWebSecurity`, `@PreAuthorize`, `SecurityFilterChain` in Java sources |
| `spring_kafka` | `@KafkaListener`, `KafkaTemplate` in Java sources |
| `spring_rabbitmq` | `@RabbitListener`, `RabbitTemplate` in Java sources |
| `spring_jms` | `@JmsListener`, `JmsTemplate` in Java sources |
| `spring_scheduled` | `@Scheduled` in Java sources |
| `spring_cache` | `@Cacheable`, `@CacheEvict` in Java sources |
| `view_layer` | Thymeleaf/JSP/FreeMarker/JSF templates in `templates/` or `WEB-INF/` |

3. **Complexity Estimation**:
   - Count total components across controllers, services, repositories, and entities:
     - `low`: < 10 components
     - `medium`: 10–50 components
     - `high`: > 50 components

---

### Step 2: Present Findings Summary

Display a findings table to the user:

```markdown
### Source Project Analysis Summary
- **Build Tool**: [Maven | Gradle]
- **Spring Boot Version**: [e.g., 3.2.0]
- **Source Java Version**: [e.g., 17]
- **Estimated Complexity**: [low | medium | high]

| Area | Detected Technology / Annotations | Status |
|---|---|---|
| Web / REST | `@RestController`, `@Controller` | [Detected / Not found] |
| Data / Persistence | `@Entity`, `JpaRepository` | [Detected / Not found] |
| Security | `SecurityConfig`, `@EnableWebSecurity` | [Detected / Not found] |
| Messaging | Kafka / RabbitMQ / JMS | [Detected (type) / Not found] |
| Scheduling | `@Scheduled` | [Detected / Not found] |
| Caching | `@Cacheable` | [Detected / Not found] |
| View Layer | Thymeleaf / JSP / FreeMarker / JSF | [Detected / Not found] |
| Tests | `@SpringBootTest`, `@WebMvcTest` | [Detected / Not found] |
```

---

### Step 3: Collect Decisions

Decisions are collected in two stages.

**Resolution Priority for any decision:**
1. **Skill Argument** (`strategy_source: argument`)
2. **Project config file** (`.quarkus-migration.yml` in source root, if present; `strategy_source: config-file`)
3. **Ask user (Interactive mode)** (`strategy_source: user`)
4. **Auto-select defaults (Non-interactive mode)** (`strategy_source: default`)

#### Stage 1: Core Decisions (Always Collect)

Ask Stage 1 questions first (in interactive mode), or apply defaults:

Both Quarkus version and Java version are resolved at runtime by calling the [`code.quarkus.io/api/streams`](https://code.quarkus.io/api/streams) API (see [#71](https://github.com/quarkusio/skills/issues/71)), which returns available Quarkus streams with their minimum JDK version.

| # | Decision | Interactive Option / Prompt | Non-interactive default |
|---|---|---|---|
| 1 | Target Quarkus version | Ask: offer API-resolved latest stable or let user specify | Resolved from `code.quarkus.io/api/streams` |
| 2 | Target Java version | Ask: offer minimum JDK required by the resolved Quarkus version or let user specify | Minimum JDK required by the resolved Quarkus version (today: JDK 17 for Quarkus 3.x, JDK 21 for Quarkus 4.x) |
| 3 | Migration strategy | Ask: `full-quarkus` (idiomatic JAX-RS/CDI/Panache) or `spring-compat` (Quarkus Spring compatibility extensions) | `full-quarkus` |

> *In interactive mode, stop and wait for the user's response to Stage 1 before presenting Stage 2 questions.*

#### Stage 2: Conditional Decisions (Based on Stage 1 and Detected Features)

If applicable based on detected features, collect conditional decisions:

| # | Decision | Condition | Options | Non-interactive default |
|---|---|---|---|---|
| 4 | Persistence strategy | `full-quarkus` + `spring_data_jpa` detected | `panache-active-record` / `panache-repository` / `hibernate-orm` | `panache-active-record` |
| 5 | REST framework | `full-quarkus` + `spring_web` detected | `quarkus-rest` (RESTEasy Reactive, recommended) / `resteasy-classic` | `quarkus-rest` |
| 6 | Messaging transport | Messaging detected (`spring_kafka`, `spring_rabbitmq`, `spring_jms`) | `kafka` / `amqp` / `artemis-jms` (matching detected transport) | Matching detected transport |
| 7 | View technology | `view_layer` detected | `qute` (recommended) / `myfaces` | `qute` |
| 8 | Security approach | `full-quarkus` + `spring_security` detected | `oidc` / `basic` / `jwt` / `none` (recorded for future use) | `none` |

*Note: If a Stage 2 condition is not met, skip the question and set the field to `none` or omit as appropriate.*

---

### Step 4: Write `migration-spec.yaml`

Write the specification to `<target>/migration-spec.yaml`.

```yaml
project:
  name: <source-name>-quarkus
  source_path: <source_directory_path>
  target_path: <target_directory_path>

source_technology:
  spring_boot_version: "<detected-version>"
  java_version: "<detected-java-version>"
  build_tool: "<Maven|Gradle>"

target_technology:
  quarkus_version: "<resolved-quarkus-version>"
  java_version: "<target-java-version>"
  build_tool: "<Maven|Gradle>"

detected_features:
  spring_web: true|false
  spring_data_jpa: true|false
  spring_security: true|false
  spring_kafka: true|false
  spring_rabbitmq: true|false
  spring_jms: true|false
  spring_scheduled: true|false
  spring_cache: true|false
  view_layer: true|false

decisions:
  strategy: "full-quarkus|spring-compat"
  strategy_source: "argument|config-file|user|default"
  persistence: "panache-active-record|panache-repository|hibernate-orm|none"
  rest_framework: "quarkus-rest|resteasy-classic|none"
  messaging_transport: "kafka|amqp|artemis-jms|none"
  view_layer: "qute|myfaces|none"
  security_approach: "oidc|basic|jwt|none"

metadata:
  complexity: "low|medium|high"
  generatedAt: "<ISO-8601-timestamp>"

decision_log:
  - decision: "Target Quarkus version <version>"
    reason: "Resolved from <source>"
  - decision: "Migration strategy: <strategy>"
    reason: "Selected via <source>"
  # Append each decision made
```
