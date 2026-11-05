# Architecture

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## Project map

```
settings.gradle.kts            # :app :domain :core :data :presentation
gradle/libs.versions.toml      # every version, SDK levels, bundles
app/                           # com.android.application — process wiring only
  └─ com/wigroup/debtnote/
     ├─ MainApplication.kt     # @HiltAndroidApp, Configuration.Provider (HiltWorkerFactory)
     ├─ MainActivity.kt        # single AppCompatActivity: splash, edge-to-edge, setContent { MainScreen() }
     ├─ MyFirebaseMessagingService.kt
     ├─ di/AppModule.kt        # binds SyncRepository, provides WorkManager
     ├─ platform/SyncRepositoryImpl.kt   # domain SyncRepository on top of WorkManager
     ├─ worker/SyncWorker.kt   # @HiltWorker two-way sync of all entities
     ├─ model/ mapper/         # app-local models (SyncState as Worker output JSON)
core/                          # android library, tiny: AppLanguage + currentLanguage()
domain/                        # kotlin("jvm") — no Android
  └─ com/wigroup/domain/
     ├─ entity/                # Contact, Debt, Note, User, Device (data classes, BigDecimal, Instant)
     ├─ error/                 # RootError + one sealed interface per area
     ├─ gateway/               # technical ports: TokenGateway, TransactionRunner, VoiceRepository
     ├─ repository/            # data ports: XxxRepository interfaces
     ├─ specification/         # CanAddXxxSpecification (Free/Pro quotas)
     ├─ usecase/<area>/        # XxxUseCase, operator fun invoke
     ├─ validation/            # XxxInputValidator + shared constants
     └─ value/                 # Result, AppSettings, AppConfig, enums, value classes
data/                          # android library — implements domain ports
  └─ com/wigroup/data/
     ├─ di/DataModule.kt       # @Binds every port, @Provides Room/Retrofit/OkHttp/DataStore
     ├─ local/database/        # AppDatabase, dao/, entity/, model/ (query tuples)
     ├─ local/datastore/       # Context.dataStore, UserDataStore
     ├─ mapper/                # Entity/DTO ↔ domain
     ├─ platform/              # CryptoManager, RoomTransactionRunner, TokenGatewayImpl, VoiceRepositoryImpl
     ├─ remote/                # ApiFactory, ApiService, interceptor/, request/, response/, Utils.safeApiCall
     └─ repository/            # XxxRepositoryImpl
presentation/                  # android library — Compose UI
  └─ com/wigroup/presentation/
     ├─ component/             # App* design-system components (AppButton, AppTextField, AppBar, DialogMessage …)
     ├─ feature/<area>/<screen>/   # Screen, ViewModel, State, Intent, Effect, Event
     ├─ mapper/CommonMapper.kt # domain → Ui
     ├─ model/                 # XxxUi (@Immutable), UI enums, ImmutableHolder
     ├─ navigation/            # Screen.kt (routes), NavGraph.kt, Extensions.kt, Animations.kt, Utils.kt
     ├─ ui/theme/              # Color, Theme (DebtNoteTheme), AppTheme (AppColorScheme), Spacing, Sizing, Type
     └─ util/                  # UiText, SnackbarController, CompositionLocals, formatters, small helpers
```

Code is grouped **by layer at module level and by feature inside `presentation/feature`**.
*Why:* [hypothesis] the domain is small and shared by many screens (a contact screen uses contact, debt, user and sync use
cases), so feature-modules would mostly re-export the same domain; the screen, however, is the unit of UI change, so its six
files live together. *Trade-off:* `:presentation` is one big module — incremental builds recompile all screens.
**[proposal]** split `:presentation` into `:feature:*` modules only when build time hurts; keep `:domain` single.

---

## Dependency direction

```
:app ──► :presentation, :data, :domain        (Hilt root must see every @Module)
:presentation ──► :domain, :core
:data ──► :domain, :core
:domain ──► nothing project-internal           (paging-common, javax.inject only)
:core ──► appcompat
```

`app/build.gradle.kts`:
```kotlin
// Modules
implementation(project(":domain"))
implementation(project(":data"))
implementation(project(":presentation"))
```

Forbidden:
- `:presentation` → `:data` (a screen must not see a DAO or DTO); `:data` → `:presentation`.
- `:domain` → any Android artifact. The only allowed external types are `PagingData` (paging-common, pure Kotlin),
  `kotlinx.coroutines.flow.Flow`, `javax.inject.Inject`, `java.math`, `java.time`.
- ViewModel → repository (always through a use case).
- `:app` code reading another module's storage directly (legacy: `MyFirebaseMessagingService` and `SyncWorker` edit
  DataStore keys `is_fcm_synced` / `installation_id` instead of going through `AppSettingsRepository.updateIsFcmSynced/
  updateInstallationId`, which already exist).

