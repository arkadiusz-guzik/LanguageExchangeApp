---
paths: 
    - "src/test/java/com/*"
---

# Testing Rules for Nodular Code

## Testing Philosophy
Tests in the modular architecture follow Hexagonal approach: test through the Facade using InMemory adapters for fast, infrastructure-free unit tests. Do not test internal classes directly.

## Test Nodule Structure
Tests mirror the main source structure under `src/test/java/com/*`:

```
modular/
    base/                                   # Shared test utilities (public classes)
        InMemoryJpaRepository.java          # Shared test utilities (public classes)
        InMemoryCache.java                  # Generic in-memory cache base class
        TestDateTimeProxy.java              # Deterministic DateTimeProxy for tests
        FieldOverrideBuilder.java           # Test data builder with field overrides

    {module}/domain/                        # Module-Level unit tests
        Test{Module}Facade.java             # sealed interface - wires real Facade with InMemory/mock deps
        Sample{Feature}.java                # non-sealed interface extends Test{Module}Facade - test data
        {Feature}Test. java                 # pkg-private test class implements Sample{Feature}
        InMemory{Entity}Repository.java     # pkg-private InMemory secondary adapter
        InMemory{ServiceName}.java          # pkg-private - InMemory infrastructure adapter
    infrastructure/{concern}/
        {Concern}Test.java  

    {module}/                               # Module-level integration tests
        {Feature}IntegrationTest.java       # Integration tests (outside domain/)

    shared/{concern}/                       # Shared kernel tests (direct unit tests, no Facade pattern)
        {Concern}Test.java                  

```

Two patterns exist depending on module complexity:

- **Complex modules** (e.g., wizard/benefit ): sealed interface hierarchy (`TestFacade` -> `Sample` -> `*Test` )
- **Simple modules** (e.g., location'): plain interface (`SampleLocation` wires Facade + data, `TestLocation` implements `SampleLocation`)


## Test Facade Pattern (sealed interface hierarchy)

The primary test pattern uses a three-level sealed interface hierarchy:

1. **`sealed interface TestBenefitWizardFacade`** - top-level sealed interface that wires the real Facade via the Configuration class:
    - Uses `InMemory*` implementations for infrastructure (repositories, cache, S3)
    - Uses Mockito mocks for cross-module dependencies (other modules' Facades/Providers)
    - The `permits` clause lists all `Sample*` interfaces

2. **`non-sealed interface Sample{Feature}Benefits extends TestBenefitWizardFacade`** - provides test fixture data for a specific feature area

3. **`class {Feature}Test implements Sample{Feature}-Benefits`** - actual JUnit test class that gets the wired Facade and test data via the interface hierarchy

```java
// Level 1: sealed, wires the Facade
sealed interface TestBenefitWizardFacade permits SampleBudgetBenefits, SampleDetailsBenefits, ...{
    BenefitCacheProvider benefitCacheProvider = new InMemoryBenefitCache();
    UUIDProxy mockUuidProxy = mock(UUIDProxy.class);
    BenefitWizardFacade benefitWizardFacade = new Benefit₩izardConfiguration()
        .benefitWizardFacade(benefitCacheProvider, mockUuidProxy, ...);
}

// Level 2: non-sealed, provides test data
non-sealed interface SampleBudgetBenefits extends TestBenefitWizardFacade { 
// test fixture constants and helper methods
}

// Level 3: test class
@Tag("unit")
class BenefitBudgetTest implements SampleBudgetBenefits {
    @Test void someTest() { /* uses benefitWizardFacade from the hierarchy */}
}
```

Note: Simpler modules (e.g., location') may Use a non-sealed interface pattern without the full sealed hierarchy.

## Base Test Utilities (modular/base/)

- `InMemoryJpaRepository<T, ID>` - generic in-memory JPA repository for testing
- `InMemoryCache<T, ID>` - generic in-memory cache for testing
- `TestDateTimeProxy` - deterministic `DateTimeProxy` implementation for tests
- `FieldOverrideBuilder` - utility for constructing test data with field overrides via `field()` and `applyOverrides()`


## InMemory Adapters

- Extend `InMemoryJpaRepository` or `InMemoryCache` from the `base` test package (base classes are `public`; module-specific adapters must be **package-private**)
- `InMemoryJpaRepository` constructor takes an ID getter (`Function<T, ID>`) and ID setter (`BiConsumer<T, ID>`)
- Implement the same interface as the production adapter
- Named `InMemory{Entity}Repository` or `InMemory{ServiceName}`


## Test Conventions

- Tag unit tests with `@Tag("unit")`
- Use JUnit 5 (`@Test`, `@ParameterizedTest` with `@MethodSource` from `org.junit.jupiter.api`)
- Follow given/when/then structure in test methods (use `//given`, `//when`, `//then` comments)
- Test at the Facade level (behavioral tests through the module's public API); avoid testing internal classes directly
- Use `TestDateTimeProxy` (from `base/`) for deterministic time in tests
- Use `mock(UUIDProxy.class)` with `when(...). thenReturn(...)` for deterministic UUIDS
- Integration tests exist separately (e.g., `RegionLanguageRepositoryIntegrationTest`) and infrastructure tests (e.g., `BenefitCacheProviderTest`)