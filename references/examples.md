# Reference examples with explanations

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

All fragments come from the project (the path is above each one), shortened to the essentials. Where real code differs from
the target rule it is marked **legacy → target**.

---

## 1. A write scenario through all layers: create note

### Intent → ViewModel
`presentation/.../feature/notes/add/AddNoteViewModel.kt`
```kotlin
is AddNoteIntent.NoteTextChanged -> {
    if (intent.value.text.trim().length > NoteInputValidator.MAX_TEXT_LENGTH) return   // cap at the entry
    _state.update { it.copy(noteValue = intent.value, noteErrorUiText = null) }       // edit clears the error
}

private fun createNote() {
    _state.update { it.copy(isProcessing = true) }
    val cleanText = _state.value.noteValue.text.trim()
    val noteChecked = _state.value.noteChecked

    viewModelScope.launch {
        createNoteUseCase.get().invoke(params = CreateNoteUseCase.Params(text = cleanText, checked = noteChecked))
            .onSuccess { _effect.send(AddNoteEffect.Saved) }
            .onError { error ->
                when (error) {
                    NoteError.Empty -> _state.update { it.copy(noteErrorUiText = UiText.StringResource(R.string.field_required)) }
                    NoteError.MaxCount -> _state.update { it.copy(showMaxCountDialog = true) }
                    NoteError.Limit -> _state.update { it.copy(showSubscriptionDialog = true) }
                    NoteError.MaxLength -> {}
                }
            }
    }.invokeOnCompletion { _state.update { it.copy(isProcessing = false) } }
}
```
- The ViewModel decides *how* each error looks; it does not decide *whether* the note is valid.
- `invokeOnCompletion` resets progress on every outcome, including cancellation.

