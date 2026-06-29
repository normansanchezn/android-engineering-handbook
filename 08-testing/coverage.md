# Test Coverage

## What to Test

Coverage is not a number — it is a question: **do the tests catch real bugs?**

---

## Priority by Layer

| Layer | What to test | Priority |
|-------|-------------|----------|
| ViewModel | Initial state, every event, success + failure paths | High |
| UseCase | Business rules, validation logic | High |
| Repository | Only when there is non-trivial mapping or conditional logic | Medium |
| Screen | Rendered output for key states, user interactions | Medium |
| Data mapping (`toDomain`, `toDto`) | Edge cases, null handling | Low |

---

## ViewModel Test Coverage Matrix

For every ViewModel, tests should cover:

| Test area | Examples |
|-----------|---------|
| Initial state | Fields empty, button disabled, no dialog |
| Each `Event` | `OnNameChanged`, `OnSubmitClick`, `OnDialogDismissed` |
| Validation | Required field missing → error shown, button disabled |
| Submit success | Loading dialog → success dialog |
| Submit failure (each Failure type) | NetworkError, ServerError, Unauthorized |
| Effects emitted | `NavigateBack` on dismiss, `NavigateBack` on cancel |
| Form validity | All required fields valid → button enabled |

---

## UseCase Test Coverage

```kotlin
class SaveGroupUseCaseTest {

    private val repository = FakeGroupRepository()
    private val useCase = SaveGroupUseCase(repository)

    @Test
    fun `blank name returns EmptyData failure`() = runTest {
        val result = useCase(GroupModel(id = "", name = "", label = "A1"))
        assertIs<Either.Error<Failure.EmptyData>>(result)
    }

    @Test
    fun `valid group delegates to repository`() = runTest {
        val group = GroupModel(id = "", name = "Algebra I", label = "A1")
        val result = useCase(group)
        assertIs<Either.Success<Unit>>(result)
        assertEquals(1, repository.savedGroups.size)
    }

    @Test
    fun `repository failure propagates`() = runTest {
        repository.shouldFail = true
        val result = useCase(GroupModel(id = "", name = "Algebra", label = "A1"))
        assertIs<Either.Error<*>>(result)
    }
}
```

---

## What NOT to Test

- Composable rendering details (exact colors, exact padding)
- Auto-generated code (Room DAO default implementations)
- Simple data class `copy()` or `equals()`
- Framework behavior (StateFlow emissions, LiveData lifecycle)
- Private methods — test them through public behavior

---

## Rules

Always:

- Write tests for every business rule in a UseCase
- Write tests for every event handler in a ViewModel
- Cover the failure path for every operation that returns `Either<Failure, T>`
- Test effects separately from state — they use different assertion patterns

Never:

- Write tests that only verify that mock methods were called
- Write tests that duplicate what the framework already guarantees
- Skip failure path tests — they are where real bugs hide
