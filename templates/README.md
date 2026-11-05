# Templates

Three sets:

| Folder | What | Verified |
|---|---|---|
| `project/` | Skeleton of a new app: Gradle (5 modules) + `build-logic` convention plugins, version catalog, `config/detekt`, `config/lint`, `checkRawValues` task, manifest, resources, `Result`/`NetworkError`, `UiText`, `SnackbarController`, theme + tokens, `App*` components (incl. `AppBottomButton`), Navigation 3 graph, `MainScreen`/`MainViewModel`, settings in DataStore, Room + `TransactionRunner`, Retrofit + `safeApiCall`, Keystore `CryptoManager` + `TokenGateway` | ✅ generated into an empty folder with other names: `assembleDebug`, `assembleRelease` (R8), `detekt`, `lintDebug`, `checkRawValues` — zero findings, unit tests green |
| `feature/` | One entity with Add / Edit / List screens: domain (entity, error, validator, repository, 5 use cases), data (Room entity, FTS, DAO, mapper, repository), presentation (Ui, shared `{Entity}Form`, `{Entity}ErrorMapper`, 3 screens × State/Intent/Effect/Event/ViewModel/Screen), tests + `testFixtures` fake | ✅ same build and checks; 9 unit tests (validator, use case, ViewModel) |
| `optional/` | `sync/` — offline two-way sync with WorkManager; `quota/` — Free/Pro limits | sync ✅ applied to the generated app: build, detekt, Lint, `checkRawValues`, tests (pull-to-refresh UI wiring is a snippet) / quota — not compiled (needs the app's source of the paid flag) |

Every `.kt.stub` / `.kts.stub` / `.xml.stub` / `.toml.stub` starts with **one comment line holding the target path** —
create the file there and delete that line. `*.snippet.stub` files are fragments to paste into existing files.
`// DECISION:` comments mark choices to make; delete them afterwards (production code has almost no comments).

## Placeholders

| Placeholder | Example | Used for |
|---|---|---|
| `{basePackage}` / `{basePackagePath}` | `com.example` / `com/example` | packages, source paths; modules are `{basePackage}.domain/.data/.presentation/.core` |
| `{appName}` | `notebook` | app module package `{basePackage}.{appName}`, `applicationId`, DB file name, Keystore alias |
| `{App}` | `Notebook` | `rootProject.name`, `{App}Theme`, `Theme.{App}` styles, `app_name` |
| `{Entity}` / `{entity}` | `Memo` / `memo` | classes / variables |
| `{Entities}` / `{entities}` | `Memos` / `memos` | list screen, use cases, lists |
| `{entities_table}` | `memos` | Room table, FTS table, string keys (snake_case plural) |
| `{entity_snake}` | `memo` | string keys, `contentType`, tombstone table `{entity_snake}_deletes` |
| `{entities-url}` | `memos` | API path segment (kebab-case) — sync only |
| `{area}` | `memos` | `domain/usecase/{area}`, `presentation/feature/{area}/add|edit|list` |

<a id="a-new-project"></a>
## A new project

1. Pick the placeholders. Copy every file of `project/` **and** `feature/` to its target path, replacing placeholders.
   The skeleton already wires the first feature (`DataModule`, `AppDatabase`, `CommonMapper`, `Screen.kt`, `NavGraph`,
   strings) — an app always starts with at least one entity, and Room needs ≥1 entity to compile.
2. Gradle wrapper and daemon JVM (not templated, generated):
   ```bash
   gradle wrapper --gradle-version <latest stable>
   ./gradlew updateDaemonJvm --jvm-version=<jdkVersion>
   ```
   `local.properties` with `sdk.dir=…` (never committed).
3. In `gradle/libs.versions.toml` the versions are the **baseline the skeleton was verified with**; at project start move
   them to the latest stable releases together (BoMs first), then build.
   `minorApiLevel` must exist in the installed SDK platform (`android-37.0` → `"0"`).
4. Decide: real `BASE_URL`s in `data/build.gradle.kts` (debug/release), launcher icon in the manifest, palette in
   `ui/theme/Color.kt` + `Theme.kt`, locales (`localeFilters`, `resources.properties`, `values-*/strings.xml`,
   `AppLanguage`), backup policy (`backup_rules.xml`, `data_extraction_rules.xml`).
5. Build and check:
   ```bash
   ./gradlew assembleDebug
   ./gradlew detekt lintDebug checkRawValues
   ./gradlew :domain:test testDebugUnitTest
   ./gradlew assembleRelease
   ```
   Run `assembleDebug` and `assembleRelease` as separate invocations the first time: with `exportSchema = true` both variants
   write `data/schemas/…/1.json`, and in one run they can race.
6. First commit — one logical change, no traces of AI.

Not in the skeleton on purpose (add only when the product needs it): Firebase/FCM, Credential Manager / Google sign-in,
biometric lock, WorkManager sync, PDF export, analytics.

<a id="a-new-feature"></a>
## A new feature (in an existing project)

| # | File | Template |
|---|---|---|
| 1 | `domain/entity/{Entity}.kt` | [feature/domain/Entity.kt.stub](feature/domain/Entity.kt.stub) |
| 2 | `domain/error/{Entity}Error.kt` | [feature/domain/EntityError.kt.stub](feature/domain/EntityError.kt.stub) |
| 3 | `domain/validation/{Entity}InputValidator.kt` | [feature/domain/EntityInputValidator.kt.stub](feature/domain/EntityInputValidator.kt.stub) |
| 4 | `domain/repository/{Entity}Repository.kt` | [feature/domain/EntityRepository.kt.stub](feature/domain/EntityRepository.kt.stub) |
| 5 | `domain/usecase/{area}/…UseCase.kt` | [Create](feature/domain/CreateEntityUseCase.kt.stub), [Update](feature/domain/UpdateEntityUseCase.kt.stub), [Delete](feature/domain/DeleteEntitiesUseCase.kt.stub), [Observe](feature/domain/ObserveEntityUseCase.kt.stub), [ObservePaged](feature/domain/ObserveEntitiesPagedUseCase.kt.stub) |
| 6 | `data/local/database/entity/{Entity}Entity.kt`, `{Entity}FtsEntity.kt` (if searchable) | [Entity](feature/data/EntityEntity.kt.stub), [Fts](feature/data/EntityFtsEntity.kt.stub) |
| 7 | `data/local/database/dao/{Entity}Dao.kt` | [feature/data/EntityDao.kt.stub](feature/data/EntityDao.kt.stub) |
| 8 | `data/mapper/{Entity}Mapper.kt` | [feature/data/EntityMapper.kt.stub](feature/data/EntityMapper.kt.stub) |
| 9 | `data/repository/{Entity}RepositoryImpl.kt` | [feature/data/EntityRepositoryImpl.kt.stub](feature/data/EntityRepositoryImpl.kt.stub) |
| 10 | Wiring: `@Binds` + `provide{Entity}Dao` in `DataModule`, entities + `abstract fun {entity}Dao()` in `AppDatabase` (**version + 1 and a `Migration`**), `toUi()` in `CommonMapper` | see the `{Entity}` lines in [project/data/DataModule.kt.stub](project/data/DataModule.kt.stub), [project/data/AppDatabase.kt.stub](project/data/AppDatabase.kt.stub), [project/presentation/CommonMapper.kt.stub](project/presentation/CommonMapper.kt.stub) |
| 11 | `presentation/model/{Entity}Ui.kt`, `presentation/mapper/{Entity}ErrorMapper.kt` | [EntityUi](feature/presentation/EntityUi.kt.stub), [EntityErrorMapper](feature/presentation/EntityErrorMapper.kt.stub) |
| 11a | `presentation/feature/{area}/component/{Entity}Form.kt` — the fields shared by Add and Edit | [feature/presentation/EntityForm.kt.stub](feature/presentation/EntityForm.kt.stub) |
| 12 | `presentation/feature/{area}/add/…` (6 files) | [feature/presentation/add/](feature/presentation/add) |
| 13 | `presentation/feature/{area}/edit/…` (6 files, assisted `{entity}Id`) | [feature/presentation/edit/](feature/presentation/edit) |
| 14 | `presentation/feature/{area}/list/…` (5 files) | [feature/presentation/list/](feature/presentation/list) |
| 15 | Routes in `Screen.kt` + `entry<…>` in `NavGraph.kt` | the `{Entity}` parts of [project/presentation/navigation/Screen.kt.stub](project/presentation/navigation/Screen.kt.stub), [NavGraph.kt.stub](project/presentation/navigation/NavGraph.kt.stub) |
| 16 | Strings in **every** `values*/strings.xml` | the `{entity_snake}` / `{entities_table}` keys of [project/presentation/res/strings.xml.stub](project/presentation/res/strings.xml.stub) |
| 17 | Tests | [feature/test/](feature/test) (`domain.Fake*` → `domain/src/testFixtures`, other `domain.*` → `domain/src/test`, `presentation.*` → `presentation/src/test`, `data.*` → `data/src/androidTest`) |
| 18 | DRY pass | anything the new feature repeats from another one (a row, a dialog, a mapper) → extract now ([references/conventions.md](../references/conventions.md#dry-and-raw-values)) |

A second top-level list (tab) = another `TopLevelScreen` object; the `NavBar` component is not in the skeleton (reference:
`presentation/component/NavBar.kt` in DebtNote).

<a id="optional-capabilities"></a>
## Optional capabilities

### Offline two-way sync — [optional/sync/](optional/sync)
1. Infrastructure once per app: `SyncRepository`, `SyncState`, `SyncUseCase` + `ObserveSyncStateUseCase` (domain);
   `SyncWorker`, `model/SyncState`, `platform/SyncRepositoryImpl`, `di/AppModule`, `MainApplication` with `HiltWorkerFactory`
   (app); catalog/build/manifest lines from `build.snippet.stub`; `ApiFactory.LIMIT_COUNT`; `FakeSyncRepository` in
   `domain/src/testFixtures` (pass it to use cases in tests).
2. Per entity: replace the base `Entity`, `Dao`, `Mapper`, `RepositoryImpl` with the sync versions; add `DeleteEntity`,
   `RequestDto`, `ResponseDto`, `SyncRequest`, `SyncResponse`, the `ApiService` endpoint; follow `wiring.snippet.stub`
   (domain entity `isSynced`, repository `clear()`/`sync()`, `syncRepository.sync()` after every write, pull-to-refresh,
   a job in `SyncWorker`).
3. Server contract the client expects: `POST sync/{entities-url}` with `{items, deleted_ids, confirmed_deleted_ids,
   client_update_seq}` → `{items, deleted_ids}`; rows carry `server_id`, `local_id`, `update_seq`, `local_created_at`,
   `local_updated_at` (see [references/data-layer.md](../references/data-layer.md#sync)).

### Free/Pro quotas — [optional/quota/](optional/quota)
`Specification.kt.stub`, the transactional `CreateUseCase.kt.stub`, `wiring.snippet.stub` (error cases, State/Intent/Event,
dialogs, strings), `SpecificationTest.kt.stub`, fakes. Needs a source of the paid flag; the reference reads
`UserRepository.observe().firstOrNull()?.isPro` — replace with the app's entitlement source.

### Other capabilities seen in the reference project (no templates — copy from DebtNote with the rules of this skill)
Authentication with a Bearer token (`AuthRepository`, `LoginUseCase`, `AuthInterceptor` is already in the skeleton), Google
sign-in via Credential Manager, app lock with `BiometricPrompt`, FCM push, PDF export, voice input (`callbackFlow` over
`SpeechRecognizer`). Where they live and what to avoid: [references/](../references/).
