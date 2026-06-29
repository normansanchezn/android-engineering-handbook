# Either — Result Type

## Purpose

`Either<L, R>` represents the result of an operation that can succeed or fail.

It replaces exceptions as the primary error-handling mechanism across the entire application.

---

## Definition

```kotlin
// core/functional/Either.kt
sealed class Either<out L, out R> {
    data class Error<out T>(val value: T) : Either<T, Nothing>()
    data class Success<out T>(val value: T) : Either<Nothing, T>()
}

inline fun <L, R, T> Either<L, R>.fold(
    onError: (L) -> T,
    onSuccess: (R) -> T,
): T = when (this) {
    is Either.Error -> onError(value)
    is Either.Success -> onSuccess(value)
}
```

---

## Creating Results

```kotlin
// Success
return Either.Success(model)
return Either.Success(Unit)     // for operations with no return value

// Error
return Either.Error(Failure.NetworkError(exception))
return Either.Error(Failure.EmptyData("No data found"))
return Either.Error(Failure.Unauthorized("Session expired"))
```

---

## Consuming Results in ViewModel

```kotlin
viewModelScope.launch {
    saveGroupUseCase(GroupModel(name = uiState.value.groupName))
        .fold(
            onSuccess = {
                updateState { it.copy(dialogState = DialogState.Success(it.groupName)) }
            },
            onError = { failure ->
                updateState { state ->
                    state.copy(
                        dialogState = DialogState.Error(
                            mapFailureToMessage(failure, state.screenUiModel)
                        )
                    )
                }
            }
        )
}
```

---

## safeApiCall

`safeApiCall` wraps throwing code — network calls, SDK methods — and converts exceptions into `Either<Failure, T>`.

### For generic clients (throws on error)

```kotlin
// data/repository
override suspend fun save(group: GroupModel): Either<Failure, Unit> =
    safeApiCall {
        api.saveGroup(group.toDto())
        Unit
    }
```

### For Retrofit (Response<T>)

```kotlin
override suspend fun getAll(): Either<Failure, List<GroupModel>> =
    safeApiCall(
        apiCall = { api.getGroups() },
        mapper = object : ResultMapper<GroupListResponse, List<GroupModel>> {
            override fun map(input: GroupListResponse) = input.items.map { it.toDomain() }
        }
    )
```

### Internal implementation

```kotlin
// core/remote/handler/NetworkHandler.kt
suspend fun <R> safeApiCall(
    ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
    apiCall: suspend () -> R,
): Either<Failure, R> =
    withContext(ioDispatcher) {
        runCatching { Either.Success(apiCall()) }
    }.getOrElse { it.toEither() }

// Throwable.toEither() maps exceptions to Failure:
fun Throwable.toEither(): Either<Failure, Nothing> = when (this) {
    is IOException -> Either.Error(Failure.NetworkError(this))
    is HttpException -> Either.Error(Failure.ServerError(code(), message()))
    else -> Either.Error(Failure.UnknownError(this))
}
```

---

## Rules

Always:

- Repository methods return `Either<Failure, T>`
- Use cases return `Either<Failure, T>`
- ViewModel consumes `Either` via `fold()` — never pattern match directly
- `safeApiCall` wraps every network call — never use try/catch directly in a repository

Never:

- Throw exceptions from repositories or use cases
- Use Kotlin's built-in `Result<T>` — use `Either<Failure, T>` for consistency
- Catch exceptions in the ViewModel — let `safeApiCall` and `toEither()` handle them
