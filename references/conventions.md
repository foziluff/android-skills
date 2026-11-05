# Conventions: naming, structure, style

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## Where new files go

| What | Where | Name |
|---|---|---|
| Domain entity | `domain/entity/` | singular noun (`Note`, `Contact`) |
| Value / settings / enum | `domain/value/` | `ContactsFilter`, `AppTheme`, `FontScale`, `SyncType` |
| Error | `domain/error/` | `{Area}Error` (`NoteError`, `LoginError`); base `RootError` |
| Validator | `domain/validation/` | `{Area}InputValidator` (`NoteInputValidator`); shared limits in `CommonValidationRules` |
| Specification | `domain/specification/` | `CanAdd{Entity}Specification` |
| Data port | `domain/repository/` | `{Entity}Repository` |
| Technical port | `domain/gateway/` | `{Capability}Gateway` (`TokenGateway`), `TransactionRunner` |
| Use case | `domain/usecase/{area}/` | `{Verb}{Object}UseCase`: `Observe…`, `Observe…Paged…`, `Get…`, `Create…`, `Update…`, `Delete…s`, `Change…` (preferences), `Sync…` |
| Room entity | `data/local/database/entity/` | `{Entity}Entity`, tombstone `{Entity}DeleteEntity`, search `{Entity}FtsEntity` |
| Query result | `data/local/database/model/` | `{Entity}Tuple`, `{X}SummaryTuple` |
| DAO | `data/local/database/dao/` | `{Entity}Dao` |
| DataStore wrapper | `data/local/datastore/` | `{Area}DataStore` (`UserDataStore`) |
| DTO | `data/remote/request|response/` | entity DTO `{Entity}RequestDto` / `{Entity}ResponseDto`; endpoint body `{Action}Request` / `{Action}Response` (`LoginRequest`, `SyncNotesResponse`) |
| Repository impl | `data/repository/` | `{Entity}RepositoryImpl` |
| Platform impl | `data/platform/` | `{Port}Impl`, `Room{Port}`, `{X}Manager` |
| Mapper | `{layer}/mapper/` | `{Entity}Mapper.kt` (data), `CommonMapper.kt` (presentation) |
| Ui model | `presentation/model/` | `{Entity}Ui`, UI-only enums (`AppBarType`, `DebtType`, `SelectingType`) |
| Screen | `presentation/feature/{area}/{screen}/` | `{Screen}Screen`, `{Screen}ViewModel`, `{Screen}State`, `{Screen}Intent`, `{Screen}Effect`, `{Screen}Event` |
| Component | `presentation/component/` | `App{Name}` for design-system parts (`AppButton`, `AppTextField`), descriptive for composites (`DialogMessage`, `NavBar`) |
| Route | `presentation/navigation/Screen.kt` | `Screen.{Name}` / `TopLevelScreen.{Name}` |

Screen package examples: `feature/notes/add`, `feature/notes/edit`, `feature/notes/list`, `feature/contacts/detail`.
Screen names: `Add{Entity}`, `Edit{Entity}`, `{Entities}` (list), `{Entity}Details`.

## Method naming

### Repository (domain port)
| Name | Returns | Meaning |
|---|---|---|
| `observe(id)` | `Flow<X?>` | one item, live |
| `observePaged(filter)` | `Flow<PagingData<X>>` | list, live, paged |
| `observe{X}Summary(...)` | `Flow<XSummary>` | aggregate, live |
| `getAll(filter)` / `get{X}(...)` | `List<X>` / value | one-shot read (export, history) |
| `getCount()` | `Int` | for specifications |
| `create(...)` | `Long` (local id) | insert |
| `update(id, ...)` / `update{Field}(id, value)` | `Unit` | partial update |
| `delete(ids: List<Long>)` | `Unit` | bulk delete with tombstones |
| `clear()` | `Unit` | wipe on logout (+ reset autoincrement) |
| `sync()` | `Result<Unit, NetworkError>` | two-way sync of the entity |

