# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository boundaries

- Work only inside the current Git repository.
- Use the built-in Read, Grep, and Glob tools for file inspection.
- Do not use PowerShell or Bash to read or search files when built-in tools can do it.
- Never inspect parent directories, the user home directory, or the filesystem root.
- If information outside the repository is required, stop and ask first.

## Project overview

Spring PetClinic: a Spring Boot 4 / Java 17 sample web app (Thymeleaf server-rendered UI) for
managing owners, pets, vets, and visits. Buildable with either Maven (`pom.xml`, primary/canonical)
or Gradle (`build.gradle`).

## Common commands

Use the Maven wrapper (`./mvnw` / `mvnw.cmd`) unless the user asks for Gradle.

```bash
# Run the app (default H2 in-memory DB) — http://localhost:8080
./mvnw spring-boot:run

# Full build (compiles, runs tests, checkstyle, spring-javaformat validation, jacoco)
./mvnw verify

# Run the whole test suite
./mvnw test

# Run a single test class
./mvnw test -Dtest=OwnerControllerTests

# Run a single test method
./mvnw test -Dtest=OwnerControllerTests#testInitCreationForm

# Auto-format code to satisfy spring-javaformat (run before committing Java changes)
./mvnw spring-javaformat:apply

# Rebuild petclinic.css from petclinic.scss (only needed after editing the scss)
./mvnw package -P css
```

Gradle equivalents: `./gradlew bootRun`, `./gradlew build`, `./gradlew test --tests OwnerControllerTests`.

The build enforces `io.spring.javaformat` (Spring's formatter) and Checkstyle
(`src/checkstyle/nohttp-checkstyle.xml`, forbidding raw `http://` URLs) as part of the `validate`
phase — a normal `mvn compile`/`test` will fail on formatting or nohttp violations before any code runs.

## Architecture

Single Spring Boot module under `org.springframework.samples.petclinic`, split into vertical
feature packages rather than technical layers:

- `owner/` — the core domain: `Owner`, `Pet`, `PetType`, `Visit` entities plus their Spring MVC
  controllers (`OwnerController`, `PetController`, `VisitController`) and Spring Data JPA
  repositories (`OwnerRepository`, `PetTypeRepository`). `Owner` eagerly loads its `Pet`s
  (`@OneToMany(fetch = EAGER)`), and each `Pet` eagerly loads its `Visit`s — there is no lazy
  loading to worry about in this package. `PetValidator` is a hand-written `Validator`
  (not Bean Validation annotations) for pet form rules; pet names must be unique per owner
  (case-insensitive) — see `Owner.getPet(String, boolean)`.
- `vet/` — read-mostly `Vet`/`Specialty` domain plus `VetController`/`VetRepository`. The vets
  list is cached (see `system/CacheConfiguration`, cache name `"vets"`); code that adds/edits
  vets must account for cache invalidation.
- `model/` — shared JPA base classes: `BaseEntity` (id), `NamedEntity` (+ name), `Person`
  (+ first/last name), all `@MappedSuperclass`.
- `system/` — cross-cutting config: `CacheConfiguration` (JCache/Caffeine), `WebConfiguration`,
  `CrashController` (deliberately throws, used to exercise the error page), `WelcomeController`.
- `PetClinicApplication` — the `@SpringBootApplication` entry point.
  `PetClinicRuntimeHints` registers GraalVM native-image hints (the project supports native
  builds via `org.graalvm.buildtools:native-maven-plugin`).

Data access is pure Spring Data JPA (no custom SQL beyond schema/seed scripts). Table naming
uses snake_case physical naming strategy, so `PetType` → `pet_types`, etc.

### Persistence profiles

Default profile uses H2 in-memory, schema/data auto-loaded from `src/main/resources/db/h2/`.
MySQL and Postgres variants live in `src/main/resources/db/{mysql,postgres}/` and are activated
via `spring.profiles.active=mysql|postgres` plus `application-{mysql,postgres}.properties`.
`docker-compose.yml` provides matching local containers (`docker compose up mysql|postgres`).
Don't assume H2-only behavior when touching schema or queries — check the other two dialects'
`schema.sql`/`data.sql` too.

For fast dev-loop testing, run the `main()` methods in `PetClinicIntegrationTests` (H2 +
devtools), `MysqlTestApplication` (Testcontainers-backed), and `PostgresIntegrationTests`
(Docker Compose-backed) directly from the IDE rather than only via `mvn test`.

### Internationalization

Message bundles live in `src/main/resources/messages/messages_*.properties`, one per locale
(base `messages.properties` is the English fallback; `messages_en.properties` is intentionally
not required to be populated — see the exclusion in `I18nPropertiesSyncTest`). Any new
user-facing string added to a Thymeleaf template or the base bundle must be added to every
locale file: `I18nPropertiesSyncTest` (in `system/`) fails the build if a key is missing from
any translation file, and separately fails on hard-coded (non-`th:text`/`#{}`) literal text in
`.html` files under `src/main`.

### Frontend

Thymeleaf templates in `src/main/resources/templates/{owners,pets,vets,fragments}`. CSS is
generated, not hand-edited directly: `petclinic.scss` (`src/main/scss/`) compiles to
`static/resources/css/petclinic.css` via the Maven `css` profile (libsass), pulling in the
webjars-vendored Bootstrap. Edit the `.scss`, then run `./mvnw package -P css` to regenerate.

## Testing conventions

- `@WebMvcTest` for controller tests (e.g. `OwnerControllerTests`, `PetControllerTests`) —
  mock the repository layer, assert view names/model attributes/redirects.
- `service/ClinicServiceTests` uses `@DataJpaTest` against the real H2 schema for repository
  behavior.
- `PetClinicIntegrationTests`, `MySqlIntegrationTests`, `PostgresIntegrationTests` are full
  `@SpringBootTest` integration tests, one per supported database; the MySQL/Postgres ones need
  Docker (Testcontainers / Docker Compose) available to run.
- `PetClinicConcurrencyTests` exercises concurrent update scenarios (relevant to the
  duplicate-pet-name-per-owner validation logic).
