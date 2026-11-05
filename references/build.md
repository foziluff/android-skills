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

## Convention plugins (`build-logic`) **[confirmed: DRY]**

Shared module configuration is written once, in an included build:

```
settings.gradle.kts            pluginManagement { includeBuild("build-logic") … }
build-logic/
  settings.gradle.kts          versionCatalogs { create("libs") { from(files("../gradle/libs.versions.toml")) } }
  convention/build.gradle.kts  `kotlin-dsl`; compileOnly AGP, KGP, detekt plugins; registers the plugin ids
  convention/src/main/kotlin/
    ProjectExtensions.kt       libs accessor, configureAndroid (compileSdk/minSdk), configureLint, configureKotlin (+detekt)
    AndroidApplicationConventionPlugin.kt   {appName}.android.application  (+ targetSdk)
    AndroidLibraryConventionPlugin.kt       {appName}.android.library      (+ test runner)
    AndroidComposeConventionPlugin.kt       {appName}.android.compose      (compose plugin, buildFeatures, BOM + UI deps)
    JvmLibraryConventionPlugin.kt           {appName}.jvm.library          (kotlin-jvm, java-test-fixtures)
    HiltConventionPlugin.kt                 {appName}.hilt                 (ksp, hilt plugin, hilt-android + compiler)
```

A module script then contains only what is unique to it:
```kotlin
plugins {
    alias(libs.plugins.{appName}.android.library)
    alias(libs.plugins.{appName}.android.compose)
    alias(libs.plugins.{appName}.hilt)
}

android {
    namespace = "{basePackage}.presentation"
}

dependencies { … }
```
- Plugin ids are declared in the catalog without a version (`{appName}-android-library = { id = "{appName}.android.library" }`).
- AGP 9: `compileSdk {}` and `defaultConfig` are on `CommonExtension`; `lint {}` only on `ApplicationExtension` /
  `LibraryExtension` — hence `configureLint` is called from each plugin.
- (legacy: the reference project repeats the `android {}` / `kotlin {}` blocks and Hilt/KSP lines in every module.)

| Module | Plugins (skeleton) |
|---|---|
| `:app` | `{appName}.android.application`, `.android.compose`, `.hilt` (+ kotlin.serialization with sync) |
| `:presentation` | `{appName}.android.library`, `.android.compose`, `.hilt`, kotlin.serialization (routes) |
| `:data` | `{appName}.android.library`, `.hilt`, kotlin.serialization |
| `:core` | `{appName}.android.library` |
| `:domain` | `{appName}.jvm.library` |

Namespaces: `{basePackage}.{appName}` (app), `{basePackage}.{module}` for libraries.

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

## Static analysis and raw-value checks

| Tool | Config | What it guards |
|---|---|---|
| detekt `2.0.0-alpha.6` (`dev.detekt`, applied by every convention plugin) | `config/detekt/detekt.yml` on top of the default config | `MagicNumber`, `StringLiteralDuplication`, `UnsafeCallOnNullableType` (`!!`), `WildcardImport`, `MaxLineLength 120`, `NewLineAtEndOfFile`, `MatchingDeclarationName`, complexity — Compose functions exempt from naming/length rules |
| Android Lint | `config/lint/lint.xml`, `abortOnError = true` | texts (`HardcodedText`, `SetTextI18n`, `MissingTranslation`, `ExtraTranslation`, `StringFormat*`), `UnusedResources`, exported components, implicit intents |
| `checkRawValues` (root task) | regexes in the root `build.gradle.kts` | `Text("…")`, `N.dp`, `N.sp`, `Color(0x…)` outside `ui/theme` — what Lint and detekt do not see in Compose code |

detekt 2 is still an alpha line (the 1.23 line does not support Gradle 9 / Kotlin 2.4) **[hypothesis]**; pin the version and
move to the first stable 2.x. Config property names changed in 2.x (`allowedDuplications`, `allowedWithLengthLessThan`).
The reference project has only Lint with default settings.

## Commands

```bash
./gradlew detekt lintDebug checkRawValues      # style, texts, raw values
./gradlew :domain:test testDebugUnitTest       # unit tests
./gradlew assembleDebug
./gradlew assembleRelease                      # verifies R8; run separately from assembleDebug on a clean schema dir
./gradlew :data:kspDebugKotlin                 # quick check of Room/Hilt code generation
```

<a id="choosing-a-library"></a>
## Choosing a library **[confirmed]**

Order: **platform / Kotlin stdlib → AndroidX (Jetpack) → official Google or JetBrains library → third-party** (only with a
reason written next to it in `libs.versions.toml`).

| Need | Use (official) | Not | In skeleton / reference |
|---|---|---|---|
| UI | Jetpack Compose + Material 3, `material-icons-extended` | third-party UI kits | ✅ |
| Navigation | Navigation 3 (`androidx.navigation3`) | Voyager, Decompose, Compose Destinations | ✅ |
| DI | Hilt (Dagger, Google), `androidx.hilt` | Koin, Kodein | ✅ |
| Async | kotlinx.coroutines + Flow | RxJava | ✅ |
| JSON | kotlinx.serialization | Gson, Moshi, Jackson | ✅ |
| Database | Room | Realm, SQLDelight, ObjectBox | ✅ |
| Key-value settings | DataStore | MMKV, Paper, plain SharedPreferences | ✅ |
| Secrets | AndroidKeyStore (`javax.crypto`) + DataStore | third-party "secure prefs" | ✅ |
| Lists from DB | Paging 3 | custom paginators | ✅ |
| Background work | WorkManager | custom services, Evernote job | ✅ (optional/sync) |
| Sign-in | Credential Manager + Google ID | deprecated Google Sign-In SDK | reference ✅ |
| Biometrics | `androidx.biometric` | third-party wrappers | reference ✅ |
| Splash | `androidx.core:core-splashscreen` | custom splash Activity | ✅ |
| Date / time | `java.time` (minSdk ≥ 26) | Joda-Time, ThreeTenABP | ✅ |
| Push / analytics / crashes | Firebase (Messaging, Analytics, Crashlytics) | Sentry, OneSignal, AppCenter | reference: Messaging + Analytics |
| HTTP | **Retrofit + OkHttp (Square) — third-party exception** **[confirmed]**: the author keeps Retrofit over the official Ktor client | Ktor client, Volley, Fuel | ✅ |
| Images | platform (`ImageDecoder`, `BitmapFactory`) while needs are simple | — | reference ✅; for remote image lists there is no official loader → Coil, with a reason **[question]** |
| Tests | JUnit 4 + `androidx.test`, `kotlinx-coroutines-test`, Compose UI test | MockK, Mockito, Turbine (fakes + `first()` instead) | ✅ |
| Static analysis | Android Lint | — | ✅ |
| Kotlin static analysis | **detekt — third-party dev-only exception** (no official equivalent for `MagicNumber` / duplication) | ktlint, Spotless | ✅ (not shipped in the APK) |
| Logging | none in release; if needed — `android.util.Log` behind `BuildConfig.DEBUG` | Timber | reference: no logging |

(legacy in the reference: `mXparser` for the calculator — third-party, kept because there is no official expression parser.)

## Adding a dependency

1. Walk the order above (philosophy #6): platform → AndroidX → official Google/JetBrains → third-party. For a third-party
   library write the reason as the `# Group` comment in the catalog (`# Retrofit — no official REST client with converters`).
2. Add version + library (+ bundle if grouped) to the catalog with a `# Group` comment.
3. Add to the **lowest** module that needs it (`implementation`, never `api`).
4. Check R8 consumer rules exist; build `assembleRelease`.
