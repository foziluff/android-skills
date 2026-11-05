# State and concurrency: ViewModel, Flow, coroutines, lifecycle, background work

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## ViewModel skeleton

`presentation/.../feature/notes/add/AddNoteViewModel.kt`
```kotlin
@HiltViewModel
internal class AddNoteViewModel @Inject constructor(
    private val createNoteUseCase: Lazy<CreateNoteUseCase>,               // action → dagger.Lazy
    private val startVoiceListenerUseCase: Lazy<StartVoiceListenerUseCase>,
) : ViewModel() {
    private val _state = MutableStateFlow(AddNoteState())
    val state = _state.asStateFlow()

    private val _effect = Channel<AddNoteEffect>(Channel.CONFLATED)
    val effect = _effect.receiveAsFlow()

    private var voiceJob: Job? = null

    fun processIntent(intent: AddNoteIntent) {
        when (intent) {
            is AddNoteIntent.NoteTextChanged -> {
                if (intent.value.text.trim().length > NoteInputValidator.MAX_TEXT_LENGTH) return
                _state.update { it.copy(noteValue = intent.value, noteErrorUiText = null) }
            }
            is AddNoteIntent.SaveClicked -> createNote()
            …
        }
    }

    private fun createNote() {
        _state.update { it.copy(isProcessing = true) }

        val cleanText = _state.value.noteValue.text.trim()      // snapshot before launching
        val noteChecked = _state.value.noteChecked

        viewModelScope.launch {
            createNoteUseCase.get().invoke(
                params = CreateNoteUseCase.Params(text = cleanText, checked = noteChecked)
            ).onSuccess {
                _effect.send(AddNoteEffect.Saved)
            }.onError { error ->
                when (error) {
                    NoteError.Empty -> _state.update {
                        it.copy(noteErrorUiText = UiText.StringResource(R.string.field_required))
                    }
                    NoteError.MaxCount -> _state.update { it.copy(showMaxCountDialog = true) }
                    NoteError.Limit -> _state.update { it.copy(showSubscriptionDialog = true) }
                    NoteError.MaxLength -> {}
                }
            }
        }.invokeOnCompletion {
            _state.update { it.copy(isProcessing = false) }
        }
    }
}
```

### Rules
- **State:** one `MutableStateFlow<XxxState>`, private; public `asStateFlow()`. Mutations only `_state.update { it.copy() }`
  (atomic, no lost updates between coroutines). Never `stateIn` for screen state (0 uses in the project).
  [hypothesis: state is assembled from several sources via `onEach { _state.update }`, so one writable flow is simpler]
- **Observed sources** in `init`: `useCase().onEach { _state.update { … } }.launchIn(viewModelScope)` — one chain per source.
  Combining sources that depend on state: `combine(a, _state.map { … }.distinctUntilChanged().flatMapLatest { … })`
  (`ReportsViewModel`).
- **Progress:** set the flag before `launch`, clear it in `invokeOnCompletion` — runs on success, error **and**
  cancellation, so the button never stays disabled.
- **Snapshot inputs** from `_state.value` before `launch` so a later keystroke does not change what is saved.
- **Input caps** at the entry: `if (text.trim().length > MAX) return` — the state never holds an invalid length.
  Numeric input goes through `currencyFilterValue(value)?.let { … }` (digits, one dot, 2 decimals, max value from
  `DebtInputValidator.MAX_VALUE`).
- **Long-running collectors** (voice, timers) keep a `Job?` and cancel the previous one before starting:
  `voiceJob?.cancel(); voiceJob = viewModelScope.launch { … }`.
- **Dependencies:** `dagger.Lazy<XxxUseCase>` for use cases called on user action, called as `useCase.get()(params)`;
  direct injection for use cases observed in `init` (they are needed immediately anyway).
  [hypothesis: ViewModel creation stays cheap — the use case graph (validator, specification, repositories) is built only if
  the user actually taps Save] *Cost:* `.get()` noise.
