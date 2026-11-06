---
name: foziluff-android-style
description: Use when starting, writing, refactoring or reviewing any Android app in foziluff's style on Kotlin / Jetpack Compose (Material 3, Navigation 3, Hilt, Room, Paging 3, DataStore, Retrofit + kotlinx.serialization, optional WorkManager sync) — Gradle modules app / presentation / domain / data / core, pure-Kotlin domain with UseCase + Validator (+ Specification for quotas), MVI screens (State / Intent / Effect / Event), local storage as the source of truth for screens, typed Result<T, RootError> errors mapped to UiText, @Immutable state, design tokens instead of raw values, Keystore-encrypted secrets, minimum third-party dependencies; includes a verified project skeleton and feature templates.
---

# foziluff Android style

This skill describes not only *how* the code is written but *why*, for **any** new or existing Android app.
The rules were extracted from the reference project **DebtNote** (an offline notebook with sync and a Pro plan); file paths in
the text point to it as evidence (`com.wigroup.*`). In another project replace the base package and names — the templates
use placeholders (`{basePackage}`, `{App}`, `{Entity}`).

**Core** (every app): modules, layers, MVI, `Result` errors, `UiText`, DRY / no raw values, tokens, components, navigation,
DI, data conventions, security, performance, tests, git. **Optional** (only if the product has it), marked *(if the app …)*:
offline sync, Free/Pro quotas, token auth, app lock, per-app languages — templates in `templates/optional/`.

Markers used in the text:
- **[confirmed]** — the author confirmed the decision personally;
- **[hypothesis]** — the reason is inferred from the code, not confirmed by the author;
- **[proposal]** — recommended target that does not exist in the project yet;
- **legacy** — this is how the project currently does it; do not write new code this way.

---

## Philosophy

1. **Every layer has exactly one job, and Gradle enforces it.** **[hypothesis]**
   `:domain` is a pure Kotlin JVM module (no Android SDK) with entities, errors, repository *interfaces*, validators,
   specifications and use cases. `:data` implements those interfaces (`internal`). `:presentation` maps domain to UI and
   renders. `:app` only wires the process (Application, Activity, Worker, FCM).
   *Why:* `:domain` physically cannot import `Context`, Room or Compose, so business rules stay testable and portable;
   `internal` makes it impossible for a screen to reach a DAO. *Cost:* three models per entity (Entity → Domain → Ui) and mappers.
   → [references/architecture.md](references/architecture.md)

2. **Screens observe local storage; the network only feeds it.** **[hypothesis]**
   Lists and details observe Room (`Flow`, `PagingSource`) or DataStore; a write goes to local storage and the screen
   updates by itself. Purely online data (e.g. a device list) is loaded through a repository into State with an explicit
   loading/error state. *(if the app syncs)* writes are marked `is_synced = 0` and the use case calls
   `SyncRepository.sync()`, which enqueues one unique WorkManager job. *Why:* the UI never waits for the server and keeps
   working offline. *Cost:* with sync — a protocol (local/server ids, tombstones, `update_seq`, batches).
   → [references/data-layer.md](references/data-layer.md)

3. **UI is a function of one immutable State; everything that happens is a typed message.** **[hypothesis]**
   `Intent` (UI → ViewModel), `State` (ViewModel → UI, `StateFlow`), `Effect` (ViewModel → UI, one-shot), `Event`
   (Screen → NavGraph, navigation). One `processIntent()` entry point, exhaustive `when`. *Why:* every screen reads the same
   way, behaviour is replayable and testable, recomposition is driven by data only. *Cost:* 6 files per screen.
   → [references/ui.md](references/ui.md)

4. **An error is a type, not an exception.** **[hypothesis]**
   `Result<T, E : RootError>`; one `sealed interface XxxError` per area; exceptions are converted at the boundary
   (`safeApiCall`) and never cross a layer; human text appears only in presentation via `UiText`.
   *Why:* the compiler lists every failure the screen must handle; domain stays language-free. *Cost:* verbose `when` branches.
   → [references/error-handling.md](references/error-handling.md)

