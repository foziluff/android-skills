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
| Feature component (used by ≥2 screens of one area) | `presentation/feature/{area}/component/` | `{Entity}Form`, `{Entity}Row` |
| Error → text mapper | `presentation/mapper/` | `{Entity}ErrorMapper.kt` with `{Entity}Error.to{Field}ErrorUiText()` |
| Test fake of a domain port | `domain/src/testFixtures/kotlin/.../fake/` | `Fake{Entity}Repository` (public, shared by every module's tests) |
| Shared Gradle configuration | `build-logic/convention/src/main/kotlin/` | `AndroidLibraryConventionPlugin`, `ProjectExtensions.kt` |
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

<a id="dry-and-raw-values"></a>
## DRY and raw values **[confirmed]**

**Goal:** changing a color, a padding, a text, a limit, a button or a dialog is an edit in **one** file; nobody searches the
project. **Rule:** whatever is used a second time gets a single home *at the second use* — however small it is.

### Where every kind of value lives

| Kind of value | Its only home | Example (skeleton / reference) |
|---|---|---|
| User-visible text | `res/values*/strings.xml` (every locale), `UiText` from ViewModels | `R.string.field_required` |
| Text with variable parts | format resource with `%1$s` / `%1$d`, never `+` or templates | `required_label` = `%1$s *`, `text_length_counter` (`translatable="false"`) |
| Plural text | `<plurals>` + `pluralStringResource` / `UiText.PluralResource` | `R.plurals.notes_deleted` |
| Color | palette `ui/theme/Color.kt` → role in `ColorScheme` / `AppColorScheme` → `MaterialTheme.colorScheme.x` | `MaterialTheme.appColorScheme.link` |
| Spacing / size / radius / stroke | `Spacing` / `Sizing` tokens | `MaterialTheme.spacing.space200`, `MaterialTheme.sizing.emptyStateIcon` |
| Text style / font size | `ui/theme/Type.kt` → `MaterialTheme.typography.x` | `bodyLarge` |
| Alpha, fractions, durations, easing | `private const val` in the component/file, or a token if shared | `INDICATOR_ALPHA`, `EMPTY_STATE_HEIGHT_FRACTION`, `DEFAULT_TRANSITION_DURATION_MILLISECOND` |
| Business limit / length / regex | `const val` in the domain validator/specification; UI reuses it | `TaskItemInputValidator.MAX_TITLE_LENGTH` in `AppTextField(maxLength = …)` |
| Technical identifier (DataStore key, work name, channel id, lazy-list key/contentType, Keystore alias/provider, DB name) | `private val`/`const val` owned by the one class that uses it | `themeKey`, `SyncWorker.WORK_NAME`, `TASK_ITEM_CONTENT_TYPE`, `KEYSTORE_PROVIDER` |
| HTTP header / MIME type / prefix | `const val` (shared ones in `ApiFactory`) | `ApiFactory.CONTENT_TYPE_JSON`, `AuthInterceptor.BEARER_PREFIX` |
| HTTP status with meaning | named constant | `HTTP_UNAUTHORIZED = 401` (legacy: `401`/`429` literals in reference ViewModels) |
| Fixed set of options | `enum` / `sealed interface` / value class with named instances | `AppTheme`, `BlockingAppType.IMMEDIATELY` |
| Defaults of a settings object | `companion object { val Default = … }` | `AppSettings.Default` |
| Base URL / environment | `buildConfigField` per build type | `BuildConfig.BASE_URL` |
| SQL | `@Query` text or SQL fragments inside the repository's query builder; fragments chosen by enums | `pagingConstructQuery` |

Allowed literals: `0`, `1`, `-1`, `true/false`, empty string, and literals **inside** their home (a token definition, a
`const val`, a resource, an `@Query`, a test).

### Where an extracted thing goes (scope = scope of its users)

| Used by | UI fragment | Logic | Value |
|---|---|---|---|
| one file | `private @Composable` in the file | `private fun` | `private const val` at the bottom of the file |
| one screen package | same file or `{Screen}Components.kt` in the package | private in ViewModel / mapper | companion `const val` |
| one feature (add + edit + list) | `feature/{area}/component/` (`{Entity}Form`) | `mapper/{Entity}ErrorMapper.kt`, `{Entity}` extensions | domain validator constants |
| several features | `presentation/component/App{Name}` (`AppBottomButton`) | `presentation/util/` (`TextFieldValue.exceeds`) | theme token / color role / string resource |
| several layers | — | domain use case / validator / specification | domain constant |
| several modules (build/test) | — | `build-logic` convention plugin; `testFixtures` | version catalog |

### What it looks like

```kotlin
// ❌ copy in AddScreen and EditScreen
Box(Modifier.imePadding().background(MaterialTheme.colorScheme.background).padding(24.dp, 8.dp, 24.dp, 16.dp)) {
    AppButton(text = "Save", modifier = Modifier.fillMaxWidth(), onClick = onSave)
}
AppTextField(label = stringResource(R.string.title) + " *", …)

// ✅ one component, one string, tokens
AppBottomButton(text = stringResource(R.string.save), enabled = !state.isProcessing, onClick = { … })
AppTextField(label = stringResource(R.string.required_label, stringResource(R.string.task_item_title)), …)
```
```kotlin
// ❌ the same branch in two ViewModels
.onError { error -> when (error) { TaskItemError.TitleEmpty -> …field_required…; else -> {} } }

// ✅ mapper/TaskItemErrorMapper.kt
internal fun TaskItemError.toTitleErrorUiText(): UiText? = when (this) {
    TaskItemError.TitleEmpty -> UiText.StringResource(R.string.field_required)
    TaskItemError.TitleMaxLength, TaskItemError.DescMaxLength, TaskItemError.ListEmpty -> null
}
```

### Components must stay generic
- Parameters for what differs (`text`, `enabled`, colors with theme defaults), never a copy with one changed line.
- No feature types in `presentation/component/` (no `TaskItemUi` parameter) — pass primitives/strings/lambdas; feature
  types belong to `feature/{area}/component/`.
- Before writing a widget, look in `component/` and `feature/{area}/component/`.

### Enforcement (skeleton)
- `detekt` (`config/detekt/detekt.yml`): `MagicNumber` (theme files and tests excluded), `StringLiteralDuplication`,
  `UnsafeCallOnNullableType`, `WildcardImport`, `MaxLineLength 120`.
- Android Lint (`config/lint/lint.xml`): `HardcodedText`, `SetTextI18n`, `MissingTranslation`, `ExtraTranslation`,
  `StringFormat*`, `UnusedResources` as errors.
- `./gradlew checkRawValues`: `Text("…")`, `N.dp`, `N.sp`, `Color(0x…)` outside `ui/theme` fail the build
  (Lint does not see literals inside Compose calls).
- Review: grep the diff for the same string/number twice; every new widget used twice is a component.

### Reference project (legacy, fix when touching)
`60_000L` and `30 * 60_000L` in `MainViewModel.evaluateLock`, `(3..7)` days in `getNextRandomTimeMillis`, page geometry and
hex colors in `PdfGenerator`, `pageSize = 30` in three repositories, `401`/`429` literals, raw `dp` in three screens,
`"${balanceString} ${currencySymbol}"` concatenations, voice-recording and multi-select blocks copied between ViewModels,
the sync loop copied in three repositories.

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