- **No Android in ViewModels.** (legacy: `MainViewModel`, `SettingsViewModel`, `DataBackupViewModel` inject
  `@ApplicationContext` to open Play Store / share text / build a PDF — target: emit an Effect (`OpenUri`, `ShareText`) and
  let the screen call `context.openUri(...)`; move PDF generation behind a domain gateway.)

<a id="effects"></a>
## One-shot effects

```kotlin
private val _effect = Channel<XxxEffect>(Channel.CONFLATED)
val effect = _effect.receiveAsFlow()
```
Screen:
```kotlin
LaunchedEffect(Unit) {
    viewModel.effect.collectLatest { effect ->
        when (effect) {
            XxxEffect.Saved -> onBack()
        }
    }
}
```
*Why a Channel, not a `SharedFlow` or a State flag:* [hypothesis] a value sent while the screen is not collecting
(rotation, background) is kept and delivered once; `CONFLATED` keeps only the latest, so double taps cannot queue two
navigations. *Trade-off:* only one collector — never collect `effect` in two places.

App-wide messages use the same pattern in `util/SnackbarController` (`object` with a `CONFLATED` channel and a single
`SnackbarHostState`), observed once by `ObserveSnackbarEvents()` in `MainScreen`, each message limited by
`withTimeoutOrNull(5.seconds)`.

## Route arguments: assisted injection

```kotlin
@HiltViewModel(assistedFactory = EditNoteViewModel.Factory::class)
internal class EditNoteViewModel @AssistedInject constructor(
    @Assisted private val noteId: Long,
    observeNoteUseCase: ObserveNoteUseCase,
    private val updateNoteUseCase: Lazy<UpdateNoteUseCase>,
) : ViewModel() {
    …
    @AssistedFactory
    internal interface Factory {
        fun create(noteId: Long): EditNoteViewModel
    }
}
```
`NavGraph`: `hiltViewModel(creationCallback = { factory: EditNoteViewModel.Factory -> factory.create(it.noteId) })`.
*Why not `SavedStateHandle`:* [hypothesis] arguments are typed constructor parameters (no string keys, no nullable reads),
and Navigation 3 routes are already serializable objects. `SavedStateHandle` is not used anywhere (0 occurrences).

## Process death and configuration changes

| What | Survives rotation | Survives process death | How |
|---|---|---|---|
| Back stack + route args | ✅ | ✅ | `rememberNavBackStack` = `rememberSerializable` of `@Serializable` routes |
| ViewModel state | ✅ | ❌ | `ViewModel` per nav entry (`rememberViewModelStoreNavEntryDecorator`) |
| Persisted data | ✅ | ✅ | Room / DataStore — the screen re-observes it (`EditNoteViewModel` re-reads the note) |
| Small UI flags | ✅ | ✅ | `rememberSaveable` (bottom sheet visibility) |
| Unsaved form text (Add screens) | ✅ | ❌ | **accepted loss** [hypothesis] |

**[proposal]** if a form must survive process death, persist the draft through a use case (DataStore) rather than
introducing `SavedStateHandle` for one screen.

## Dispatchers

- Room `suspend`/`Flow` DAOs, DataStore and Retrofit `suspend` are main-safe, so repositories normally do **not** switch
  dispatchers.
- `withContext(Dispatchers.IO)` is used only around blocking work: `ContentResolver.query` (`getContactNameFromUri`),
  `@RawQuery` list reads (`getAll`), `PdfGenerator`, bitmap download.
- Dispatchers are **not injected** — hard-coded `Dispatchers.IO` in those places. [hypothesis: no tests that need to
  replace them] **[proposal]** when tests appear, inject `@IoDispatcher CoroutineDispatcher` via a Hilt qualifier in the
  classes that switch, and keep everything else dispatcher-free.

## Structured concurrency

- Scopes used: `viewModelScope`, `coroutineScope { }` in `SyncWorker.doWork`, `callbackFlow { … awaitClose { } }` for
  callback APIs (`VoiceRepositoryImpl` registers a `RecognitionListener` and destroys the recognizer in `awaitClose`),
  `LaunchedEffect` / `rememberCoroutineScope` in UI.
