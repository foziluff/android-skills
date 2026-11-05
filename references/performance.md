# Performance: recomposition, lists, formatting, startup, memory

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

The reference app targets ~400 000 debts and 2 000 contacts per Pro user (`CanAddDebtSpecification.LIMIT_PRO`,
`CanAddContactSpecification.LIMIT_PRO`). Everything below keeps the UI cost proportional to what is **on screen**, not to the
size of the database.

---

## 1. Stability and recomposition

- Every State and Ui model is `@Immutable data class` (38 annotations in the project). Collections inside are read-only
  (`Set<Long>`, `List<…>`), always replaced, never mutated.
- Types Compose cannot infer as stable (domain classes from a non-Compose module, `ContactsFilter`) are wrapped:
  ```kotlin
  @Immutable
  @JvmInline
  internal value class ImmutableHolder<T>(val value: T)

  val filterHolder: ImmutableHolder<ContactsFilter> = ImmutableHolder(ContactsFilter.Default)
  ```
  *Why:* `:domain` has no Compose compiler, so its classes are unstable to Compose; the holder promises immutability
  without adding Compose to domain. [hypothesis]
- Small helper classes kept in `remember` are `@Stable` (`KeyboardController`).
- Classes used as `VisualTransformation` implement `equals`/`hashCode` (`CurrencyVisualTransformation`) so
  `remember`/skipping works.
- Theme tokens are `staticCompositionLocalOf` (never change at runtime → no tracking); app state locals are
  `compositionLocalOf` (they do change).
- Derived booleans in State bodies (`val isSearching = …`) are computed per state, not per recomposition.
- Lambdas passed down: method references `viewModel::processIntent`; sections receive values, not the whole ViewModel.

## 2. Lists

`feature/notes/list/NotesScreen.kt`
```kotlin
val notes = viewModel.notes.collectAsLazyPagingItems()

val isMessageContents by remember {
    derivedStateOf { notes.loadState.refresh is LoadState.NotLoading && notes.itemCount == 0 }
}

LazyColumn {
    item("message") { … }
    if (isMessageContents) item("message_content") { … }
    items(
        count = notes.itemCount,
        key = notes.itemKey { it.id },
        contentType = { "note" }
    ) { index ->
        val note = notes[index]
        if (note != null) NoteItem(note = note, modifier = Modifier.animateItem(), …)
    }
    item(key = "bottom_padding") { Spacer(Modifier.imePadding().height(MaterialTheme.sizing.navBarHeight)) }
}
```
Rules:
- Stable `key` (DB id) and `contentType` for paged items; string keys for static items.
- `derivedStateOf` for values computed from frequently changing state (load state, scroll position).
- Per-row expensive values `remember(index, colors) { … }` (avatar gradient in `ContactsScreen`).
- `Modifier.animateItem()` only works with keys — another reason they are mandatory.
- Paging config: page 30, prefetch 10, initial 30.

## 3. Formatting cost

- Strings shown in lists are built **once** in `toUi()`: `balanceString = balance.toCustomPrice()`,
  `createdAtString = createdAt.toDateTime()`, `nameFirst = name.first().uppercase()`.
- Formatters are cached per locale: `DateFormatter.FormatterCache` (`AtomicReference<CachedFormatters>`), `DecimalFormatter.
  NumberFormatCache` — no `DateTimeFormatter.ofPattern`/`NumberFormat.getInstance` per row.
- Aggregates (balances, summaries) are computed by SQLite (`SUM`, `MAX`, `GROUP BY`) with matching indices, not in Kotlin.
- Search is FTS4 (`MATCH`), not `LIKE '%…%'`.

## 4. Work on the right thread

- Room/DataStore/Retrofit suspend APIs are main-safe; blocking APIs (ContentResolver, raw list export, PDF, bitmap decode)
  run in `withContext(Dispatchers.IO)`.
- No `runBlocking` on Main (the only one, `AuthInterceptor`, runs on an OkHttp thread).
- Heavy one-time work (sync) is in WorkManager, not in a ViewModel.

## 5. Startup

- Single Activity, `installSplashScreen()` before `super.onCreate`, Compose content set immediately.
- `MainViewModel.init` only subscribes to flows and enqueues sync (no network on the critical path).
- ViewModels take `dagger.Lazy<UseCase>` for actions → the dependency graph behind Save/Delete is not built at screen open.
- **[proposal]** a Baseline Profile module (`androidx.baselineprofile`) for the start → contacts list path; there is none yet.
- **[proposal]** move `Modifier.composed` (`util/CircularIndicator.circularIndicator`) to `Modifier.Node` — `composed` is
  discouraged for performance.

## 6. Memory

- No `Context`/`Activity` stored in singletons; `@ApplicationContext` only (DataModule, repositories, ViewModels).
- `LoginScreen` wraps the Activity context in a `MutableContextWrapper` updated each composition, so the coroutine that
  outlives a recomposition never holds a stale Activity. [hypothesis]
- Listeners and observers are always removed: `awaitClose { recognizer.destroy() }`, `ProcessLifecycleOwner …
  removeObserver(this)` in `onCleared`.
- Bitmaps: decoded once for a notification; `Bitmap.createScaledBitmap` for the PDF brand image.
  legacy: `PdfGenerator` decodes the brand bitmap on every export and never recycles it — fine for a rare action, do not copy
  into list code.
- `SnackbarController` and navigation debounce are process-wide `object`/top-level state by design (one Activity).

## 7. Build-time performance

- `org.gradle.configuration-cache=true`; KSP (not kapt) for Hilt and Room.
- **[proposal]** `org.gradle.parallel=true` (currently commented out) — modules are decoupled enough.

---

## Performance checklist (every change)

- [ ] New State / Ui class is `@Immutable`; foreign/unstable types wrapped in `ImmutableHolder`.
- [ ] Lazy list items have `key` + `contentType`; static items have string keys.
- [ ] Values derived from fast-changing state use `derivedStateOf`; expensive per-row values use `remember(keys)`.
- [ ] Display strings are prepared in `toUi()` with cached formatters, not in Composables.
- [ ] Aggregations/filters/sorting happen in SQL with a supporting index; search uses FTS.
- [ ] Paged lists: `cachedIn(viewModelScope)`; filter changes through `distinctUntilChanged().flatMapLatest`.
- [ ] No blocking call on Main; blocking platform APIs inside `withContext(Dispatchers.IO)`.
- [ ] No Activity/View/Context held beyond its lifecycle; every listener has a matching unregister.
