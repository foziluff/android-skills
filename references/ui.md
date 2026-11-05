# UI: Compose, MVI, navigation, theme

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

Jetpack Compose + Material 3 only (no XML layouts, no Fragments). `MainActivity` → `MainScreen` → `{App}Theme`
(reference: `DebtNoteTheme`) → `NavGraph`.

---

<a id="screen-anatomy"></a>
## Screen anatomy (MVI)

```
             processIntent(Intent)                    Lazy<UseCase>.get()(…)
  XxxScreen ───────────────────────► XxxViewModel ─────────────────────────► domain
     ▲   ▲                              │    │
     │   └── state: StateFlow<XxxState> ┘    │
     │       (collectAsStateWithLifecycle)   │
     └────── effect: Flow<XxxEffect> ────────┘   one-shot (Channel CONFLATED)
  XxxScreen ── processEvent(XxxEvent) ──► NavGraph ──► backStack
```

| File | Kind | Direction | Example |
|---|---|---|---|
| `XxxState` | `@Immutable data class` with defaults | VM → UI, continuous | `AddNoteState(noteValue, noteChecked, noteErrorUiText, isProcessing, showMaxCountDialog, …)` |
| `XxxIntent` | `sealed interface` | UI → VM | `NoteTextChanged(value)`, `SaveClicked`, `ShowMaxCountDialogChanged(visible)` |
| `XxxEffect` | `sealed interface` | VM → UI, once | `AddNoteEffect.Saved` |
| `XxxEvent` | `sealed interface` | Screen → NavGraph | `AddNoteEvent.Back`, `UpgradeSubscriptionClicked` |

*Why Effect and Event are different types:* [hypothesis] Effect is the ViewModel telling *its* screen something happened
(saved, deleted); Event is the screen asking the *outside* to navigate. The screen translates one into the other
(`AddNoteEffect.Saved -> onBack()`), so the ViewModel never knows about navigation and the NavGraph never knows about
ViewModels.

### State
`presentation/.../feature/notes/list/NotesState.kt`
```kotlin
@Immutable
internal data class NotesState(
    val searchQuery: TextFieldValue = TextFieldValue(),
    val appBarType: AppBarType = AppBarType.DEFAULT,
    val selectedNoteIds: Set<Long> = emptySet()
) {
    val isSearching = appBarType == AppBarType.SEARCH && searchQuery.text.isNotBlank()
}
```
- Every field has a default → `XxxState()` is the initial state.
- Text inputs are `TextFieldValue` (keeps cursor/selection when the VM rewrites text, e.g. voice input).
- Errors are `UiText?`, never `String`.
- Derived flags are `val`s in the body — computed once per state instance, not per recomposition.
- Domain types that Compose cannot prove stable go through `ImmutableHolder` (`filterHolder: ImmutableHolder<ContactsFilter>`).
  (legacy: `DevicesState.devices: List<Device>` holds domain objects — target: `List<DeviceUi>` via `toUi()`.)

### Intent
```kotlin
internal sealed interface AddNoteIntent {
    data class NoteTextChanged(val value: TextFieldValue) : AddNoteIntent
    data class NoteCheckedChanged(val checked: Boolean) : AddNoteIntent
    data object SaveClicked : AddNoteIntent
    data class ShowMaxCountDialogChanged(val visible: Boolean) : AddNoteIntent
    data class ShowSubscriptionDialogChanged(val visible: Boolean) : AddNoteIntent
    data object NoteVoiceClicked : AddNoteIntent
}
```
Dialogs owned by the VM (because the VM opens them on an error) are closed by an Intent (`Show…DialogChanged(false)`).

<a id="screen"></a>
### Screen
`presentation/.../feature/notes/add/AddNoteScreen.kt` (shortened)
```kotlin
@Composable
internal fun AddNoteScreen(
    processEvent: (AddNoteEvent) -> Unit,
    modifier: Modifier = Modifier,
    viewModel: AddNoteViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val keyboardController = rememberKeyboardController()
    val appSettings = LocalAppSettings.current

    val onBack = {
        keyboardController.hide()
        processEvent(AddNoteEvent.Back)
    }

    LaunchedEffect(Unit) {
        viewModel.effect.collectLatest { effect ->
            when (effect) {
                AddNoteEffect.Saved -> onBack()
            }
        }
    }

    Scaffold(
        modifier = modifier.hideKeyboard(keyboardController).fillMaxSize(),
        topBar = { AppBar(title = stringResource(R.string.new_note), onBack = onBack) }
    ) { innerPadding ->
        Column(Modifier.fillMaxSize().padding(innerPadding).consumeWindowInsets(innerPadding)) {
            AppTextField(
                value = state.noteValue,
                onValueChange = { viewModel.processIntent(AddNoteIntent.NoteTextChanged(it)) },
                errorMessage = state.noteErrorUiText?.asString(),
                maxLength = NoteInputValidator.MAX_TEXT_LENGTH,   // limit from domain, not a literal
                autofocus = appSettings.autofocus,
            )
            BottomSaveButton(
                isEnabled = !state.isProcessing,
                onClick = { viewModel.processIntent(AddNoteIntent.SaveClicked) }
            )
        }
    }

    AddNoteDialogs(state = state, processIntent = viewModel::processIntent, processEvent = processEvent, onBack = onBack)
}
```
Rules:
- Parameter order: `processEvent`, screen-specific values (`backHandlerEnabled`, `debt`, `contactId`), `modifier`,
  `viewModel` last with a default `hiltViewModel()`.
