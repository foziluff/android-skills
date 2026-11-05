# Errors: from the network to the screen

> Paths and code below come from the reference project DebtNote (`com.wigroup.*`) as evidence; the rules apply to any
> app built with this skill. *(if the app …)* / *(optional)* marks capabilities only some apps have. A verified universal
> version of the base code is in [templates/project](../templates/project) and [templates/feature](../templates/feature).

## The model

`domain/.../value/Result.kt`
```kotlin
sealed interface Result<out T, out E : RootError> {
    data class Success<out T>(val value: T) : Result<T, Nothing>
    data class Error<out E : RootError>(val error: E) : Result<Nothing, E>

    companion object {
        fun <T> success(value: T): Result<T, Nothing> = Success(value)
        fun <E : RootError> error(error: E): Result<Nothing, E> = Error(error)
    }
}

inline fun <T, E : RootError> Result<T, E>.onSuccess(action: (T) -> Unit): Result<T, E>
inline fun <T, E : RootError> Result<T, E>.onError(action: (E) -> Unit): Result<T, E>
inline fun <T, R, E : RootError> Result<T, E>.map(transform: (T) -> R): Result<R, E>
inline fun <T, E : RootError, R : RootError> Result<T, E>.mapError(transform: (E) -> R): Result<T, R>
fun <T, E : RootError> Result<T, E>.getValue(): T?
```
- Own type, not `kotlin.Result`: the error side is typed (`E : RootError`), not `Throwable`. [hypothesis]
- `onSuccess`/`onError` are `inline`, so `return` from the enclosing function works inside them
  (`LoginUseCase`: `.onError { error -> tokenGateway.clearToken(); return Result.error(LoginError.Network(error)) }`).
- Construct with `Result.success(x)` / `Result.error(e)` (legacy: direct `Result.Error(...)` in `LoginWithGoogleUseCase`).

## Error types

`domain/.../error/`
```kotlin
interface RootError

sealed interface NetworkError : RootError {
    data object UnknownHost : NetworkError
    data object SocketTimeout : NetworkError
    data object Connect : NetworkError
    data object IO : NetworkError
    data class Network(val code: Int) : NetworkError      // HTTP status
    data object Any : NetworkError
}

sealed interface NoteError : RootError {
    data object Empty : NoteError
    data object MaxLength : NoteError
    data object MaxCount : NoteError      // hard cap (Pro limit) → "limit reached" dialog
    data object Limit : NoteError         // Free limit → "upgrade" dialog
}

sealed interface LoginError : RootError {
    data object LoginMinLength : LoginError
    …
    data class Network(val error: NetworkError) : LoginError
}
```
Rules:
- One `sealed interface {Area}Error : RootError` per area/form; cases are `data object`s named by the problem
  (`Empty`, `{Field}MinLength`, `{Field}MaxLength`, `{Field}Symbol`, `{Field}Format`, `MaxCount`, `Limit`, `ListEmpty`, `DateRange`).
- Network failures of a scenario are wrapped, not mixed in: `data class Network(val error: NetworkError)`.
- A use case that combines areas returns `Result<T, RootError>` (`CreateContactWithDebtUseCase`) and the ViewModel
  `when`s over both `ContactError.*` and `DebtError.*`.
- Errors carry **no text**. Text is a presentation concern.

## Where errors are produced

| Layer | Produces | How |
|---|---|---|
| `data/remote` | `NetworkError` | `safeApiCall` converts exceptions and non-2xx |
| `data/repository` | `{Area}Error.Network` | `.mapError { LoginError.Network(it) }` |
| `domain/validation` | `{Area}Error?` | `when { … -> Error; else -> null }` |
| `domain/specification` | `{Area}Error.MaxCount/Limit` | `check(currentCount, isPro)` |
| `domain/usecase` | `Result.error(it)` | early `?.let { return Result.error(it) }` |

Exceptions are allowed only for programming errors (`require(value in -3..3)` in `FontScale`, `requireNotNull(body)`,
`error("CompositionLocal … not present")`).

## Mapping to the screen

In the ViewModel, exhaustively:
```kotlin
.onError { error ->
    when (error) {
        NoteError.Empty -> _state.update { it.copy(noteErrorUiText = UiText.StringResource(R.string.field_required)) }
        NoteError.MaxCount -> _state.update { it.copy(showMaxCountDialog = true) }
        NoteError.Limit -> _state.update { it.copy(showSubscriptionDialog = true) }
        NoteError.MaxLength -> {}          // impossible: input is capped in processIntent
    }
}
```
Network errors go through one helper with a per-screen code mapping:
```kotlin
getNetworkMessage(it.error) { code ->
    when (code) {
        401 -> UiText.StringResource(R.string.error_invalid_credentials)
        429 -> UiText.StringResource(R.string.error_too_many_requests)
        else -> UiText.StringResource(R.string.error_unknown)
    }
}?.let { SnackbarController.showMessage(it) }
```
`getNetworkMessage` (`presentation/util/Utils.kt`) maps transport errors once: `UnknownHost → error_no_internet`,
`SocketTimeout → error_timeout`, `Connect → error_connection_refused`, `IO → error_io_connection`, `Any → error_unknown`.
**[proposal]** name HTTP codes (`HttpCode.UNAUTHORIZED = 401`, `TOO_MANY_REQUESTS = 429`) instead of literals.

| Error kind | UI |
|---|---|
| Field validation | `xxxErrorUiText` under the field; cleared on the next change of that field |
| Quota | dialog flag in State |
| Network / unexpected | snackbar via `SnackbarController` |
| Loading a screen's data failed | `error: UiText?` in State + retry intent (`DevicesIntent.LoadDevices`) |
| Session expired (401 in sync or devices) | `LogoutUseCase()` + snackbar `error_session_expired` (+ `Effect.Back`) |

## `UiText`

`presentation/.../util/UiText.kt`
```kotlin
internal sealed interface UiText {
    data class DynamicString(val value: String) : UiText
    class StringResource(@StringRes val id: Int, vararg val formatArgs: Any) : UiText
    class PluralResource(@PluralsRes val id: Int, val count: Int, vararg val formatArgs: Any) : UiText

    @Composable @ReadOnlyComposable
    fun asString(): String
    fun asString(context: Context): String
}
```
- ViewModels never resolve strings (no `Context`); they create `UiText`, the UI calls `asString()`.
- Format args are domain constants: `UiText.StringResource(R.string.error_min_length, LoginInputValidator.MIN_LOGIN_LENGTH)`.
- `DynamicString` only for text that already comes localized (legacy: speech-recognizer messages from `:data`).

## Logging and crash reporting

- There is **no logging** in the project: no `Log`, no Timber, no Crashlytics (Firebase Analytics and Messaging only).
  [hypothesis: nothing personal ever reaches logcat]
- legacy: `printStackTrace()` in `CryptoManager.decrypt`, `MyFirebaseMessagingService.getBitmap`, `PdfGenerator`,
  `Context.openUri` — remove in touched code.
- **[proposal]** if diagnostics are needed: Firebase Crashlytics (already on the Firebase BoM), non-fatal reports with an
  error *type* only (`NetworkError.Any`, HTTP code) — never names, amounts, emails, tokens.

## Where swallowing is allowed

- `FocusRequester.requestFocusCatch()` (focus before attach).
- `setForeground()` `IllegalStateException` in `SyncWorker` (background start restrictions).
- `Context.getAppVersionCode()` → `0L` on failure.
Everything else returns an error value.
