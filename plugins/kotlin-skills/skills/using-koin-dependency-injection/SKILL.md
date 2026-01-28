---
name: Koin Dependency Injection
description: Patterns for Koin dependency injection modules. Use when defining Koin modules, singletons, factories, or bindings in Kotlin projects.
---

## Module Definition Patterns

Use constructor reference DSL (`singleOf`, `factoryOf`, `viewModelOf`) with optional `bind<Interface>()`:

```kotlin
val featureModule: Module = module {
    singleOf(::DataStoreProvider)
    singleOf(::RepositoryImpl) {
        bind<Repository>()
    }
    factoryOf(::Mapper)
    viewModelOf(::FeatureViewModel)
}
```

Avoid lambda definitions with get() when constructor references suffice:

```kotlin
// Avoid
single { Repository(get(), get(), get()) }
```

## Annotations

NEVER use Koin annotations (`@Single`, `@Factory`, `@KoinViewModel`, `@Module`, etc.).

**Exception**: `@InjectedParam` is permitted on constructor parameters that are injected at runtime (not from the Koin graph). This enables Koin module verification tests to pass.

## Complex Object Creation

NEVER put complex object creation logic directly in module definitions.

Use a factory function:

```kotlin
// FlavorFactory.kt
fun createFlavor(): Flavor = Flavor(
    flavor = BuildConfig.FLAVOR,
    buildType = BuildConfig.BUILD_TYPE,
    versionCode = BuildConfig.VERSION_CODE.toLong(),
    versionName = BuildConfig.VERSION_NAME,
    gitBranch = BuildConfig.GIT_BRANCH_NAME,
    gitCommit = BuildConfig.GIT_COMMIT_HASH,
)

// Module
val featureModule: Module = module {
    single<Flavor> { createFlavor() }
}
```
