# Tests

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

**Reference project:** only the Android Studio samples exist (`ExampleUnitTest`, `ExampleInstrumentedTest`), so this is a
**[proposal]** for it. **Skeleton:** the setup below is in place and verified — validator, use case and ViewModel tests run
green (`./gradlew :domain:test testDebugUnitTest`); the DAO test compiles for `androidTest`.

---

## The choice: JVM-first, fakes over mocks

| Level | What | Where | Runner |
|---|---|---|---|
| 1 | Validators, specifications, `Result` helpers, value classes | `domain/src/test` | JUnit 4 (already in the catalog) |
| 2 | Use cases with fake repositories | `domain/src/test` | JUnit 4 + `kotlinx-coroutines-test` |
| 3 | ViewModels (state transitions, effects, error mapping) | `presentation/src/test` | JUnit 4 + `kotlinx-coroutines-test` |
| 4 | DAOs, sync loop against an in-memory Room | `data/src/androidTest` | AndroidJUnit4 |
| 5 | Mappers (`toDomain`, `toUi`) | module `src/test` | JUnit 4 |

*Why fakes:* every dependency of a use case or ViewModel is a domain **interface** (`NoteRepository`, `SyncRepository`,
`UserRepository`, `TransactionRunner`), so a 20-line in-memory fake is simpler and more honest than a mocking library —
and it keeps philosophy #6 (no MockK/Mockito). *Trade-off:* fakes must be maintained when an interface grows.
*Why no Turbine:* `StateFlow.value` and `effect.first()` cover the existing patterns. [proposal; ask before adding libraries]

## One-time setup

`gradle/libs.versions.toml`
```toml
[versions]
# Coroutines test
coroutines = "<same version as kotlinx-coroutines pulled by the project>"

[libraries]
# Coroutines test
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }
```
`domain/build.gradle.kts`
```kotlin
dependencies {
    // Test
    testImplementation(libs.junit)
    testImplementation(libs.kotlinx.coroutines.test)
}
```
`presentation/build.gradle.kts`: `testImplementation(libs.kotlinx.coroutines.test)` (JUnit is already there).

Main dispatcher rule (presentation tests), `presentation/src/test/.../MainDispatcherRule.kt`:
```kotlin
internal class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

## Structure and naming

```
domain/src/test/java/com/wigroup/domain/
  ├─ fake/FakeNoteRepository.kt, FakeUserRepository.kt, FakeSyncRepository.kt, FakeTransactionRunner.kt
  ├─ validation/NoteInputValidatorTest.kt
  ├─ specification/CanAddNoteSpecificationTest.kt
  └─ usecase/note/CreateNoteUseCaseTest.kt
presentation/src/test/java/com/wigroup/presentation/
  └─ feature/notes/add/AddNoteViewModelTest.kt
```
- Test class: `{ClassUnderTest}Test`, `internal`.
- Method names in backticks describing behaviour: `` fun `blank text returns Empty`() ``.
- Arrange / act / assert separated by blank lines; one behaviour per test.
- Fakes live in `src/test/.../fake/` of each module that needs them (the skeleton keeps a small copy of
  `Fake{Entity}Repository` in `presentation/src/test`); move to `testFixtures` only when copies start to diverge.

## What to test (by priority)

1. **Validators & specifications** — every branch and every boundary (`MAX_TEXT_LENGTH`, `MAX_TEXT_LENGTH + 1`,
   `LIMIT_FREE - 1`, `LIMIT_FREE`, Pro vs Free).
2. **Write use cases** — input is trimmed; validator error short-circuits (repository untouched, sync not called);
   specification error; success writes and calls `sync()` exactly once.
3. **ViewModels** — each error maps to the right State field / dialog flag / snackbar; success sends the Effect;
   `isProcessing` is reset after success and failure; input caps in `processIntent`.
4. **Mappers** — money/time conversions (`Double ↔ BigDecimal`, millis ↔ `Instant`), `isPro`/`isExpired` against server time.
5. **Sync loop** (androidTest, in-memory Room + fake `ApiService`) — tombstones, batching at `LIMIT_COUNT`, upsert by local
   id then server id, acknowledgement round.

## Fake example

```kotlin
internal class FakeNoteRepository : NoteRepository {
    val notes = mutableMapOf<Long, Note>()
    private var nextId = 1L

    override suspend fun create(text: String, checked: Boolean): Long {
        val id = nextId++
        val now = Instant.EPOCH
        notes[id] = Note(id = id, text = text, isChecked = checked, createdAt = now, updatedAt = now, isSynced = false)
        return id
    }

    override suspend fun getCount(): Int = notes.size
    override fun observe(noteId: Long): Flow<Note?> = flowOf(notes[noteId])
    // other members: minimal in-memory behaviour or error("not used in this test")
    …
}

internal class FakeSyncRepository : SyncRepository {
    var syncCalls = 0
    override fun observeState(): Flow<SyncState> = emptyFlow()
    override suspend fun sync() { syncCalls++ }
}
```

## Use case test

```kotlin
internal class CreateNoteUseCaseTest {
    private val noteRepository = FakeNoteRepository()
    private val syncRepository = FakeSyncRepository()
    private val userRepository = FakeUserRepository(isPro = false)
    private val useCase = CreateNoteUseCase(
        userRepository = userRepository,
        noteRepository = noteRepository,
        validator = NoteInputValidator(),
        specification = CanAddNoteSpecification(),
        syncRepository = syncRepository,
    )

    @Test
    fun `blank text returns Empty and does not write`() = runTest {
        val result = useCase(CreateNoteUseCase.Params(text = "   ", checked = false))

        assertEquals(Result.error(NoteError.Empty), result)
        assertEquals(0, noteRepository.notes.size)
        assertEquals(0, syncRepository.syncCalls)
    }

    @Test
    fun `free user over the limit gets Limit`() = runTest {
        repeat(CanAddNoteSpecification.LIMIT_FREE) { noteRepository.create("n$it", false) }

        val result = useCase(CreateNoteUseCase.Params(text = "one more", checked = false))

        assertEquals(Result.error(NoteError.Limit), result)
    }
}
```

## ViewModel test

```kotlin
internal class AddNoteViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `blank save shows field error and resets progress`() = runTest {
        val viewModel = AddNoteViewModel(
            createNoteUseCase = Lazy { createNoteUseCase() },     // dagger.Lazy is a Java SAM interface
            startVoiceListenerUseCase = Lazy { StartVoiceListenerUseCase(FakeVoiceRepository()) },
        )

        viewModel.processIntent(AddNoteIntent.SaveClicked)

        val state = viewModel.state.value
        assertNotNull(state.noteErrorUiText)
        assertFalse(state.isProcessing)
    }

    @Test
    fun `successful save emits Saved`() = runTest {
        val viewModel = …
        viewModel.processIntent(AddNoteIntent.NoteTextChanged(TextFieldValue("Buy milk")))

        viewModel.processIntent(AddNoteIntent.SaveClicked)

        assertEquals(AddNoteEffect.Saved, viewModel.effect.first())
    }
}
```
Note: `UiText.StringResource` compares by reference (it is a plain `class` with `vararg`), so assert on presence or on
`(state.noteErrorUiText as UiText.StringResource).id`.

- `androidTest` method names are camelCase (no backticks with spaces — not supported on older ART).

## Before committing

```bash
./gradlew :domain:test testDebugUnitTest
./gradlew connectedDebugAndroidTest   # only when DAO/sync tests changed; needs a device/emulator
```
