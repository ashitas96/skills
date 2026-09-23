# Module: Build System

Migrate the build descriptor and configuration files from Spring Boot to Quarkus.

This is the first module to run. It creates `<target>` and populates it with the source project files before transforming the build configuration.

## Instructions

### Create the target project

Before any transformation, copy the entire source project into the target directory:

1. Create `<target>` if it does not exist.
2. Copy all files from `<source>` into `<target>`, preserving the directory structure. This includes `src/`, resources, build files, wrapper scripts, and any other project files.
3. From this point on, all modifications happen in `<target>`. Do not modify `<source>`.

### Detect and migrate the build system

- Detect the build tool by checking which files exist at `<source>`:

| File | Build tool | Sub-module |
|---|---|---|
| `pom.xml` | Maven | [maven.md](maven.md) |
| `build.gradle` or `build.gradle.kts` | Gradle | [gradle.md](gradle.md) |

- Load [references/dependency-map.md](../../references/dependency-map.md) and [references/config-map.md](../../references/config-map.md) before starting.
- Then load and execute the matching submodule above. All build file modifications happen in `<target>`.
- After the submodule completes, return here and continue with the Configuration Migration and Watch Out sections below.

## Configuration Migration

Rename Spring properties to Quarkus equivalents in `<target>` using config-map.md. Key mappings:

- `spring.datasource.*` → `%prod.quarkus.datasource.*` (see below)
- `spring.jpa.*` → `quarkus.hibernate-orm.*`
- `server.port` → `quarkus.http.port`
- `logging.level.*` → `quarkus.log.category."*".level`

### Datasource properties and Dev Services

When the project has no `application-{profile}.properties` files, prefix datasource connection properties with `%prod.` so they only apply in production. This lets Quarkus Dev Services automatically start a containerized database in dev and test modes — no local database setup needed.

```properties
# connection details only for prod
%prod.quarkus.datasource.jdbc.url=jdbc:mysql://127.0.0.1:3306/todo
%prod.quarkus.datasource.username=root
%prod.quarkus.datasource.password=root
```

If `application-{profile}.properties` files exist, place production datasource config in `application-prod.properties` instead (no `%prod.` prefix needed there).

## Watch out

- **Profile handling**: Spring's `application-{profile}.properties` → Quarkus `%profile.` prefix in a single `application.properties`
- **Naming strategy mismatch**: Spring Boot defaults to snake_case (`firstName` → `first_name`). Quarkus/Hibernate 6 preserves camelCase. Set `quarkus.hibernate-orm.physical-naming-strategy=org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy`. **Also update `import.sql`/`data.sql` column names**.
- **`quarkus-spring-boot-properties`** (Spring compat only): `@ConstructorBinding` NOT supported (needs no-arg constructor + setters). `Map<K,V>` types NOT supported.
- **Build tool wrapper**: If the project has `mvnw`/`gradlew`, always use `./mvnw` or `./gradlew` instead of the system-installed `mvn` or `gradle` command. This ensures reproducible builds with the exact tool version the project expects.