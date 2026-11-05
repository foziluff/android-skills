# Build: Gradle, version catalog, SDK levels, R8, locales, signing

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## Toolchain (baseline the skeleton was verified with — move to the latest stable at project start)

| Item | Version / value | Where |
|---|---|---|
| Gradle | 9.7.1 (wrapper, `distributionSha256Sum` pinned) | `gradle/wrapper/gradle-wrapper.properties` |
| AGP | 9.3.2 | `libs.versions.toml: agp` |
| Kotlin | 2.4.10 (+ Compose compiler plugin of the same version) | `kotlin` |
| KSP | 2.3.10 | `ksp` |
| JDK | 25 via `jvmToolchain(libs.versions.jdkVersion)` + foojay resolver | `settings.gradle.kts`, `gradle/gradle-daemon-jvm.properties` |
| compileSdk | 37 (reference: minor API level 1; skeleton: 0 — must exist in the installed SDK) | `compileSdk`, `minorApiLevel` |
| targetSdk | 37 | `targetSdk` |
| minSdk | 28 | `minSdk` |
| NDK | reference pins `ndkVersion` in every module; the skeleton drops it (no native code) | `ndkVersion` |
| Compose BOM | 2026.08.00 | `composeBom` |

Rules:
- **Every** version, including SDK levels and JDK, lives in `gradle/libs.versions.toml`. The `# Variables` block holds the
  non-library values (`jdkVersion`, `compileSdk`, `targetSdk`, `minorApiLevel`, `minSdk`, `ndkVersion`).
- Module scripts read them: `minSdk = libs.versions.minSdk.get().toInt()`.
- New compileSdk syntax (AGP 9):
  ```kotlin
  compileSdk {
      version = release(libs.versions.compileSdk.get().toInt()) {
          minorApiLevel = libs.versions.minorApiLevel.get().toInt()
      }
  }
  ```
- *Why minSdk 28:* [hypothesis] no legacy Keystore/biometric code paths, `NotificationChannel` always available; the only
  remaining version checks are Q (foreground service type) and TIRAMISU (`POST_NOTIFICATIONS`, `PackageInfoFlags`).

## Version catalog

```toml
[versions]
# Room3
room_version = "3.0.2"

[libraries]
# Room3
room3-runtime = { module = "androidx.room3:room3-runtime", version.ref = "room_version" }

[plugins]
# KSP
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }

[bundles]
room3 = ["room3-runtime", "room3-paging"]
```
- A `# Group` comment above each group in every section.
- Libraries that are always used together form a bundle: `navigation3`, `room3`, `hilt`, `paging`, `retrofit`,
  `credentials`, `work`, `firebase`.
- BoMs (`compose-bom`, `firebase-bom`) via `platform(...)`; BoM-managed libraries have no `version.ref`.
- Plugins are declared `apply false` in the root `build.gradle.kts` and applied with `alias(...)` in modules.
- Repositories: `pluginManagement.google { content { includeGroupByRegex(...) } }`; `RepositoriesMode.FAIL_ON_PROJECT_REPOS`
  — no module may declare its own repository.

## Module scripts

| Module | Plugins |
|---|---|
| `:app` | android.application, kotlin.compose, ksp, hilt (+ kotlin.serialization for the sync worker, google-services only with Firebase) |
| `:presentation` | android.library, kotlin.compose, ksp, kotlin.serialization (routes), hilt |
| `:data` | android.library, ksp, hilt, kotlin.serialization |
| `:core` | android.library |
| `:domain` | kotlin.jvm |

Each Android module repeats the same `android { ndkVersion; namespace; compileSdk {…}; defaultConfig { minSdk;
testInstrumentationRunner } }` and `kotlin { jvmToolchain(...) }` blocks. **[proposal]** a `build-logic` included build with
convention plugins (`debtnote.android.library`, `debtnote.android.compose`, `debtnote.hilt`) once a sixth module appears —
until then the duplication is small and explicit.

Namespaces: `com.wigroup.debtnote` (app), `com.wigroup.{module}` for libraries.

## Build types, R8, keep rules

```kotlin
buildTypes {
    release {
        optimization {
            enable = true          // AGP 9 DSL: R8 shrink + optimize + obfuscate
        }
    }
}
```
- Keep rules live in `src/main/keepRules/rules.keep` of each module (AGP 9 convention); all are currently empty — rely on
  library consumer rules (kotlinx.serialization, Retrofit, Room, Hilt).
- No product flavors. **[proposal]** if a staging backend appears, prefer a `staging` build type with a different
  `buildConfigField("String", "BASE_URL", …)` over flavors.
- Signing: not configured in Gradle files. **[question]** release signing is done in CI / Android Studio? Never add
  `storePassword` to build files; read from environment or `local.properties`.

## Locales

```kotlin
androidResources {
    generateLocaleConfig = true
    localeFilters += setOf("ru", "en", "tg", "uz", "b+uz+Cyrl")
}
bundle {
    language {
        enableSplit = false
    }
}
```
- `generateLocaleConfig` + `AppCompatDelegate.setApplicationLocales(...)` = per-app language picker (Android 13+ system
  settings and in-app).
- `enableSplit = false` — all languages ship in the base APK so in-app switching works without Play downloading splits.
  [hypothesis]
- `res/resources.properties: unqualifiedResLocale=ru` — default `values/` is Russian.

## Gradle properties

```
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
org.gradle.configuration-cache=true
kotlin.code.style=official
org.gradle.warning.mode=all
```
`warning.mode=all` — deprecation warnings are visible on every build; new code must not add warnings.

## Static analysis

- Configured: Android Lint (AGP default), Kotlin compiler warnings.
- Not configured: ktlint, detekt, spotless, CI. **[proposal]** add only after agreeing with the author; minimal option is
  `./gradlew lintDebug` + `testDebugUnitTest` in CI.

## Commands

```bash
./gradlew assembleDebug
./gradlew lintDebug
./gradlew testDebugUnitTest :domain:test
./gradlew assembleRelease          # verifies R8
./gradlew :data:kspDebugKotlin     # quick check of Room/Hilt code generation
```

## Adding a dependency

1. First check the platform/Jetpack (philosophy #6). Write down why it is needed.
2. Add version + library (+ bundle if grouped) to the catalog with a `# Group` comment.
3. Add to the **lowest** module that needs it (`implementation`, never `api`).
4. Check R8 consumer rules exist; build `assembleRelease`.