*Why `:core` exists:* [hypothesis] `AppLanguage` is needed both in `:data` (default currency symbol per language in
`AppSettingsRepositoryImpl`) and in `:presentation` (language picker), and needs `AppCompatDelegate`, so it cannot live in
`:domain`.

---

<a id="domain"></a>
## Domain

Everything a product owner would call "a rule" lives here.

| Package | What | Example |
|---|---|---|
| `entity` | Business objects with identity | `Contact(id, name, desc, balance: BigDecimal, createdAt: Instant, …, isSynced)` |
| `value` | Settings, filters, enums, value classes, `Result` | `ContactsFilter`, `AppTheme`, `FontScale`, `BlockingAppType` |
| `error` | `RootError` + `sealed interface XxxError : RootError` | `NoteError.Empty`, `LoginError.Network(error)` |
| `validation` | Shape and bounds of input | `NoteInputValidator.validate(text): NoteError?` |
| `specification` | Quotas / permissions that need state | `CanAddNoteSpecification.check(currentCount, isPro)` |
| `repository` | Data ports | `NoteRepository` |
| `gateway` | Technical ports | `TokenGateway`, `TransactionRunner`, `VoiceRepository` |
| `usecase/<area>` | One scenario per class | `CreateNoteUseCase`, `ObserveNotesPagedUseCase` |

### Use case
`domain/.../usecase/note/CreateNoteUseCase.kt`
```kotlin
class CreateNoteUseCase @Inject constructor(
    private val userRepository: UserRepository,
    private val noteRepository: NoteRepository,
    private val validator: NoteInputValidator,
    private val specification: CanAddNoteSpecification,
    private val syncRepository: SyncRepository,
) {
    data class Params(val text: String, val checked: Boolean)

    suspend operator fun invoke(params: Params): Result<Long, NoteError> {
        val cleanText = params.text.trim()

        validator.validate(cleanText)?.let { return Result.error(it) }

        val currentCount = noteRepository.getCount()
        val isPro = userRepository.observe().firstOrNull()?.isPro == true

        specification.check(currentCount = currentCount, isPro = isPro)?.let { return Result.error(it) }

        val noteId = noteRepository.create(text = cleanText, checked = params.checked)

        syncRepository.sync()

        return Result.success(noteId)
    }
}
```
(legacy detail: here `getCount()` + `create()` run without a transaction, so two fast saves can both pass the quota; the
template [CreateUseCase.kt.stub](../templates/optional/quota/CreateUseCase.kt.stub) puts them inside
`transactionRunner.withWriteTransaction`, as `CreateContactWithDebtUseCase` does.)

Rules:
- `@Inject constructor`, no `@Singleton` (stateless, cheap). Validators/specifications are injected, not `object`s,
  so a test can pass another instance. (`EmailValidator` is an `object` — a pure regex helper, allowed.)
- `operator fun invoke`; `suspend` for one-shot, non-suspend returning `Flow` for observation.
- Input normalisation is the use case's job: `trim()`, `?.trim()?.ifBlank { null }` for optional text,
  `take(MAX_…)` for values that come from the system (device name, picked contact name).
- When ≥2 writes or a count-then-write must be atomic, wrap them in `transactionRunner.withWriteTransaction { }` and return
  `Result` from inside (`return@withWriteTransaction Result.error(it)`) — `CreateContactWithDebtUseCase`.
  (legacy: `CreateDebtUseCase` reads `getCount()` **outside** the transaction — keep count and insert in one transaction.)
- Every write ends with `syncRepository.sync()` (enqueue only, returns immediately).
- Trivial pass-through use cases are still written (`ObserveContactUseCase`, `ChangeThemeUseCase`), one line each.
  *Trade-off:* many tiny classes; in exchange the presentation layer has one uniform dependency type.

### Validator vs Specification
- **Validator** — needs only the input: emptiness, length, regex, range. Returns the first error in `when` order.
- **Specification** — needs current state (counts, Pro flag). Called inside the transaction.
- Both expose their limits as `const val` so the UI can reuse them (`maxLength = NoteInputValidator.MAX_TEXT_LENGTH`).

### Ports: `repository` vs `gateway`
- `repository/XxxRepository` — a source of business data (Room/DataStore/API).
- `gateway/` — a technical capability with no business data: `TokenGateway` (secure storage), `TransactionRunner`
  (DB transaction without leaking Room into domain), `VoiceRepository` (speech recognizer). [hypothesis]
  (Name inconsistency: `VoiceRepository` sits in `gateway` — new technical ports are named `XxxGateway`.)

