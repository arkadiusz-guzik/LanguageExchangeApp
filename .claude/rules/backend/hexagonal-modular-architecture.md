---
paths: 
    - "src/main/java/com/*"
---

# Hexagonal Architecture Rules

This project follows hexagonal architecture approach. Each module is a self-contained hexagon with strict encapsulation via Java's package-private visibility.

## Module Structure
Every module under `modular/` followers this package layout:

```
modular/{module}/
    domain/                                         # Core domain logic 
        {Module}Facade.java                         # PUBLIC - the module's API (primary port)
        {Module}Configuration.java                  # pkg-private - Spring @Configuration, wires beans
        {Module}Aggregate.java                      # pkg-private - DDD aggregate root (if needed)
        {Module}Factory. java                       # pkg-private - creates domain objects
        {Entity}Entity.java                         # pkg-private - JPA entities
        {Entity}Repository.java                     # pkg-private - secondary port (Spring Data)
        {subdomain}/                                # Optional nested sub-domain hexagons (own Facade + Config)
        infrastructure/                             # Adapters (controllers, external integrations)
            {Module}ControllerImpl.java             # pkg-private - REST controller (primary adapter)
            {Module}MetadataProvider.java           # PUBLIC interface - secondary port for cross-module use
            exception/                              # Module-specific exceptions
            cache/                                  # Cache adapters
            s3/                                     # S3 file storage adapters
            properties/                             # Spring Boot @ConfigurationProperties
            sanitizer/                              # File scanning/sanitization adapters
    dto/                                            # PUBLIC - DTOs for inter-module and API communication
    query/                                          # CORS read-side (j00Q-based query repositories)
    metadata/                                       # Reference data entities and providers
```

### Existing Modules



## Access Modifiers Rules (CRITICAL - DO NOT BREAK)
Package-private is the default. Only make classes public when other modules need them.

| Component                    | Access              | Rationale                                              |
|------------------------------|---------------------|--------------------------------------------------------|
| `*Facade`                    | `public`            | Module's API - the only entry point for other modules  |
| `*Dto` in `dto/` package     | `public`            | Shared across module boundaries                        |
| `*MetadataProvider` interface| `public`            | Cross-module contract in `infrastructure/`             |
| `*Configuration`             | **package-private** | Only Spring needs to see it                            |
| `xControllerImpl`            | **package-private** | Primary adapter - hidden from other modules            |
| `*Aggregate`                 | **package-private** | Internal domain logic                                  |
| `*Factory`                   | **package-private** | Internal wiring                                        |
| `*Entity`                    | **package-private** | Persistence detail                                     |
| `*Repository`                | **package-private** | Secondary port used only within the module             |

## Facade Pattern

Each module exposes a `*Facade` class as its public API:
- The Facade orchestrates use cases by delegating to aggregates and factories
- Other modules depend only on the Facade (or its `*MetadataProvider` interface)
- The Facade is instantiated by the `*Configuration` class, not via `@Service` or `@Component`
- Use `@RequiredArgsConstructor` for constructor injection, never `@Autowired`
- Modules may have sub-domain Facades (e.g., `BenefitBudgetFacade`) wired by their own Configuration class

```java
@RequiredArgsConstructor
public class ExampleFacade
    private final SomeDependency dependency;
    // public methods = the module's use cases
```

## Configuration Class Pattern

Each module/sub-domain has a `*Configuration` class that manually wires beans:
- Marked with `@Configuration`
- Must be **package-private** (no `public` keyword)
- Creates the Facade as a `@Bean`, injecting infrastructure dependencies
- Internal collaborators (factories, resolvers) are instantiated with `new` not as beans


```java
@Configuration
class ExampleConfiguration {
    @Bean
    ExampleFacade exampleFacade(SomeRepository repo) {
        ExampleFactory factory = new ExampleFactory(); // internal, not a bean
        return new ExampleFacade(factory, repo);
    }
}
```

## Controller Pattern (Primary Adapter)

Controllers live in `domain/infrastructure/`:
- Must be **package-private**
- Annotated with `@RestController` and `@RequiredArgsConstructor`
- Implement the OpenAPI-generated `Api` interface (from `org.openapitools.api`)
- Delegate all business logic to the Facade; controllers only handle HTTP concerns

## Inter-Module Communication

Modules communicate through:
1. **Facade injection**: Module A's Configuration injects Module B's Facade as a bean dependency
2. **MetadataProvider interfaces**: Defined in `domain/infrastructure/`, implemented by the Facade itself (e.g., `LocationFacade implements LocationMetadataProvider`)
3. **Shared value objects**: From `modular/shared/value0bject/` (e.g., `CodeName<C>`)

## CORS Pattern

The query side is separated from the command side:
- Command side: Facade -> Aggregate -> Repository (Spring Data JPA)
- Query side: `query/` package with dedicated query DTOs. Most query repositories Use `DSLContext` (e.g. `BenefitWizardQueryRepositoryImpl`), but some use Spring Data JPA `@Query` projections (e.g., `LinkOptionTypeQueryRepository`)
- Query configurations follow the same package-private `*Configuration` pattern
- Note: `BenefitEntityRepository` (command-side) extends `BenefitWizardQueryRepository` (query-side interface) -- this is an existing cross-boundary pattern

## Domain Objects

- Aggregates use interfaces to expose role-based views (e.g. `BenefitWizardAggregate implements BenefitDetail, BenefitInit, BenefitBudget, BenefitContent, BenefitSummary`)
- Domain sub-objects (Headline, Details, Content, Budget) are interfaces with implementations (e.g., `HeadlineImpl`, `DetailsImpl`). They provide static `init()` factory methods and instance `update()` methods
- Sub-objects Live in sub-packages: `domain/headline/`, `domain/details/`, `domain/content/`, `domain/budget/`
- Enums for domain constants live in the relevant domain sub-package
- Use `CodeName<C>` value object for code-name pairs throughout the domain


## Shared Kernel ("modular/shared/")

Contains cross-cutting concerns used by all modules:
- `valueObject/CodeName<C>` - immutable generic code-name pair
- `proxy/DateTimeProxy, proxy/UUIDProxy` - testable wrapper interfaces for time (`OffsetDateTime.now()`, `LocalDate.now()`) and `UUID.randomUUID()`
- `proxy/DateTimeProxyImpl, 'proxy/UUIDProxyImpl` - production implementations
- `cache/CacheProvider` - cache abstraction interface
- `secrets/SecretManager` - secrets management abstraction with `VaultSecretManagerImpl` and `ContainerSecretManagerImpl`
