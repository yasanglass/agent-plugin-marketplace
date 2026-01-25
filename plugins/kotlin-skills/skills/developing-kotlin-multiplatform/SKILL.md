---
name: Kotlin Multiplatform Development
description: Provides patterns for Kotlin Multiplatform (KMP) expect/actual declarations, source sets, Gradle tasks, and Kotlin/Native builds. Use when developing KMP projects, sharing code across Android, iOS, JVM, JS, and Wasm, or configuring multiplatform dependencies.
---

## File Naming
Use platform suffixes to distinguish files with the same name across platforms:
- `FileManager.kt` → common expect
- `FileManager.android.kt` → Android actual
- `FileManager.ios.kt` → iOS actual

## Expect/Actual Patterns

### Factory Function (for class instantiation in common code)
```kotlin
// Common
expect class Platform()
expect fun createPlatform(): Platform

// Android
actual class Platform actual constructor()
actual fun createPlatform(): Platform = Platform()

// iOS
actual class Platform actual constructor()
actual fun createPlatform(): Platform = Platform()
```

### Function
```kotlin
// Common
expect fun getPlatformName(): String

// Android
actual fun getPlatformName(): String = "Android ${Build.VERSION.SDK_INT}"

// iOS
actual fun getPlatformName(): String = UIDevice.currentDevice.systemName + " " + UIDevice.currentDevice.systemVersion
```

### Object (singleton)
```kotlin
// Common
expect object Logger {
    fun log(message: String)
}

// Android
actual object Logger {
    actual fun log(message: String) = Log.d("App", message)
}

// iOS
actual object Logger {
    actual fun log(message: String) = NSLog(message)
}
```

### Property
```kotlin
// Common
expect val isDebug: Boolean

// Android
actual val isDebug: Boolean = BuildConfig.DEBUG

// iOS
actual val isDebug: Boolean = Platform.isDebugBinary
```

### Typealias (wrap platform types)
```kotlin
// Common
expect class NativeDate

// Android
actual typealias NativeDate = java.util.Date

// iOS
actual typealias NativeDate = platform.Foundation.NSDate
```

### Interface + Factory (preferred for complex APIs)
```kotlin
// Common
interface DeviceInfo {
    val platformName: String
    val osVersion: String
}
expect fun createDeviceInfo(): DeviceInfo

// Android
class AndroidDeviceInfo : DeviceInfo {
    override val platformName = "Android"
    override val osVersion = Build.VERSION.RELEASE
}
actual fun createDeviceInfo(): DeviceInfo = AndroidDeviceInfo()

// iOS
class IosDeviceInfo : DeviceInfo {
    override val platformName = "iOS"
    override val osVersion = UIDevice.currentDevice.systemVersion
}
actual fun createDeviceInfo(): DeviceInfo = IosDeviceInfo()
```

### Choosing a Pattern

| Pattern | Use When |
|---------|----------|
| Interface + Factory | Default choice. Flexible, testable, multiple implementations per platform |
| Expect/actual fun | Simple factory functions or standalone platform-specific logic |
| Expect/actual object | Singletons with a fixed API |
| Expect/actual typealias | Wrapping existing platform types |
| Expect/actual class | Only when inheriting from platform-specific base classes |

### Common Mistakes

- Mismatched packages between `expect/actual` declarations
- Using `expect/actual class` when an interface would suffice

## Gradle Tasks
- Use target-specific tasks instead of `gradlew build`
- Run `./gradlew tasks` to list available tasks
- For iOS, prefer a single target (e.g., `iosSimulatorArm64`) unless all are needed

### Compilation
| Pattern | Examples |
|---------|----------|
| `compileKotlin<Target>` | `compileKotlinJvm`, `compileKotlinJs`, `compileKotlinIosArm64`, `compileKotlinIosSimulatorArm64`, `compileKotlinWasmJs` |
| `compile<BuildType>KotlinAndroid` | `compileDebugKotlinAndroid`, `compileReleaseKotlinAndroid` |
| `compile<SourceSet>KotlinMetadata` | `compileCommonMainKotlinMetadata`, `compileAppleMainKotlinMetadata`, `compileIosMainKotlinMetadata` |

### Linking (Kotlin/Native)
| Pattern | Examples |
|---------|----------|
| `link<BuildType>Framework<Target>` | `linkDebugFrameworkIosArm64`, `linkReleaseFrameworkIosSimulatorArm64`, `linkDebugFrameworkIosX64` |
| `linkDebugTest<Target>` | `linkDebugTestIosArm64`, `linkDebugTestIosSimulatorArm64` |

### Testing
| Pattern | Examples |
|---------|----------|
| `allTests` | `allTests` (runs all targets) |
| `<target>Test` | `iosSimulatorArm64Test`, `jvmTest`, `jsTest`, `jsBrowserTest`, `wasmJsTest` |
| `test<BuildType>UnitTest` | `testDebugUnitTest`, `testReleaseUnitTest` |
| `connected<BuildType>AndroidTest` | `connectedDebugAndroidTest` |

## Source Set Hierarchy
Exact source sets depend on your configured targets. The plugin automatically creates intermediate source sets based on declared targets.

```
commonMain
├── androidMain
├── jvmMain
├── webMain
│   ├── jsMain
│   └── wasmMain
│       ├── wasmJsMain
│       └── wasmWasiMain
└── nativeMain
    ├── appleMain
    │   ├── iosMain
    │   │   ├── iosArm64Main
    │   │   └── iosSimulatorArm64Main
    │   ├── macosMain
    │   │   ├── macosX64Main
    │   │   └── macosArm64Main
    │   ├── tvosMain
    │   └── watchosMain
    ├── linuxX64Main
    └── mingwX64Main
```

Intermediate source sets for shared logic:
- `webMain` → JS and Wasm targets (Kotlin 2.2.20+)
- `nativeMain` → all Kotlin/Native targets
- `appleMain` → iOS, macOS, watchOS, tvOS
- `iosMain` → all iOS targets (iosArm64, iosSimulatorArm64)

### Custom Source Sets
Create custom intermediate source sets with `dependsOn`:

```kotlin
kotlin {
    applyDefaultHierarchyTemplate()

    sourceSets {
        val jvmAndroidMain by creating {
            dependsOn(commonMain.get())
        }
        androidMain.get().dependsOn(jvmAndroidMain)
        jvmMain.get().dependsOn(jvmAndroidMain)
    }
}
```

## Dependencies
Add dependencies per source set. Dependencies in parent source sets propagate to children.

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("group:artifact:version")
            api("group:exposed-artifact:version")
        }

        // Platform-specific (artifact suffix varies by library)
        androidMain.dependencies {
            implementation("group:artifact-<android>:version")
        }
        iosMain.dependencies {
            implementation("group:artifact-<ios>:version")
        }

        // Custom source sets
        // - "by getting": reference existing source sets
        // - "by creating": create new custom source sets
        val androidMain by getting
        val jvmMain by getting

        val jvmAndroidMain by creating {
            dependsOn(commonMain.get())
        }
        jvmAndroidMain.dependencies {
            implementation("group:artifact-<jvm-android>:version")
        }
        androidMain.dependsOn(jvmAndroidMain)
        jvmMain.dependsOn(jvmAndroidMain)
    }
}
```