---

## Data

- `XxxRepositoryImpl @Inject constructor(...) : XxxRepository` — `internal`, bound `@[Binds Singleton]` in `DataModule`.
- Reads return Room `Flow`/`PagingSource` mapped with `toDomain()`; writes set `is_synced = 0` and `updated_at = now`.
- Remote calls only via `safeApiCall { apiService.xxx() }`.
- `TransactionRunner` is implemented by `RoomTransactionRunner` (`appDatabase.withWriteTransaction`), so domain can open a
  transaction and even `rollback(result)`.
→ [data-layer.md](data-layer.md)

## Presentation

- `MainScreen` (public, the only public symbol of the module) owns app-wide state and provides it via CompositionLocals.
- Screens are `internal` and know only their ViewModel and `processEvent`.
- `NavGraph` is the only place that maps Events to back-stack operations.
→ [ui.md](ui.md)

## App

- `MainApplication` — `@HiltAndroidApp` + `Configuration.Provider` with `HiltWorkerFactory` (WorkManager default
  initializer must be removed in the manifest for this to work **[hypothesis — manifest not present in the analysed copy]**).
- `MainActivity` extends `AppCompatActivity` (needed for `AppCompatDelegate.setApplicationLocales`) and only installs the
  splash screen, enables edge-to-edge, creates the notification channel, asks for `POST_NOTIFICATIONS`.
- `SyncRepositoryImpl` lives in `:app` because it needs `SyncWorker`, which needs every repository. [hypothesis]

---

## DI (Hilt)

`data/.../di/DataModule.kt`
```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal interface DataModule {
    @[Binds Singleton]
    fun bindNoteRepository(impl: NoteRepositoryImpl): NoteRepository

    companion object {
        @[Provides Singleton]
        fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase =
            Room.databaseBuilder<AppDatabase>(context, AppDatabase.DB_NAME).build()
    }
}
```
Rules:
- One `internal interface XxxModule` per module (`DataModule`, `AppModule`); `@Binds` for ports, `@Provides` in the
  `companion object` for third-party objects.
- Annotation-array syntax `@[Binds Singleton]` / `@[Provides Singleton]`.
- Everything is `SingletonComponent`; ViewModels are `@HiltViewModel`; Worker is `@HiltWorker` + `@AssistedInject`.
- Use cases, validators, specifications are unscoped.
- `@Inject` is `javax.inject.Inject` (legacy: `jakarta.inject.Inject` in `SyncRepositoryImpl`).

---

<a id="solid"></a>
## SOLID in this architecture **[confirmed]**

| Principle | How the code applies it | Example |
|---|---|---|
| **S** — one reason to change | one job per layer and per class: screen renders, ViewModel maps intents to state, use case runs one scenario, validator checks input, repository moves data, mapper converts | `CreateNoteUseCase` vs `NoteInputValidator` vs `NoteRepositoryImpl` |
| **O** — extend without editing callers | new failure = new `data object` in a sealed error; new screen = new route + `entry<…>`; components grow by parameters with defaults | `AppButton(progress = …, icon = …)` |
| **L** — any implementation can replace a port | every port has an in-memory fake that the use cases and ViewModels accept unchanged | `FakeTaskItemRepository` in `testFixtures` |
| **I** — small ports | a port per capability, not a "god" data source | `TokenGateway`, `TransactionRunner`, `SyncRepository`, one repository per entity |
| **D** — depend on abstractions | use cases and ViewModels depend on domain interfaces; Hilt binds `internal` implementations in `:data`/`:app` | `@[Binds Singleton] fun bindNoteRepository(impl: NoteRepositoryImpl): NoteRepository` |

Violations to watch for: a ViewModel doing formatting and validation (S), a `when (screen)` growing in a shared component (O),
a fake that throws for half of the interface (I — split the port), a ViewModel importing `:data` (D).

## Decision summary

| Decision | Why | Cost |
|---|---|---|
| Pure-JVM `:domain` | Rules testable without Android, no `Context` leaks | Mapping layers |
| Use case for every ViewModel call | One dependency type in presentation; rules have an obvious home | Many one-line classes |
| Validator + Specification split | Stateless checks reusable by UI; stateful ones run in a transaction | Two classes per entity |
| Repository interfaces in domain, `internal` impls | Presentation cannot reach storage | Bindings boilerplate |
| Single Activity + Navigation 3 | Typed routes, back stack is a plain list | Nav3 still evolving (custom `CustomNavDisplay`) |
| Offline-first + WorkManager sync | Works without network; sync survives process death | Sync protocol complexity |
| `:presentation` single module | Simple graph for ~20 screens | Slower incremental builds |
