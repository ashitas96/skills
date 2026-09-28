# Module: Check JDK Version

Verify that the installed JDK meets the minimum version requirement before proceeding with the migration. This module runs before planning — `migration-spec.yaml` does not exist yet — so the required version is resolved from available inputs using the priority order below.

## Preconditions

This module has no preconditions — it must **always** run as the very first step.

## Instructions

- **DO NOT** skip this module.

### Step 1: Resolve the required JDK version

Use the first source that provides a value (highest priority first):

| Priority | Source | How to read it |
|---|---|---|
| 1 | Skill argument | `java_version` passed directly when invoking the skill |
| 2 | `.quarkus-migration.yml` | `java_version` field in `<source>/.quarkus-migration.yml` (if the file exists) |
| 3 | Safe default | **17** — the absolute minimum required by any supported Quarkus version (Quarkus 3.x requires JDK 17; Quarkus 4.x requires JDK 21) |

Call the resolved value `<required_jdk>`.

### Step 2: Check the installed JDK

- [ ] Run `java -version` and capture the installed version.
- [ ] If the installed version is **>= `<required_jdk>`**, mark this module as passed and proceed to the planning module.
- [ ] If the installed version is **< `<required_jdk>`** or `java` is not found:
    - **Warn the user**: "JDK `<required_jdk>` or later is required for this migration (resolved from: `<source>`). Currently installed: `<detected version or 'none'>`. Please install JDK `<required_jdk>` and ensure it is on your PATH before retrying."
    - **Stop the migration** — do not proceed to any subsequent module.