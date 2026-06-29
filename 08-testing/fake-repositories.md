# Fake Repositories

## Purpose

Fake repositories are in-memory implementations of repository interfaces used in tests.

They replace the real data source with controllable behavior — success responses, error responses, loading delays.

Prefer fakes over mocks for repositories: fakes give you a real implementation that verifies the contract, mocks only verify calls.

---

## Standard Fake Pattern

```kotlin
// test/fake/FakeGroupRepository.kt
class FakeGroupRepository : GroupRepository {

    // Configure behavior from the test
    var shouldFail = false
    var groups: MutableList<GroupModel> = mutableListOf()

    override suspend fun getAll(): Either<Failure, List<GroupModel>> {
        if (shouldFail) return Either.Error(Failure.NetworkError(IOException("Test network error")))
        return Either.Success(groups.toList())
    }

    override suspend fun save(group: GroupModel): Either<Failure, Unit> {
        if (shouldFail) return Either.Error(Failure.ServerError(500, "Server error"))
        groups.add(group.copy(id = UUID.randomUUID().toString()))
        return Either.Success(Unit)
    }

    override suspend fun delete(id: String): Either<Failure, Unit> {
        if (shouldFail) return Either.Error(Failure.UnknownError(RuntimeException("Test error")))
        groups.removeIf { it.id == id }
        return Either.Success(Unit)
    }
}
```

---

## Fake With Multiple Failure Scenarios

```kotlin
class FakeGroupRepository : GroupRepository {

    sealed class Behavior {
        data object Success : Behavior()
        data object NetworkError : Behavior()
        data class ServerError(val code: Int) : Behavior()
        data object Unauthorized : Behavior()
        data object EmptyData : Behavior()
    }

    var behavior: Behavior = Behavior.Success
    val savedGroups = mutableListOf<GroupModel>()

    override suspend fun save(group: GroupModel): Either<Failure, Unit> = when (behavior) {
        Behavior.Success -> { savedGroups.add(group); Either.Success(Unit) }
        Behavior.NetworkError -> Either.Error(Failure.NetworkError(IOException()))
        is Behavior.ServerError -> Either.Error(Failure.ServerError((behavior as Behavior.ServerError).code, null))
        Behavior.Unauthorized -> Either.Error(Failure.Unauthorized("Token expired"))
        Behavior.EmptyData -> Either.Error(Failure.EmptyData("No data"))
    }

    override suspend fun getAll(): Either<Failure, List<GroupModel>> = when (behavior) {
        Behavior.Success -> Either.Success(savedGroups.toList())
        else -> Either.Error(Failure.NetworkError(IOException()))
    }

    override suspend fun delete(id: String): Either<Failure, Unit> = when (behavior) {
        Behavior.Success -> { savedGroups.removeIf { it.id == id }; Either.Success(Unit) }
        else -> Either.Error(Failure.NetworkError(IOException()))
    }
}
```

---

## Fake Gateway (non-repository dependency)

For dependencies that are not repositories (auth gateways, analytics, etc.):

```kotlin
private class FakeAuthGateway : AuthGateway {
    var currentSession: UserSession? = null
    val sessionStatusFlow = MutableSharedFlow<SessionStatus>(replay = 1)

    override fun currentSessionOrNull(): UserSession? = currentSession
    override val sessionStatus: Flow<SessionStatus> get() = sessionStatusFlow
}
```

---

## Usage in Tests

```kotlin
override fun beforeTest() {
    repository = FakeGroupRepository()
}

@Test
fun `save failure shows error dialog`() = runTest(mainDispatcherRule.dispatcher) {
    repository.shouldFail = true
    val viewModel = buildViewModel()

    viewModel.event(GroupContract.Event.OnNameChanged("Algebra"))
    viewModel.event(GroupContract.Event.OnSubmitClick)
    advanceUntilIdle()

    assertIs<GroupContract.DialogState.Error>(viewModel.state.value.dialogState)
}

@Test
fun `save success shows success dialog with group name`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()

    viewModel.event(GroupContract.Event.OnNameChanged("Algebra I"))
    viewModel.event(GroupContract.Event.OnSubmitClick)
    advanceUntilIdle()

    val dialog = viewModel.state.value.dialogState
    assertIs<GroupContract.DialogState.Success>(dialog)
    assertEquals("Algebra I", dialog.name)
}
```

---

## File Location

```
presentation/src/test/
└── fake/
    ├── FakeGroupRepository.kt
    ├── FakeStudentRepository.kt
    └── FakeAuthGateway.kt
```

---

## Rules

Always:

- Fakes implement the real repository interface — they must compile against the contract
- Fakes default to a success response — tests opt in to failure via `shouldFail = true` or `behavior = ...`
- Fakes are thread-safe for test use (single-threaded by design in `runTest`)

Never:

- Use `mockk<GroupRepository>()` for repositories in ViewModel tests — use a Fake
- Make fakes depend on Android, network, or database
- Share mutable fake state across tests — reset in `beforeTest()`