### DAO
`loadById` (Flow), `pagingSource`, `getList` (RawQuery), `getCount`, `insert`, `update` (partial `@Query`),
`update{Entities}` (`@Update` list), `delete`, `deleteWithSync` (`@Transaction`), `clear`/`deleteAll`, `resetAutoIncrement`,
sync helpers `getUnsynced{Entities}(limit)`, `getDeletedIds(limit)`, `getMaxUpdateSeq`, `removeDeletedIds`,
`deleteByServerIds`, `getIdById`, `getIdByServerId`, `upsertFromServer`.

### ViewModel
- Public: `state`, `effect`, paged flows named after content (`notes`, `contacts`, `debts`), `processIntent(intent)`.
- Private: `handle{Intent}()` for non-trivial branches (`handleDelete`, `handleSaveClicked`), verbs for actions
  (`createNote`, `updateNote`, `loadDevices`, `startRecording`, `stopRecording`).

### Intent / Effect / Event
- Intent: what happened in UI, past tense or noun: `NoteTextChanged`, `SaveClicked`, `Delete`, `Refresh`,
  `Show{X}DialogChanged(visible)`, `{X}TypeChanged`.
- Effect: result: `Saved`, `NoteDeleted`, `LoggedIn`, `ContactDeleted`, `PdfUriGenerate(uri)`.
- Event: destination: `Back`, `AddNote`, `EditNote(id)`, `Subscription`, `UpgradeSubscriptionClicked`, `NavigateTo{X}`.

### State fields
`{field}Value`/`{field}` as `TextFieldValue`, `{field}ErrorUiText: UiText?`, `isProcessing`/`isLoading`/`progress`
(prefer `isProcessing` for saves, `isLoading` for reads), `show{X}Dialog: Boolean`, `selected{Entity}Ids: Set<Long>`,
`filterHolder: ImmutableHolder<Filter>`, derived `val is{X}` inside the class.

---

<a id="visibility"></a>
## Visibility

- `:domain` — public (it is the API of the module).
- `:data`, `:presentation`, `:app` — `internal` for **every** declaration, including DI modules, ViewModels, test classes.
  The only public symbol of `:presentation` is `MainScreen`.
- `private` for file-local Composables (sections of a screen: `BottomSaveButton`, `AddNoteDialogs`).
- (legacy: some DTOs and tombstone entities in `:data` are public — `ContactRequestDto`, `NoteDeleteEntity`; make new ones
  `internal`.)

---

## Enums, value classes, sealed types

- **Fixed set without data** → `enum class` (`AppTheme`, `ContactType`, `ContactsSort`, `SyncType`, `AppBarType`).
  Stored in DataStore by `name`, read back with `valueOf` (`AppTheme.valueOf(it)`).
- **Persisted numeric code with named constants** → `@JvmInline value class` with `companion object` values
  (`BlockingAppType(-1/0/1/2)`), or with an `init { require(...) }` range (`FontScale(-3..3)`).
  [hypothesis: stable storage codes independent of declaration order]
- **Alternatives with data** → `sealed interface` + `data object` / `data class` (`Result`, `XxxError`,
  `VoiceRecognitionStatus`, Intents/Effects/Events, routes).
- Always `sealed interface`, never `sealed class`, for these hierarchies.
- UI labels for domain enums are extension properties in presentation, not fields of the enum:
  ```kotlin
  internal val ContactsSort.labelRes: Int
      get() = when (this) {
          ContactsSort.BY_NAME_ASC -> R.string.sort_by_name_asc
          …
      }
  ```
  *Why:* domain does not know `R`.

## Constants, no magic values

- Every limit/size/key is a `const val UPPER_SNAKE` in the `companion object` of the class that owns the rule
  (`NoteInputValidator.MAX_TEXT_LENGTH`, `CanAddDebtSpecification.LIMIT_PRO`, `SyncWorker.WORK_NAME`,
  `AppDatabase.DB_NAME`, `CryptoManager.IV_LENGTH`) or a top-level `internal const val` for a file
  (`DEFAULT_TRANSITION_DURATION_MILLISECOND`, `MAXIMUM_FRACTION_DIGITS`).