- A ViewModel with an assisted route argument has **no default**: `processEvent, viewModel: EditXxxViewModel,
  modifier = Modifier` — the NavGraph must pass `hiltViewModel(creationCallback = …)`, and a forgotten argument becomes a
  compile error instead of a runtime crash. (legacy: reference screens keep `= hiltViewModel()` for assisted ViewModels.)
- `collectAsStateWithLifecycle()` for state; effects in `LaunchedEffect(Unit)` with `collectLatest`.
- The screen body is split into `private` composables per section (`BottomSaveButton`, `TopBars`, `XxxDialogs`), each
  taking plain values + lambdas (`processIntent: (XxxIntent) -> Unit`) — never the ViewModel.
- Dialogs are rendered in one `when { state.showA -> …; state.showB -> … }` block at the end.
- Permissions are requested in the screen (`rememberLauncherForActivityResult`), the result becomes an Intent.
- Activity-bound APIs (Credential Manager, biometric prompt, share sheet) stay in the UI edge
  (`LoginScreen.googleLogin`, `util/authenticate`) and report back through Intents
  (`GoogleLoginSuccess(idToken)`, `GoogleLoginFailure`, `GoogleLoginCancelled`).
- System back: `BackHandler(enabled) { … }` first collapses search/selection, then emits `Event.Back`.
- `LifecycleEventEffect(Lifecycle.Event.ON_STOP)` to drop transient modes (selection) when leaving the app.
- No `@Preview` in the project (0 occurrences). **[proposal]** previews for `component/*` only, with fake state.

### What may live in `remember` instead of State
Purely visual, screen-local flags that no rule depends on: `var showConfirmDeletionDialog by remember { mutableStateOf(false) }`,
`rememberSaveable` for a bottom sheet's `visible`. Anything a rule or the VM needs → State.

---

<a id="navigation"></a>
## Navigation (Navigation 3)

`presentation/.../navigation/Screen.kt`
```kotlin
@Serializable
internal sealed interface Route : NavKey

internal sealed interface TopLevelScreen : Route {
    @Serializable data object Contacts : TopLevelScreen
    @Serializable data object Notes : TopLevelScreen
    @Serializable data object Reports : TopLevelScreen
    @Serializable data object Settings : TopLevelScreen

    companion object { val startTopLevelScreen = Contacts }
}

internal sealed interface Screen : Route {
    @Serializable data class EditNote(val noteId: Long) : Screen
    @Serializable data object AddNote : Screen
    @Serializable data class Calculator(val hasResultEventBus: Boolean = false) : Screen
    …
}
```
`NavGraph.kt` — the only place that touches the back stack:
```kotlin
entry<Screen.EditNote> {
    EditNoteScreen(
        processEvent = { event ->
            when (event) {
                EditNoteEvent.Back -> backStack.popBackStack()
            }
        },
        viewModel = hiltViewModel(
            creationCallback = { factory: EditNoteViewModel.Factory -> factory.create(it.noteId) }
        )
    )
}
```
Rules:
- Arguments are route properties (primitives only: ids, flags, email) — never objects.
- Arguments reach the ViewModel through `@AssistedFactory` + `hiltViewModel(creationCallback = …)`.
- Entry decorators: `rememberSaveableStateHolderNavEntryDecorator()`, `rememberViewModelStoreNavEntryDecorator()`
  (ViewModel per entry), `rememberResultEventBusNavEntryDecorator()` (results between screens).
- Back stack is `rememberNavBackStack(TopLevelScreen.startTopLevelScreen)` = `rememberSerializable(serializer())` — survives
  process death.
- Push: `backStack.navigateSingleton(screen)` (debounced by the transition duration, ignores duplicates);
  pop: `backStack.popBackStack()` (never pops the last entry); pop to: `backStack.popBackStackTo<Screen.EditPassword>()`;
  tabs: `backStack.navigateTopLevel(screen)` (start tab always at the bottom).
- Results between screens: `LocalResultEventBus.current.sendResult(value)` in the producer entry,
  `resultBus.conflateAsState<Double?>(null)` in the consumer entry, passed to the screen as a parameter (`debt = debt`).
- Top-level tabs use `metadata = navAnimationNone()`; inner screens use the shared horizontal slide specs from
  `Animations.kt`.
- Gating (auth): decided in `NavGraph` with `LocalAuthState` (`if (isAuthorized) Devices else Login`).
- (legacy: `private var lastNavTime` is a top-level mutable `var` in `Extensions.kt` — process-global state; acceptable
  only because there is one back stack.)

---

## App-wide state: CompositionLocals

