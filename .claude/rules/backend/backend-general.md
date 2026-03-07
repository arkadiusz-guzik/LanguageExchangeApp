---
paths: 
    - "src/main/java/com/*"
---

# Project Rules

See `.claude/rules/backend/` for all project rules.

## Tech Stack

- Java 21, Spring Boot 4.1.0-M2, Maven
- Lombok for boilerplate reduction
- MapStruct for DTO/entity mapping uses manual Factory classes
- JOOQ for query-side database access (CQRS reads)
- Spring Data JPA for command-side persistence
- OpenAPI Generator for REST controller interfaces (controllers implement generated `*Api` interfaces)
- PostgreSQL (JOOQ generation, Flyway migrations) as database
- Flyway for database migrations
- AWS S3 for file storage
- Spring Security with `@PreAuthorize` for endpoint authorization

## Development Iteration & Verification Workflow

When implementing a new feature, refactoring, or modifying existing behavior:
1. Implement the feature strictly according to:
    - Module boundaries
    - DDD rules
    - CQRS separation
    - Package discipline
2. Self-review the implementation and explicitly verify:
    Including:
    1. If a change requires modification outside the declared scope:
       - STOP
       - Explicitly document why
       - Do not proceed silently
    2. Hexagonal Architecture Compliance
       - No business logic leaked into controllers
       - No domain → infrastructure dependency
       - No infrastructure package accessed from domain
       - No entity exposed outside module
       - No repository exposed outside module
       - No Facade bypassing
       - No illegal visibility changes (public added without rule justification)
       - Access modifiers strictly follow the Access Modifiers Rules table
    3. CQRS Separation
       - No JOOQ used in command flow
       - No JPA used in query flow (except documented legacy cross-boundary pattern)
       - Query DTOs are not reused as command DTOs
       - Query side contains zero business logic
    4. Bean & Wiring Discipline
       - No @Service or @Component on Facades
       - No public *Configuration classes
       - Internal collaborators instantiated with new inside Configuration
       - No field injection
       - Constructor injection only via @RequiredArgsConstructor
    5. Domain Integrity
       - Aggregates enforce invariants
       - No anemic domain model
       - Factories create aggregates (no external new)
       - Domain does not depend on Spring
       - Value objects used instead of primitives where meaningful
    6. Transaction & Security
       - Transactions defined at Facade level only
       - No @Transactional on repositories
       - All endpoints have explicit @PreAuthorize
       - No security logic inside domain
    7. Package Visibility Audit
       - Everything package-private by default
       - Only allowed public types:
          - *Facade
          - DTOs in dto/
          - *MetadataProvider interfaces
       - No accidental public on:
          - Aggregates
          - Factories
          - Entities
          - Controllers
          - Configuration classes
3. Compare the implementation against Project Rules and check for:
    - Architectural violations
    - Package visibility leaks (public where not allowed)
    - Missing invariants in aggregates
    - Anemic domain model symptoms
    - Cross-module coupling
    - Improper transaction boundaries
    - Missing tests (unit + integration)
4. Fix every violation found.
5. Re-review the updated implementation.
6. Repeat steps 2–5 at least twice, even if no issues are found in the first pass.
7. Only stop when:
    - No architectural violations remain
    - All rules in this document are satisfied
        - The implementation fully respects DDD + Hexagonal + CQRS boundaries

**Do NOT stop after a single review pass.**
**Always perform at least two structured validation rounds.**


## Build & Verify
- Build: `mvn clean install`
- Run tests: `mvn test` (runs unit tests by default via surefire groups=unit)
- Run single test: `mvn test -Dtest=ClassName`
- Run all tests including integration: `mvn test -Dgroups=unit,integration`


## Project Structure
Each module follows a consistent package structure:

```
{module}/
  domain/             # Business logic: Facade (public API), Aggregate, Factory, Entities Repositories
    {subdomain}/      # Optional nested sub-domains with own Facade + Configuration
    infrastructure/   # Adapters: Controller, cache, S3, exception handlers, config properties
  dto/                # Public DTOs shared across module boundaries
  query/              # CQRS read-side: JOOQ query repositories + query DTOs
  metadata/           # Reference/lookup data: entities and provider interfaces
```
- `domain/` is the hexagonal core -- everything here is **package-private** except the Facade and types exposed in its signatures (interfaces, exceptions)
- `domain/infrastructure/` holds adapters (controllers, external service clients) -- they depend on the domain, never the reverse
- `dto/` and `metadata/` sit outside `domain/` because they cross module boundaries


## Key modules under modular/:
- `shared/` - Shared kernel (value objects, proxies, cache, secrets)

## General Conventions
- Use Lombok (`@RequiredArgsConstructor`, `@Getter`, `@Slf4j`, etc.) instead of manual boilerplate
- Never use field injection (`@Autowired` on fields); use constructor injection via `@RequiredArgsConstructor`