- Defaults for settings/value objects are a `companion object { val Default = … }` (`AppSettings.Default`,
  `ContactsFilter.Default`, `AppConfig.Default`, `DebtsSummaryUi.Default`).
- DataStore keys are `private val xxxKey = …PreferencesKey("snake_case")` properties of the class that owns them.
- Big numbers with `_`: `2_000`, `400_000`.
- legacy: `60_000L` and `30 * 60_000L` in `MainViewModel.evaluateLock`, `(3..7)` days in `getNextRandomTimeMillis`,
  page geometry and hex colors in `PdfGenerator`, `pageSize = 30` repeated in three repositories — extract constants.

## Code style

- Kotlin official style (`kotlin.code.style=official`), 4 spaces, LF endings (legacy: some files have CRLF —
  `DebtMapper.kt`, `NoteRepositoryImpl.kt`, `DeviceMapper.kt`). **[proposal]** add `.editorconfig` with `end_of_line = lf`.
- Named arguments whenever the meaning is not obvious from the value:
  `specification.check(currentCount = currentCount, isPro = isPro)`, `viewModel.processIntent(intent = …)`.
- Expression bodies for one-liners (`override fun observe(...) = dao.loadById(id).map { it?.toDomain() }`).
- `?.let { return Result.error(it) }` early return for validators/specifications.
- `when` as the only branching for sealed/enum values; exhaustive, no `else` when the set is closed
  (legacy: `else -> {}` in `EditNoteViewModel.updateNote`).
- No `!!` anywhere (0 occurrences in the project). Use `?:`, `requireNotNull` with intent, or `?.let`.
- Unused catch parameter is `_`: `catch (_: UnknownHostException)`.
- Comments: almost none in code; build files group dependencies with `// Hilt`, `// Room3`, `// Modules` comments.
  `TODO` only with a reason (legacy: `// TODO server not sync limit` in `NoteRepositoryImpl`).
- Imports: no wildcards; alias on clashes (`import com.wigroup.domain.value.SyncState as DomainSyncState`).

## Resources and languages

- Locales: the default `values/` language is declared in `res/resources.properties` (`unqualifiedResLocale=…`) and listed
  in `localeFilters`; every module that has strings has **every** locale file with the same key set. Reference: default
  Russian + `en`, `uz`, `b+uz+Cyrl`, `tg`, 223 keys in each presentation file; skeleton: English only.
- String keys: `snake_case`, grouped by meaning: `error_*`, `empty_*_title/text/link`, `action_*`, `*_title`, `*_message`,
  `sort_by_*`, `filter_*`. Plurals for counts (`R.plurals.notes_deleted`).
- Strings belong to the module that shows them; `:app` has notification texts, `:presentation` has UI texts.
  (legacy: `:data` has `voice_error_*` because `VoiceRepositoryImpl` builds messages — target: return an error type and map
  to text in presentation.)
- Non-translatable ids are marked `translatable="false"` (`default_notification_channel_id`).
- Drawables: `snake_case` (`ic_notification`, `splash_green`, `brand_green`).

## Gradle files

- Plugins via `alias(libs.plugins.…)`, each group preceded by a comment (`// KSP`, `// Hilt`).
- Dependencies via catalog accessors only, bundles where they exist (`libs.bundles.room3`).
- See [build.md](build.md).

<a id="git"></a>
## Git

- A commit = one logical change with a meaningful message (not `changes`, not `fix`). **[confirmed]**
- **No traces of AI in git:** no `Co-Authored-By: Claude…`, no "Generated with…", no mention of an assistant in commit
  messages, branch names, PR descriptions or code comments. **[confirmed]**
- Never commit `local.properties`, keystores, `google-services.json` with real keys of another environment, or
  `project_analysis.md`-style scratch files (already in `.gitignore`).
- The analysed copy has no `.git`, so message format (prefixes, language) is not derivable — ask the author before inventing
  a convention.