`MainScreen` provides values every screen may read without a ViewModel parameter:
```kotlin
CompositionLocalProvider(
    LocalAppSettings provides state.appSettings,
    LocalAuthState provides state.isAuthorized,
    LocalUser provides state.user,
    LocalSyncState provides state.syncState,
    LocalAppConfig provides state.appConfig,
) { DebtNoteTheme { … NavGraph(…) … } }
```
- Declared in `util/CompositionLocals.kt` with `compositionLocalOf { noLocalProvidedFor("…") }` — fail fast if missing.
- Values are `@Immutable` Ui models.
- When a ViewModel needs the current value at creation, the NavGraph passes it through the assisted factory
  (`SettingsViewModel.Factory.create(appSettings = LocalAppSettings.current)`).
- *Why:* [hypothesis] settings, user and sync status are needed by almost every screen (currency symbol, Pro badge, sync
  indicator); one observer in `MainViewModel` instead of N.

---

<a id="theme-and-tokens"></a>
## Theme and tokens

`ui/theme/`:
- `Color.kt` — raw palette (`Green60`, `Gray5`, `Red50`, gradient pairs). **Only** referenced from `Theme.kt`.
- `Theme.kt` — `DarkColorScheme`/`LightColorScheme` (Material) + `AppDarkColorScheme`/`AppLightColorScheme`
  (extra roles: `link`, `warning`, `gradients`, …); `{App}Theme` chooses by `AppTheme.LIGHT/DARK/SYSTEM` and applies the
  *(optional)* user font scale (`fontScale + appSettings.fontScale.value * 0.1f`) through a custom `LocalDensity`.
- `Spacing.kt` — `space0 … space900` (`space100 = 8.dp`, `space200 = 16.dp`, `space300 = 24.dp`): the number is the
  multiple of 8dp ×100.
- `Sizing.kt` — named component sizes (`topAppBarHeight`, `navBarHeight`, `iconNormal`, `dividerNormal`, …).
- Access: `MaterialTheme.spacing.space200`, `MaterialTheme.sizing.iconNormal`, `MaterialTheme.appColorScheme.link`
  (`@Composable @ReadOnlyComposable` extension getters over `staticCompositionLocalOf`).
- Status/navigation bar appearance via `SystemBarEffect(!darkTheme)`.

Rules: a value used twice becomes a token; a new color role is added to both `AppColorScheme` instances.
(legacy raw values: `CalculatorScreen` shapes `40.dp`, `SubscriptionScreen` `8.5.dp`, `ContactsScreen` `1.dp` borders,
11 `.sp` literals.)

## Components

`presentation/component/` is the design system. Screens use `AppBar`, `AppButton`, `AppTextButton`, `AppIconButton`,
`AppTextField`, `AppCheckbox`, `AppDialog`/`DialogMessage`, `AppModalBottomSheet`, `AppDropdownMenu`, `AppTooltip`,
`PullToRefresh`, `MessageContent` (empty states), `AppWarningMessage`, `AppTopSnackbar`.

Component signature:
```kotlin
@Composable
internal fun AppButton(
    text: String,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    progress: Boolean = false,
    contentColor: Color = LocalContentColor.current,
    containerColor: Color = MaterialTheme.colorScheme.primary,
    icon: @Composable (() -> Unit)? = null,
    onClick: () -> Unit
)
```
- Required data first, `modifier` first optional, `onClick` last (trailing lambda).
- Strings come in already resolved (`stringResource` at the call site).
- **Second use → component, however small** **[confirmed]**: the skeleton's `AppBottomButton` is just a padded `AppButton`,
  `{Entity}Form` is two fields — both exist because Add and Edit use them. Cross-feature widgets → `presentation/component/`
  (no feature types in parameters); widgets shared by the screens of one area → `feature/{area}/component/`.
- Before writing a new widget, check `component/` and `feature/{area}/component/`.
- Components take resolved strings and theme-default colors/sizes as parameters; they never contain literal text, `dp`, `sp`
  or colors — values come from tokens ([conventions.md](conventions.md#dry-and-raw-values)).

## Feedback to the user

| Situation | Mechanism |
|---|---|
| Field validation | `xxxErrorUiText` in State → `AppTextField(errorMessage = …)` |
| Quota reached | `showMaxCountDialog` / `showSubscriptionDialog` → `DialogMessage` |
| Network / transient errors, confirmations ("3 notes deleted") | `SnackbarController.showMessage(UiText)` from the VM; `ObserveSnackbarEvents()` + one `SnackbarHost` in `MainScreen` |
| Empty list / empty search | `MessageContent(title, text, linkText, onLinkClick)` as a lazy item |
| Sync progress | `LocalSyncState` → `PullToRefresh(isRefreshing)` / `CircleIndicatorAnimated` |

## Accessibility

53 `contentDescription = null` vs 1 with a string resource — decorative-only icons are fine with `null`, but icon-only
buttons need a description. **[proposal]** every `AppIconButton` gets a `contentDescription: String?` parameter passed from
a string resource.