5. **Maximum DRY and SOLID: every change is made in exactly one place.** **[confirmed]**
   **Second use → extract**, however small: color → color role, size → token, text → string resource, number → `const val`,
   UI piece → component (even a padded button — `AppBottomButton`, two fields — `{Entity}Form`), logic → function / mapper /
   use case, build config → convention plugin, fake → `testFixtures`. **No raw values** in logic or screens — literal design
   values live only in `ui/theme/*`, texts only in `strings.xml`. **SOLID** is how the layers are cut
   ([architecture.md](references/architecture.md#solid)).
   *Why:* when a brand color, padding, limit, text or button changes, one file is edited and every screen follows — nobody
   searches the project. *Cost:* more small files; components must stay generic. Enforced by detekt, Lint and
   `checkRawValues`. → [conventions.md](references/conventions.md#dry-and-raw-values)

6. **Minimum dependencies; when one is needed — the official one.** **[confirmed]**
   Order of choice: Android platform / Kotlin stdlib → **AndroidX / Jetpack** → other **official** libraries of Google
   (Dagger/Hilt, Material, Firebase, Play Services) and JetBrains (`kotlinx.*`, Ktor) → only then a third-party library,
   with a written reason. Hence kotlinx.serialization (not Gson/Moshi), Hilt (not Koin), Room (not Realm/SQLDelight),
   DataStore + Keystore (not third-party secure prefs), Navigation 3, coroutines/Flow (not RxJava), Paging 3, WorkManager,
   Credential Manager, `java.time` (not Joda/ThreeTen), Crashlytics if crash reports are needed, Android Lint first.
   *Why:* official libraries follow the platform's lifecycle, get security fixes and keep working with the next AGP/Kotlin/
   Compose; one vendor's docs; fewer surprises in R8. *Trade-off:* sometimes less convenient APIs, own code to maintain
   (`CryptoManager`, `PdfGenerator`, sync loop). Current third-party exceptions: Retrofit/OkHttp (Square, **[confirmed]** over Ktor) and detekt
   (dev tool) — see [build.md](references/build.md#choosing-a-library).

7. **Secure by default.** **[hypothesis]** The auth token is encrypted with an AndroidKeyStore AES-GCM key before it touches
   disk; there are no logs at all; SQL parameters are always bound; input is validated in domain before any write or request;
   the FileProvider exposes one cache sub-folder; the app can be locked with biometrics.
   → [references/security.md](references/security.md)

8. **Performance by construction, not by profiling later.** **[hypothesis]** `@Immutable` on every State and Ui model,
   `ImmutableHolder` for foreign types, `key` + `contentType` in every lazy list, Paging straight from Room, strings formatted
   once in the mapper (`balanceString`), cached formatters, `Lazy<UseCase>` for actions.
   → [references/performance.md](references/performance.md)

---

## Architecture in one paragraph

`MainActivity` (the only Activity, `AppCompatActivity` for per-app locales) calls `setContent { MainScreen() }`.
`MainScreen` collects `MainViewModel.state`, provides app-wide values through `CompositionLocal`s (always `LocalAppSettings`;
plus `LocalAuthState`, `LocalUser`, `LocalSyncState`, … when the app has them), applies `{App}Theme` and hosts the
Navigation 3 `NavGraph`. Each screen gets `processEvent` (navigation out) and a `hiltViewModel()`; route arguments reach the
ViewModel through an `@AssistedFactory`. The ViewModel calls use cases; use cases trim input, run a validator (and a
specification), write through a repository interface (inside `TransactionRunner` when needed). Repositories read
Room/DataStore and call Retrofit only through `safeApiCall`. *(if the app syncs)* a `SyncWorker` does two-way sync in the
background.

```
:app ───────────────► :presentation ─┐
  │  (Hilt root, Worker) │            ├──► :domain  (pure Kotlin: entity, error, repository API, usecase, validation, value)
  │                      │            │       ▲
  └─────────────────► :data ─────────┘       │ implements
                         └──► :core (AppLanguage, currentLanguage())  ◄── :presentation

Screen ── processIntent(Intent) ──► ViewModel ── Lazy<UseCase>.get()(Params) ──► UseCase
  ▲  state.collectAsStateWithLifecycle()   │                                       │ trim → Validator → Specification
  │  effect.collectLatest (LaunchedEffect) │                                       │ → TransactionRunner → Repository
  │                                        ▼                                       │ → (SyncRepository.sync())
  └── processEvent(Event) ──► NavGraph (NavBackStack<Route>)                       ▼
                                                            RepositoryImpl (internal) ─┬─ Room Dao (Flow / PagingSource)
  Flow<Domain> ── map { it.toUi() } ── State                                           ├─ DataStore (settings, user, token)
                                                                                       └─ ApiService ─ safeApiCall → Result
(if the app syncs) SyncRepository.sync() → WorkManager unique "sync_data" (KEEP) → SyncWorker → XxxRepository.sync()
```

Details: [references/architecture.md](references/architecture.md).

---

## Main rules

### Modules and layers
- New business rules go to `:domain` only: validator (`XxxInputValidator`), limit (`CanAddXxxSpecification`), scenario
  (`XxxUseCase`). Domain imports nothing from Android. → [architecture.md](references/architecture.md#domain)
- Everything in `:data`, `:presentation`, `:app` is `internal`; only `MainScreen` is public. → [conventions.md](references/conventions.md#visibility)
- Repository **interfaces** live in `domain/repository` (data) or `domain/gateway` (platform/technical: token, transaction,
  voice); implementations live in `data/repository` or `data/platform` and are bound with `@[Binds Singleton]`.
- A ViewModel talks to use cases only, never to a repository. Even a pass-through gets its use case
  (`ObserveNoteUseCase`). **[hypothesis: uniform dependency graph; the use case is the place a rule will appear later]**
- *(if the app syncs)* every write use case ends with `syncRepository.sync()`. → [data-layer.md](references/data-layer.md#sync)

### Domain
- Use case: `class XxxUseCase @Inject constructor(...)` with `operator fun invoke`; several inputs → nested `data class Params`.
  Order inside: **trim/normalize → validator → write (in a transaction when ≥2 writes or a quota check) → *(sync)* →
  `Result.success`**.
  → [examples.md](references/examples.md#1-a-write-scenario-through-all-layers-create-note)
- Validators return `XxxError?` (`null` = valid) from one `when`; limits are `const val` in their `companion object`.
- *(if the product has quotas)* Free/Pro limits are specifications (`check(currentCount, isPro): XxxError?`) with
  `LIMIT_FREE` / `LIMIT_PRO`, checked inside the write transaction.
- Money is `BigDecimal`, time is `java.time.Instant` in domain. → [data-layer.md](references/data-layer.md#types)

### UI (Compose, MVI)
- One folder per screen `feature/<area>/<screen>/` with exactly `XxxScreen`, `XxxViewModel`, `XxxState`, `XxxIntent`,
  `XxxEffect` (if one-shot results exist), `XxxEvent` (if it navigates). → [ui.md](references/ui.md#screen-anatomy)
- Screen signature: `processEvent: (XxxEvent) -> Unit, [screen params], modifier: Modifier = Modifier,
  viewModel: XxxViewModel = hiltViewModel()`. The screen never touches the back stack. → [ui.md](references/ui.md#screen)
- All navigation is decided in `NavGraph` by a `when (event)`; routes are `@Serializable` in `navigation/Screen.kt`;
  push with `navigateSingleton`, pop with `popBackStack`. → [ui.md](references/ui.md#navigation)
- No business logic or formatting in Composables: they render `state` and send intents. Local, purely visual flags
  (`showConfirmDeletionDialog`) may be `remember { mutableStateOf() }`.
- Design-system widgets come from `presentation/component` (`AppBar`, `AppButton`, `AppTextField`, `DialogMessage` …);
  Material primitives (`Scaffold`, `Text`, `Icon`, `FloatingActionButton`) are used directly. Values only from
  `MaterialTheme.spacing/.sizing/.appColorScheme/.colorScheme/.typography`; no raw `dp`, `sp`, `Color(0x…)` in screens. (legacy: raw `dp` in `CalculatorScreen`, `SubscriptionScreen`, `ContactsScreen`.)
  → [ui.md](references/ui.md#theme-and-tokens)

### State and concurrency
- `private val _state = MutableStateFlow(XxxState())` + `val state = _state.asStateFlow()`; mutate only with `_state.update { it.copy(...) }`.
- One-shot results: `Channel<XxxEffect>(Channel.CONFLATED)` + `receiveAsFlow()`, collected in
  `LaunchedEffect(Unit) { viewModel.effect.collectLatest { … } }`. → [state-and-concurrency.md](references/state-and-concurrency.md#effects)
- Progress flag pattern: set `isProcessing = true`, `viewModelScope.launch { … }.invokeOnCompletion { isProcessing = false }`.
- Route arguments come through `@HiltViewModel(assistedFactory = …)` + `hiltViewModel(creationCallback = …)`, not `SavedStateHandle`.
- Action use cases are injected as `dagger.Lazy<XxxUseCase>`; use cases observed in `init` are injected directly.
- Only `viewModelScope` / `coroutineScope` / `callbackFlow`; no `GlobalScope`, no ad-hoc `CoroutineScope(...)`
  (legacy: `MyFirebaseMessagingService`). Rethrow `CancellationException` in every catch-all.
  → [state-and-concurrency.md](references/state-and-concurrency.md)

### Data
- Room entities: `internal data class XxxEntity`, `snake_case` columns via `@ColumnInfo`, timestamps as epoch-millis
  `Long`, an `Index` for every `WHERE`/`ORDER BY`. *(if the app syncs)* plus `server_id`, `server_update_seq`, `is_synced`,
  and deletes go through `deleteWithSync` (tombstone `xxx_deletes` + delete in one `@Transaction`).
- Dynamic SQL only via `RoomRawQuery` with bound arguments; `ORDER BY`/`HAVING` fragments come from enums, never from input.
- Network: `ApiService` returns `Response<T>`; every call is wrapped in `safeApiCall { }` → `Result<T, NetworkError>`.
  DTOs are `@Serializable` with `@SerialName("snake_case")`. → [data-layer.md](references/data-layer.md)
- Mappers are top-level `internal fun X.toDomain()/toRequest()/toUi()` in a `mapper` package of the target layer.

### Errors
- New failure = a `data object` in the area's `sealed interface XxxError : RootError`; network failures are wrapped as
  `data class Network(val error: NetworkError)`. → [error-handling.md](references/error-handling.md)
- The ViewModel maps errors: field validation → `xxxErrorUiText` in State; quota → dialog flag in State; network →
  `SnackbarController.showMessage(getNetworkMessage(error) { code -> … })`.
- *(if the app has auth)* HTTP `401` means the session is gone → `LogoutUseCase` + snackbar `error_session_expired`
  (reference: `SyncType.LOGOUT` handled in `MainViewModel`).

### Security
- Secrets (auth token, keys) only through `TokenGateway`/`CryptoManager` (Keystore AES-GCM + DataStore); never in plain preferences, logs, URLs or analytics.
- No `Log`, no `printStackTrace` in new code (legacy: 4 places); no PII in any output.
- `PendingIntent` always `FLAG_IMMUTABLE`; FileProvider paths restricted to the needed cache folder.
  Full list: [security.md](references/security.md#security-checklist-every-change)

### Performance
- `@Immutable` on every State / Ui model; wrap unstable foreign types in `ImmutableHolder`.
- Lazy lists: `items(count, key = items.itemKey { it.id }, contentType = { "xxx" })`; static items have string keys.
- Derived UI booleans: `remember { derivedStateOf { … } }`; expensive values: `remember(key) { … }`.
- Ui models carry pre-formatted strings (`amountString`, `createdAtString`) built once in `toUi()`.
  → [performance.md](references/performance.md)

### DRY and raw values **[confirmed]**
- Extract at the **second** use, not the third. Scope of the extraction = scope of its users: one screen → `private` in
  the file; one feature → `feature/{area}/component/`, `mapper/{Entity}ErrorMapper.kt`; several features →
  `presentation/component/`, `util/`, theme tokens; several layers → domain constants/use cases; several modules →
  `build-logic` / `testFixtures`. → [conventions.md](references/conventions.md#dry-and-raw-values)
- UI text is never concatenated (`label + " *"`, `"$count/$max"`): use a format resource (`required_label`,
  `text_length_counter`) so word order and punctuation can change per locale.
- Literal design values (`Color(0x…)`, `16.dp`, `14.sp`) exist **only** in `ui/theme/*`; a new value = a new token.
- Duplicated logic in ViewModels (input caps, error → text) goes to extensions/mappers (`TextFieldValue.exceeds`,
  `{Entity}Error.toTitleErrorUiText()`).
- Lists of the same thing are data, not copy-paste: a row of settings items is a list + one item component.

### Style
- Named arguments for anything non-obvious (`params = …`, `intent = …`, `currentCount = …`); trailing lambda for `onClick`.
- `companion object { const val UPPER_SNAKE }` for every number/string with meaning; no magic numbers
  (legacy: `60_000L`, `(3..7)` in `MainViewModel`, PDF geometry/colors in `PdfGenerator`).
- Comments almost never; module build files keep `// Hilt`, `// Room3`-style group comments over dependencies.
- Every string exists in **every** locale the app ships, same key set (reference: default Russian + `en`, `uz`,
  `b+uz+Cyrl`, `tg`); the default locale is declared in `res/resources.properties`.
  → [conventions.md](references/conventions.md)

### Build
- **Always the latest versions** **[confirmed]** of every library, plugin, Gradle, SDK platform and JDK the toolchain
  supports — updated together, at project start, at every build change and monthly; breakages are fixed, not pinned around.
  → [build.md](references/build.md#latest-versions)
- Versions only in `gradle/libs.versions.toml` (including `compileSdk/minSdk/targetSdk/jdkVersion` under `# Variables`);
  related libraries are grouped into `[bundles]`.
- Shared module configuration lives in `build-logic` convention plugins (`{appName}.android.library`,
  `.android.application`, `.android.compose`, `.jvm.library`, `.hilt`) — a module script holds only its namespace and its own
  dependencies. (legacy: the reference project repeats the `android {}` block in every module.)
  → [build.md](references/build.md)

### Tests
- JVM unit tests for validators, specifications, use cases (in-memory fakes of domain interfaces) and ViewModels with
  `kotlinx-coroutines-test`; DAO tests on an in-memory Room in `androidTest`. The reference project has none yet — this is a
  **[proposal]**, verified in the skeleton. → [testing.md](references/testing.md)

### Git
- A commit = one logical change with a meaningful message. **[confirmed]**
- **No traces of AI in git:** no `Co-Authored-By: Claude…`, no "Generated with…", no mention of an assistant in messages,
  branches, PRs or code. **[confirmed]** → [conventions.md](references/conventions.md#git)

---

## Checklist before handing in code

**Layers**
- [ ] New rules are in `:domain` (validator / specification / use case); `:domain` has no Android import.
- [ ] ViewModel depends on use cases only; no DAO, `ApiService`, `DataStore` or repository in presentation.
- [ ] New classes in `:data` / `:presentation` / `:app` are `internal`; new bindings are `@[Binds Singleton]` in the module's `di`.
- [ ] Write use cases trim input and validate; quota checks and multi-writes run inside `TransactionRunner`;
  *(if the app syncs)* they end with `syncRepository.sync()`.

**UI / State**
- [ ] Screen folder has Screen / ViewModel / State / Intent (+ Effect / Event); one `processIntent`, exhaustive `when`.
- [ ] State is `@Immutable data class` with defaults; derived values are `val`s inside it.
- [ ] Navigation only through `processEvent` → `NavGraph`; new route is `@Serializable` in `Screen.kt`.
- [ ] No raw `dp`/`sp`/colors/strings in screens; every new string exists in every `values*/strings.xml`.
- [ ] Text inputs are capped in `processIntent` with the domain constant and in `AppTextField(maxLength = …)`.

**Data**
- [ ] New table: indices for its queries, mapper `toDomain()`; *(if it syncs)* sync columns, tombstone table, `toRequest()`.
- [ ] Every Retrofit call goes through `safeApiCall`; DTO fields use `@SerialName`.
- [ ] Dynamic SQL uses `RoomRawQuery` with bound arguments only.
- [ ] Schema change: bump `AppDatabase.version` + a `Migration`; `exportSchema = true` with `room.schemaLocation`
  (skeleton) **[proposal for the reference project]**.

**Concurrency**
- [ ] Only `viewModelScope`/structured scopes; no `GlobalScope`, no new `CoroutineScope(...)`, no `runBlocking` in new code.
- [ ] Catch-alls rethrow `CancellationException`; long jobs keep a `Job` reference to cancel the previous one.

**Performance**
- [ ] `key` + `contentType` in lazy lists; `derivedStateOf` for values derived from scrolling/paging state.
- [ ] Paged lists: `flatMapLatest` on a `distinctUntilChanged` filter + `cachedIn(viewModelScope)`.
  Full list: [performance.md](references/performance.md#performance-checklist-every-change).

**Security**
- [ ] No `Log`/`printStackTrace`/`println`; no token, email, name or amount in any output.
- [ ] Secrets/URLs are not added to code or VCS; `PendingIntent` is immutable; new manifest components are `exported="false"`
  unless they must be public. Full list: [security.md](references/security.md#security-checklist-every-change).

**DRY / raw values**
- [ ] Nothing new is used twice without a single home (component, token, color role, string, const, function, mapper).
- [ ] No literal text, number, `dp`/`sp`, color, key, header or MIME type outside its home; no concatenated UI text.

**Quality** (configured in the skeleton; the reference project has only Lint)
- [ ] New dependency: official (AndroidX / Google / JetBrains) or a written reason in the catalog comment.
- [ ] Every catalog entry, the Gradle wrapper and the SDK platform are the latest release; Lint shows no
  `GradleDependency` / `NewerVersionAvailable` warnings.
- [ ] `./gradlew detekt lintDebug checkRawValues` — zero findings.
- [ ] `./gradlew :domain:test testDebugUnitTest` — green.
- [ ] `./gradlew assembleDebug` — no new warnings (`org.gradle.warning.mode=all`).
- [ ] `./gradlew assembleRelease` — R8 passes (`optimization.enable = true`), keep rules updated if reflection was added.
- [ ] Android Studio *Reformat Code* with the official Kotlin style (`kotlin.code.style=official`), LF line endings.

**Git**
- [ ] One logical change per commit, meaningful message, no traces of AI.

---

## Never do

| ❌ Forbidden | Why |
|---|---|
| Business rules (limits, validation, "is Pro") in a ViewModel or Composable | Rules belong to `:domain`, the only module that is pure and reusable from Worker, FCM and UI |
| Android imports (`Context`, `Uri`, `R`, Room) in `:domain` | Breaks the pure-JVM module and its testability |
| A ViewModel injecting a repository, DAO, `ApiService` or `DataStore` | Skips the use case where rules, transactions and sync live |
| A public class in `:data` / `:presentation` | Other modules start depending on implementation details |
| Navigating from a Screen or ViewModel (`backStack.add`, `NavController`) | Navigation is decided in one place, `NavGraph`, from typed Events |
| Exposing `MutableStateFlow` / mutable collections from a ViewModel | One writer per state; the UI must not mutate it |
| One-shot results as State booleans that the UI must reset (`isSaved = true`) | Replays on recomposition / re-subscription; use a `CONFLATED` Effect channel |
| Throwing exceptions across layers for expected failures | Failures are `Result.Error(XxxError)`; the compiler must list them |
| `catch (e: Exception)` without rethrowing `CancellationException` | Breaks structured cancellation (see `safeApiCall`) |
| Hard-coded user-visible strings or text built in domain/data | Only presentation turns errors into `UiText`; all locales must keep the same keys (legacy: `VoiceRecognitionStatus.Error(message)` built in `:data`) |
| Raw `dp`, `sp`, `Color(0x…)` in screens | Tokens in `ui/theme` are the single source; dark theme and font scale depend on them |
| Magic numbers / strings | `const val` in a `companion object` with a meaningful name |
| `GlobalScope`, `CoroutineScope(Dispatchers.IO).launch` | Unstructured work leaks and survives its owner |
| `runBlocking` in new code | Blocks a thread; legacy only in `AuthInterceptor` |
| Network call without `safeApiCall` | An exception escapes as a crash instead of `NetworkError` |
| Showing server data that is also stored locally straight from the network | Screens observe local storage; the network feeds it |
| *(sync)* Deleting a synced row with a plain `DELETE` | The server never learns about it; use `deleteWithSync` (tombstone) |
| String-concatenating user input into SQL | Injection; bind with `RoomRawQuery.onBindStatement` |
| `Double` for money in domain or UI | Rounding drift; domain is `BigDecimal` (legacy: `Double` in Room/DTO) |
| Token in plain `SharedPreferences`/`DataStore`, in logs or URLs | Only `TokenGateway` → Keystore AES-GCM |
| `Log.*`, `printStackTrace()`, `println` | No logging policy exists; PII (names, amounts, emails) would leak to logcat |
| `PendingIntent` without `FLAG_IMMUTABLE`, `exported="true"` by default | Intent hijacking |
| Lazy list items without `key` / `contentType` | State jumps between rows; no reuse of compositions |
| Unstable types (`List<Domain>`, `Instant` holders, lambdas recreated) in State without `@Immutable` / `ImmutableHolder` | Every parent recomposition recomposes the whole subtree |
| A third-party library when the platform, AndroidX or an official Google/JetBrains library covers the need | Official first **[confirmed]**; third-party only with a written reason in the catalog comment |
| Copy-pasting a ViewModel block (voice recorder, multi-select) into another screen | DRY: extract (legacy: voice recording in 4 ViewModels, multi-select in 3) |
| A second copy of any UI fragment, even a padded button or two fields | Second use → component; a change must be made in one place **[confirmed]** |
| The same color / size / text / number written in two places | One home per value: color role, token, string resource, `const val` **[confirmed]** |
| Concatenated UI text (`label + " *"`, `"$a $b"`) | Format resource with arguments; word order differs per locale |
| Literal `Color(0x…)`, `dp`, `sp` outside `ui/theme` | `checkRawValues` fails the build; tokens are the only design source |
| Repeating `android {}` / Hilt / Compose setup in module scripts | One convention plugin in `build-logic` |
| Copying a fake into another module's tests | `testFixtures` of the module that owns the interface |
| Pinning an old version, skipping an update "for later", downgrading to avoid a migration | Always the latest versions **[confirmed]**; fix the migration in the same change |
| `Co-Authored-By`/"Generated with" in commits | No traces of AI in git **[confirmed]** |

---

## Navigating the references

| Task | Read |
|---|---|
| Where a new class goes, module graph, who may depend on whom | [references/architecture.md](references/architecture.md) |
| Naming files/classes/resources, enums vs value classes, constants, formatting, git | [references/conventions.md](references/conventions.md) |
| A new screen: Screen/State/Intent/Effect/Event, navigation, theme, components, dialogs | [references/ui.md](references/ui.md) |
| ViewModel state, effects, coroutines, Flow, lifecycle, process death, WorkManager | [references/state-and-concurrency.md](references/state-and-concurrency.md) |
| Room, DataStore, Retrofit, DTOs, mappers, paging, search, sync protocol | [references/data-layer.md](references/data-layer.md) |
| A new error, how it reaches the screen, texts, logging | [references/error-handling.md](references/error-handling.md) |
| Token storage, crypto, manifest, backups, release, logs | [references/security.md](references/security.md) |
| Recomposition, lists, formatting cost, startup, memory | [references/performance.md](references/performance.md) |
| Gradle, version catalog, convention plugins, detekt / Lint / `checkRawValues`, R8, locales, signing | [references/build.md](references/build.md) |
| What counts as a raw value and where it lives; when to extract a component | [references/conventions.md](references/conventions.md#dry-and-raw-values) |
| What and how to test | [references/testing.md](references/testing.md) |
| Reference code with explanations and anti-examples | [references/examples.md](references/examples.md) |
| Starting a new project from an empty folder | [templates/README.md](templates/README.md#a-new-project) |
| Creating a new feature (CRUD screen set) | [templates/README.md](templates/README.md#a-new-feature) |
| Adding offline sync or Free/Pro quotas | [templates/README.md](templates/README.md#optional-capabilities) |

## Order of work on a new feature

0. **New project?** Generate the skeleton from `templates/project/` first and build it
   ([templates/README.md](templates/README.md#a-new-project)).
1. **Domain model:** entity (`domain/entity`), value types/enums (`domain/value`), `XxxError` (`domain/error`).
2. **Rules:** `XxxInputValidator` with constants; *(quotas)* `CanAddXxxSpecification`.
3. **Contract:** `XxxRepository` interface; use cases `Observe…`, `Create…`, `Update…`, `Delete…`.
4. **Data:** Room entity (+ FTS if searchable) → DAO → register in `AppDatabase` (version + migration) → mappers →
   `XxxRepositoryImpl` → `@Binds` in `DataModule`. *(sync)* tombstone, DTOs, `ApiService` endpoint, `sync()`, a job in
   `SyncWorker`, `clear()` on logout.
5. **Presentation:** `XxxUi` + `toUi()` → State / Intent / Effect / Event → ViewModel → Screen → route in `Screen.kt` +
   `entry<…>` in `NavGraph` → strings in every locale.
6. **Tests** for validator, (specification), use case, ViewModel, DAO → the Quality commands above →
   security checklist → a logical commit without traces of AI.

Templates for all files: [templates/](templates/README.md).
