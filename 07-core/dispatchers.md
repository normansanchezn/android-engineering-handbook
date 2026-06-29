# Dispatchers

## Standard Dispatchers

| Dispatcher | Use |
|-----------|-----|
| `Dispatchers.Main` | UI updates, StateFlow collectors |
| `Dispatchers.IO` | Network, file I/O, database |
| `Dispatchers.Default` | CPU-intensive work (sorting, parsing large datasets) |
| `Dispatchers.Unconfined` | Tests only — runs inline, no thread switching |

---

## Default Behavior

**ViewModels** use `viewModelScope`, which defaults to `Dispatchers.Main`.

**Room** switches to `Dispatchers.IO` internally.

**Retrofit** with `suspend` functions switches to `Dispatchers.IO` internally.

For most features, you do not need to specify a dispatcher explicitly.

---

## When to Specify a Dispatcher

```kotlin
// CPU-heavy parsing — move off Main
viewModelScope.launch(Dispatchers.Default) {
    val parsed = parseLargeDataSet(rawData)
    updateState { copy(items = parsed) }
}

// File write — move to IO
viewModelScope.launch(Dispatchers.IO) {
    exportToFile(data, filePath)
}
```

---

## Injecting Dispatchers for Testability

Never hardcode `Dispatchers.IO` in a UseCase or ViewModel — inject it so tests can replace it with `StandardTestDispatcher`.

```kotlin
// core/dispatchers/AppDispatchers.kt
interface AppDispatchers {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val default: CoroutineDispatcher
}

class AppDispatchersImpl @Inject constructor() : AppDispatchers {
    override val main: CoroutineDispatcher = Dispatchers.Main
    override val io: CoroutineDispatcher = Dispatchers.IO
    override val default: CoroutineDispatcher = Dispatchers.Default
}
```

DI:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class DispatcherModule {
    @Binds
    @Singleton
    abstract fun bindAppDispatchers(impl: AppDispatchersImpl): AppDispatchers
}
```

Usage in a UseCase:

```kotlin
class ProcessDataUseCase @Inject constructor(
    private val repository: DataRepository,
    private val dispatchers: AppDispatchers,
) {
    suspend operator fun invoke(input: RawData): Either<Failure, ProcessedData> =
        withContext(dispatchers.default) {
            safeApiCall { repository.process(input) }
        }
}
```

In tests, inject `TestDispatchers` with `StandardTestDispatcher` for all three.

---

## MainDispatcherRule (Tests)

See [08-testing/unit-testing.md](../08-testing/unit-testing.md) for `MainDispatcherRule` setup.

```kotlin
@get:Rule
val mainDispatcherRule = MainDispatcherRule(StandardTestDispatcher())
```

This replaces `Dispatchers.Main` with the test dispatcher, allowing `advanceUntilIdle()` to drain all coroutines.

---

## Rules

Always:

- Inject `AppDispatchers` in anything that explicitly switches dispatchers
- Keep `viewModelScope.launch { ... }` on `Dispatchers.Main` unless there's CPU/IO-heavy work

Never:

- Hardcode `Dispatchers.IO` or `Dispatchers.Default` in production code that needs testing
- Call `withContext(Dispatchers.Main)` from a background thread to update UI — use `StateFlow.update` instead