### Use case
`domain/.../usecase/note/CreateNoteUseCase.kt` — see [architecture.md](architecture.md#use-case): trim → validator →
count + specification → `create` → `sync()` → `Result.success(noteId)`.

### Repository → DAO
`data/.../repository/NoteRepositoryImpl.kt`
```kotlin
override suspend fun create(text: String, checked: Boolean): Long {
    val now = System.currentTimeMillis()
    return noteDao.insert(
        note = NoteEntity(
            id = 0, serverId = null, serverUpdateSeq = null,
            text = text, checked = checked,
            isSynced = false,                    // dirty until the worker uploads it
            createdAt = now, updatedAt = now,
        )
    )
}
```

### Screen reacts
```kotlin
LaunchedEffect(Unit) {
    viewModel.effect.collectLatest { effect ->
        when (effect) {
            AddNoteEffect.Saved -> onBack()      // Effect → Event: the screen translates
        }
    }
}
```
The list screen updates by itself: it observes Room through Paging; nobody "refreshes" it.

### ❌ Not like this
```kotlin
// ViewModel with rules, storage and navigation mixed in
fun save() = viewModelScope.launch {
    if (text.isBlank()) { error = "Required"; return@launch }               // rule + hard-coded text in VM
    if (!user.isPro && noteDao.getCount() >= 10) { showUpgrade = true }     // magic number, DAO in presentation
    api.createNote(NoteDto(text))                                           // network first: fails offline
    isSaved = true                                                           // one-shot as state → replays
    navController.popBackStack()                                             // navigation from the VM
}
```

---

## 2. A paged, searchable list

### ViewModel
`presentation/.../feature/notes/list/NotesViewModel.kt`
```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
val notes = _state
    .map { it.searchQuery.text.trim().ifBlank { null } }     // only the part that affects the query
    .distinctUntilChanged()                                  // cursor moves do not re-query
    .flatMapLatest { searchQuery -> observeNotesPagedUseCase(searchQuery) }   // cancel previous query
    .map { pagingData -> pagingData.map { note -> note.toUi() } }
    .cachedIn(viewModelScope)
```

### Repository
`data/.../repository/NoteRepositoryImpl.kt` builds a `RoomRawQuery`: `SELECT * FROM notes ORDER BY updated_at DESC` or a
`JOIN notes_fts … WHERE notes_fts MATCH ? || '*'` with the query **bound** — see [data-layer.md](data-layer.md#room).

### Screen
`items(count = notes.itemCount, key = notes.itemKey { it.id }, contentType = { "note" })`, empty state through
`derivedStateOf`, search/selection in `NotesState`, `BackHandler` collapses search/selection first — see
[performance.md](performance.md#2-lists).

### ❌ Not like this
```kotlin
val notes = MutableStateFlow<List<Note>>(emptyList())                      // whole table in memory
fun search(q: String) = viewModelScope.launch {
    notes.value = repository.getAll().filter { it.text.contains(q) }       // Kotlin filtering, no FTS, no cancel
}
// "SELECT * FROM notes WHERE text LIKE '%$q%'"                             // injection + full scan
LazyColumn { items(notes) { NoteItem(it) } }                               // no key, no contentType
```

---

## 3. A network error from `safeApiCall` to the snackbar

`data/.../repository/AuthRepositoryImpl.kt`
```kotlin
override suspend fun login(username: String, password: String, deviceName: String): Result<String, LoginError> =
    safeApiCall { apiService.login(request = LoginRequest(username = username, password = password, deviceName = deviceName)) }
        .map { it.token }
        .mapError { LoginError.Network(it) }
```
`domain/.../usecase/auth/LoginUseCase.kt`
```kotlin
return authRepository.login(username = clearUsername, password = clearPassword, deviceName = clearDeviceName)
    .onSuccess { token ->
        tokenGateway.saveToken(token)
        userRepository.sync()
            .onSuccess {
                userRepository.registerInstallationId()
                syncRepository.sync()
            }
            .onError { error ->
                tokenGateway.clearToken()                                  // no half-logged-in state
                return Result.error(LoginError.Network(error))             // non-local return from inline onError
            }
    }.map {}
```
`presentation/.../feature/login/LoginViewModel.kt`
```kotlin
is LoginError.Network -> {
    getNetworkMessage(it.error) { code ->
        when (code) {
            401 -> UiText.StringResource(R.string.error_invalid_credentials)
            429 -> UiText.StringResource(R.string.error_too_many_requests)
            else -> UiText.StringResource(R.string.error_unknown)
        }
    }?.let { networkMessage -> SnackbarController.showMessage(networkMessage) }
}
```
- Transport errors (no internet, timeout) are mapped once in `getNetworkMessage`; the screen maps only HTTP codes that mean
  something to it.
- **legacy → target:** `401`/`429` literals → named constants.

### ❌ Not like this
```kotlin
try {
    val token = api.login(body).body()!!.token            // !!, crash on 4xx
} catch (e: Exception) {                                  // swallows CancellationException
    _state.update { it.copy(error = e.message) }          // raw exception text to the user, not localized
}
```

---

## 4. Atomic multi-write with quotas

`domain/.../usecase/contact/CreateContactWithDebtUseCase.kt`
```kotlin
contactValidator.validate(clearContactName, clearContactDesc)?.let { return Result.error(it) }
debtValidator.validate(debtParams.amount, clearDebtDesc)?.let { return Result.error(it) }

return transactionRunner.withWriteTransaction {
    val currentContactsCount = contactRepository.getCount()
    val isPro = userRepository.observe().firstOrNull()?.isPro == true

    contactSpecification.check(currentCount = currentContactsCount, isPro = isPro)
        ?.let { return@withWriteTransaction Result.error(it) }

    val currentDebtsCount = debtRepository.getCount()
    debtSpecification.check(currentCount = currentDebtsCount, isPro = isPro)
        ?.let { return@withWriteTransaction Result.error(it) }

    val contactId = contactRepository.create(name = clearContactName, desc = clearContactDesc)
    debtRepository.create(contactId = contactId, amount = debtParams.amount, desc = clearDebtDesc)

    Result.success(contactId)
}.onSuccess { syncRepository.sync() }        // sync only after commit
```
- Validation (cheap, stateless) is outside the transaction; counting + inserting is inside — no "contact without its first
  debt", no quota race between two quick saves.
- Return type `Result<Long, RootError>` because two areas can fail.

### ❌ Not like this — **legacy** `CreateDebtUseCase`
```kotlin
val currentCount = debtRepository.getCount()                // read outside …
return transactionRunner.withWriteTransaction {             // … check and write inside: count may be stale
    debtSpecification.check(currentCount = currentCount, isPro = isPro)?.let { … }
```
Target: move `getCount()` inside the transaction, as in example 4.

---

## 5. Route arguments and navigation

`presentation/.../navigation/Screen.kt`
```kotlin
@Serializable
data class EditNote(val noteId: Long) : Screen
```
`presentation/.../navigation/NavGraph.kt`
```kotlin
entry<TopLevelScreen.Notes>(metadata = navAnimationNone()) {
    NotesScreen(
        processEvent = { event ->
            when (event) {
                NotesEvent.Back -> backStack.navigateTopLevel(screen = TopLevelScreen.startTopLevelScreen)
                NotesEvent.AddNote -> backStack.navigateSingleton(screen = Screen.AddNote)
                is NotesEvent.EditNote -> backStack.navigateSingleton(screen = Screen.EditNote(noteId = event.noteId))
                NotesEvent.Subscription -> backStack.navigateSingleton(Screen.Subscription)
            }
        },
        backHandlerEnabled = backHandlerEnabled
    )
}
entry<Screen.EditNote> {
    EditNoteScreen(
        processEvent = { event -> when (event) { EditNoteEvent.Back -> backStack.popBackStack() } },
        viewModel = hiltViewModel(creationCallback = { factory: EditNoteViewModel.Factory -> factory.create(it.noteId) })
    )
}
```
- The whole app's navigation is readable in one file.
- `navigateSingleton` debounces double taps (transition duration) and ignores pushing the same route twice.

### ❌ Not like this
```kotlin
@Composable fun NotesScreen(backStack: NavBackStack<Route>) {     // screen coupled to navigation
    NoteItem(onClick = { backStack.add(Screen.EditNote(note.id)) })
}
class EditNoteViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    private val noteId = savedStateHandle.get<Long>("noteId")!!   // stringly-typed key, !!
}
```

---

## 6. Offline delete with a tombstone

`data/.../local/database/dao/NoteDao.kt`
```kotlin
@Query("""
    INSERT OR IGNORE INTO note_deletes (server_id)
    SELECT server_id FROM notes
    WHERE id IN (:ids) AND server_id IS NOT NULL
""")
suspend fun insertDeletedIdsFromNotes(ids: List<Long>)

@Transaction
suspend fun deleteWithSync(ids: List<Long>) {
    insertDeletedIdsFromNotes(ids)
    delete(ids)
}
```
`domain/.../usecase/note/DeleteNotesUseCase.kt`
```kotlin
suspend operator fun invoke(noteIds: List<Long>): Result<Unit, NoteError> {
    if (noteIds.isEmpty()) return Result.error(NoteError.Empty)
    noteRepository.delete(noteIds)
    syncRepository.sync()
    return Result.success(Unit)
}
```
- The selection is capped at `CommonValidationRules.MAX_BULK_DELETE_COUNT` (200) in the ViewModel, which keeps `IN (:ids)`
  under SQLite's variable limit and matches the sync batch.

### ❌ Not like this
```kotlin
@Query("DELETE FROM notes WHERE id IN (:ids)") suspend fun delete(ids: List<Long>)   // called directly → server never learns
```

---

## 7. A secret on disk

`data/.../platform/TokenGatewayImpl.kt`
```kotlin
override suspend fun saveToken(token: String) {
    val encryptedToken = cryptoManager.encrypt(token)                 // AndroidKeyStore AES-GCM, IV prepended
    dataStore.edit { preferences -> preferences[tokenKey] = encryptedToken }
}

override fun observeToken(): Flow<String?> = dataStore.data.map { preferences ->
    preferences[tokenKey]?.let { encryptedToken -> cryptoManager.decrypt(encryptedToken) }   // null on failure → logged out
}.distinctUntilChanged()
```
`domain/.../usecase/auth/ObserveAuthStatusUseCase.kt`: `tokenGateway.observeToken().map { !it.isNullOrBlank() }` — the rest of
the app sees only a `Boolean`.

### ❌ Not like this
```kotlin
prefs.edit { putString("token", token) }          // plaintext, readable from backups / rooted devices
Log.d("Auth", "token=$token")                     // leaks to logcat and bug reports
```

---

## 8. A callback API as a Flow

`data/.../platform/VoiceRepositoryImpl.kt`
```kotlin
override fun startListening(): Flow<VoiceRecognitionStatus> = callbackFlow {
    trySend(VoiceRecognitionStatus.Ready)
    val recognizer = SpeechRecognizer.createSpeechRecognizer(context)
    recognizer.setRecognitionListener(object : RecognitionListener {
        override fun onPartialResults(partialResults: Bundle?) { … trySend(VoiceRecognitionStatus.PartialResult(matches[0])) }
        override fun onResults(results: Bundle?) { … trySend(VoiceRecognitionStatus.FinalResult(matches[0])); close() }
        override fun onError(error: Int) { trySend(VoiceRecognitionStatus.Error(mapErrorCode(error))); close() }
        …
    })
    recognizer.startListening(intent)

    awaitClose {                       // runs on cancel (user taps mic again, screen closes) and on close()
        recognizer.stopListening()
        recognizer.destroy()
    }
}
```
ViewModel: `voiceJob?.cancel(); voiceJob = viewModelScope.launch { useCase().collect { … } }` — leaving the screen cancels
`viewModelScope`, which runs `awaitClose`, which frees the recognizer. No manual cleanup anywhere.

**legacy → target:** `Error(message: String)` carries a string built in `:data` with `context.getString`. Target:
`Error(reason: VoiceError)` (sealed: `NoMatch`, `Network`, `Permission`, `Other(code)`) and texts in presentation.
**legacy → target:** the same `startRecording/stopRecording` block is copied in `AddNoteViewModel`, `EditNoteViewModel`,
`AddContactViewModel`, `ContactDetailsViewModel` — extract a `VoiceInputController` (plain class owned by the ViewModel,
taking `viewModelScope`).

---

## 9. Side effects that need a Context

**legacy** `presentation/.../feature/settings/SettingsViewModel.kt`
```kotlin
SettingsIntent.RateApp -> {
    context.openUri((AppLinks.PLAY_STORE_APP_LINK + context.packageName).toUri())   // @ApplicationContext in VM
}
```
**target**
```kotlin
// SettingsEffect.kt
internal sealed interface SettingsEffect {
    data object OpenStorePage : SettingsEffect
}

// SettingsViewModel
SettingsIntent.RateApp -> viewModelScope.launch { _effect.send(SettingsEffect.OpenStorePage) }

// SettingsScreen
LaunchedEffect(Unit) {
    viewModel.effect.collectLatest { effect ->
        when (effect) {
            SettingsEffect.OpenStorePage ->
                context.openUri((AppLinks.PLAY_STORE_APP_LINK + context.packageName).toUri())
        }
    }
}
```
*Why:* the ViewModel stays Android-free and testable; the Activity context is used for `startActivity` (no
`FLAG_ACTIVITY_NEW_TASK` hack needed).

---

## 10. Tokens instead of literals

**legacy** `presentation/.../feature/subscription/SubscriptionScreen.kt`
```kotlin
val dotSize = 8.5.dp
val dotSpacing = 6.dp
```
**target**
```kotlin
// ui/theme/Sizing.kt
val pagerDotSize: Dp = 8.5.dp,

// screen
val dotSize = MaterialTheme.sizing.pagerDotSize
val dotSpacing = MaterialTheme.spacing.space75          // 6.dp already exists
```
And in `AddNoteScreen` (reference):
```kotlin
.padding(
    start = MaterialTheme.spacing.space300,
    end = MaterialTheme.spacing.space300,
    top = MaterialTheme.spacing.space100,
    bottom = MaterialTheme.spacing.space200
)
HorizontalDivider(thickness = MaterialTheme.sizing.dividerNormal)
```

---

## 11. Settings in DataStore with defaults

`data/.../repository/AppSettingsRepositoryImpl.kt`
```kotlin
private val themeKey = stringPreferencesKey("theme")
private val fontScaleKey = intPreferencesKey("font_scale")

override fun observe(): Flow<AppSettings> = dataStore.data.map { preferences ->
    AppSettings(
        theme = preferences[themeKey]?.let { AppTheme.valueOf(it) } ?: AppSettings.Default.theme,
        fontScale = preferences[fontScaleKey]?.let { FontScale(it) } ?: AppSettings.Default.fontScale,
        …
    )
}.distinctUntilChanged()

override suspend fun updateTheme(theme: AppTheme) {
    dataStore.edit { preferences -> preferences[themeKey] = theme.name }
}
```
- Defaults in one place (`AppSettings.Default`); enums by `name`; value classes validate on read (`FontScale.init`).
- Use case per setting (`ChangeThemeUseCase`), observed once in `MainViewModel` and provided as `LocalAppSettings`.

### ❌ Not like this
```kotlin
val theme = runBlocking { dataStore.data.first()[stringPreferencesKey("theme")] } ?: "SYSTEM"   // blocking, raw strings
```