- Parallel independent work: `launch` per job + `joinAll()`; shared mutable result guarded by `Mutex.withLock`
  (`SyncWorker`). Dependent work is nested (`contactRepository.sync().onSuccess { debtRepository.sync() }` — debts need
  contact server ids).
- `CancellationException` is always rethrown before a generic catch (`safeApiCall`).
- legacy:
  - `MyFirebaseMessagingService` uses `CoroutineScope(Dispatchers.IO).launch` (unowned scope) and a new `OkHttpClient()` per
    image. Target: enqueue work (WorkManager) or use a service-owned `SupervisorJob` scope cancelled in `onDestroy`, and an
    injected client.
  - `AuthInterceptor` reads the token with `runBlocking { tokenGateway.observeToken().firstOrNull() }` — runs on OkHttp's
    thread, so it does not block Main, but decrypts on every request. [hypothesis: acceptable at current traffic]
    **[proposal]** cache the decrypted token in memory in `TokenGatewayImpl`, invalidated on save/clear.

## Flow operators in use

| Need | Operator |
|---|---|
| Re-query when a filter/search changes | `_state.map { filter }.distinctUntilChanged().flatMapLatest { useCase(filter) }` |
| Keep paging data across recompositions/config changes | `.cachedIn(viewModelScope)` |
| Map paged items | `.map { pagingData -> pagingData.map { it.toUi() } }` |
| Ignore identical emissions from storage | `.distinctUntilChanged()` at the repository (`UserDataStore.observeUser`, `observeToken`) |
| One-time current value | `observe().first()` / `firstOrNull()` (current user's Pro flag in a use case) |
| Skip nulls of a deleted row | `.filterNotNull()` |

## Lifecycle

- UI collects with `collectAsStateWithLifecycle()` (stops when the app is in background).
- App foreground/background (app lock, backup prompt) — `MainViewModel` implements `DefaultLifecycleObserver` and registers on
  `ProcessLifecycleOwner` in `init`, removes in `onCleared`. `onStop` stores `lastBackgroundTime`, `onStart` evaluates the lock.
- Screen-level: `LifecycleEventEffect(Lifecycle.Event.ON_STOP) { … }`.

## *(optional)* Background work: WorkManager sync

`app/.../worker/SyncWorker.kt` (shortened)
```kotlin
suspend fun enqueue(dataStore: DataStore<Preferences>, workManager: WorkManager) {
    dataStore.edit { it[isSyncPendingKey] = true }           // "someone wants a sync"

    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
        .build()

    workManager.enqueueUniqueWork(WORK_NAME, ExistingWorkPolicy.KEEP, request)
}

override suspend fun doWork(): Result = coroutineScope {
    do {
        dataStore.edit { it[isSyncPendingKey] = false }
        // launch config/user/notes/contacts→debts in parallel, joinAll()
    } while (dataStore.data.first()[isSyncPendingKey] == true)   // a write happened during the sync → run again
    Result.success(workDataOf(KEY_SYNC_STATE to Json.encodeToString(SyncState.serializer(), syncState)))
}
```
- *Why `KEEP` + a pending flag:* [hypothesis] every write triggers `sync()`; `KEEP` coalesces bursts into one running job,
  and the flag guarantees a write made during the running sync is not lost (the loop runs again).
- Expedited with fallback; `setForeground(getForegroundInfo())` wrapped in `try/catch (_: IllegalStateException)`
  (not allowed from background on new Android versions) with `FOREGROUND_SERVICE_TYPE_DATA_SYNC` on Q+.
- Result per area (`commonState`, `contactsState`, `notesState`) is serialized into the work output; `SyncRepositoryImpl`
  turns `WorkInfo` into domain `SyncState` (`LOADING/SUCCESS/ERROR/LOGOUT`) as a `Flow`.
- legacy: the `401 → SyncType.LOGOUT else ERROR` expression is repeated four times — extract a private function.
