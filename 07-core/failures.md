# Failure Types

## Purpose

`Failure` is the error type used with `Either<Failure, T>`.

It is a sealed class that covers every error scenario the application can encounter.

---

## Definition

```kotlin
// core/error/Failure.kt
sealed class Failure {
    data class NetworkError(val exception: IOException) : Failure()
    data class ServerError(val code: Int, val message: String?) : Failure()
    data class UnknownError(val throwable: Throwable) : Failure()
    data class EmptyData(val message: String?) : Failure()
    data class Unauthorized(val message: String?) : Failure()
}
```

---

## When Each Type Is Used

| Failure | Source | When |
|---------|--------|------|
| `NetworkError` | `safeApiCall` automatic | No internet, DNS failure, timeout |
| `ServerError` | `safeApiCall` automatic | HTTP 4xx / 5xx |
| `UnknownError` | `safeApiCall` automatic | Any other exception |
| `EmptyData` | Repository manual | 200 OK but no data, empty body |
| `Unauthorized` | Repository manual | No session, expired token, 401 |

`NetworkError`, `ServerError`, and `UnknownError` are produced automatically by `safeApiCall`.

`EmptyData` and `Unauthorized` are returned manually inside the repository:

```kotlin
override suspend fun getProfile(): Either<Failure, ProfileModel> =
    safeApiCall {
        val session = authClient.currentSession()
            ?: return@safeApiCall Either.Error(Failure.Unauthorized("No active session"))

        val profile = api.getProfile()
            ?: return@safeApiCall Either.Error(Failure.EmptyData("Profile not found"))

        Either.Success(profile.toDomain())
    }
```

---

## Mapping to User Messages (ViewModel)

```kotlin
private fun mapFailureToMessage(failure: Failure, uiModel: XxxScreenUiModel): String =
    when (failure) {
        is Failure.NetworkError -> uiModel.networkErrorMessage
        is Failure.ServerError -> uiModel.serverErrorMessageTemplate
            .format(failure.code, failure.message.orEmpty())
            .trim()
        is Failure.UnknownError -> uiModel.unknownErrorMessage
        is Failure.EmptyData -> uiModel.emptyDataErrorMessage
        is Failure.Unauthorized -> uiModel.unauthorizedMessage
    }
```

All messages come from `screenUiModel` — never hardcoded in the ViewModel.

---

## Rules

Always:

- Handle every `Failure` subtype in `mapFailureToMessage()` — exhaustive `when`
- Keep `Failure` centralized in `core` — do not create feature-specific subclasses
- Use `EmptyData` when the request succeeds but returns no usable data
- Use `Unauthorized` for expired sessions before hitting the server

Never:

- Create a `Failure.ValidationError` — validation failures use `Either.Error(Failure.EmptyData(...))` or field-level state
- Expose `Failure` to the UI layer — always convert to a display string in the ViewModel
- Catch `Failure.UnknownError` silently — always show feedback to the user
