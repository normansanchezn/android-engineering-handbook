# Use Cases

## Purpose

Use cases contain the business logic of the application.

They orchestrate repository calls, apply business rules, and return a typed result.

One class per action. One action per class.

---

## Standard Form

```kotlin
// domain/usecases/groups/SaveGroupUseCase.kt
class SaveGroupUseCase @Inject constructor(
    private val repository: GroupRepository,
) {
    suspend operator fun invoke(group: GroupModel): Either<Failure, Unit> =
        repository.save(group)
}
```

Called from ViewModel:

```kotlin
private fun submit() {
    updateState { it.copy(dialogState = DialogState.Loading) }
    viewModelScope.launch {
        saveGroupUseCase(GroupModel(name = uiState.value.groupName)).fold(
            onSuccess = { updateState { it.copy(dialogState = DialogState.Success(it.groupName)) } },
            onError = { f -> updateState { state -> state.copy(dialogState = DialogState.Error(mapFailure(f, state.screenUiModel))) } }
        )
    }
}
```

---

## With Business Rule Validation

```kotlin
class SaveGroupUseCase @Inject constructor(
    private val repository: GroupRepository,
) {
    suspend operator fun invoke(group: GroupModel): Either<Failure, Unit> {
        if (group.name.isBlank()) {
            return Either.Error(Failure.EmptyData("Group name is required"))
        }
        if (group.endTime != null && group.startTime != null && group.endTime <= group.startTime) {
            return Either.Error(Failure.EmptyData("End time must be after start time"))
        }
        return repository.save(group)
    }
}
```

---

## Combining Two Repositories

```kotlin
class LoadDashboardUseCase @Inject constructor(
    private val groupRepository: GroupRepository,
    private val studentRepository: StudentRepository,
) {
    suspend operator fun invoke(): Either<Failure, DashboardModel> {
        val groups = groupRepository.getAll()
        if (groups is Either.Error) return groups

        val students = studentRepository.getAll()
        if (students is Either.Error) return students

        return Either.Success(
            DashboardModel(
                groups = (groups as Either.Success).value,
                students = (students as Either.Success).value,
            )
        )
    }
}
```

---

## Naming Conventions

| Operation | Name |
|-----------|------|
| Create or update | `SaveXxxUseCase` |
| Read list | `GetXxxListUseCase` |
| Read single | `GetXxxUseCase` |
| Delete | `DeleteXxxUseCase` |
| Load combined data | `LoadXxxUseCase` |
| Send (email, notification) | `SendXxxUseCase` |
| Validate | `ValidateXxxUseCase` |

---

## Rules

Always:

- `suspend operator fun invoke()` — called as `useCase(params)`
- Returns `Either<Failure, T>` — never throws
- `@Inject constructor` — no manual factory
- One responsibility per class

Never:

- Import Android classes — no `Context`, `ViewModel`, or Compose
- Call another use case from a use case — share a repository instead
- Format strings or dates for the UI inside a use case
- Put infrastructure logic (network, DB) in a use case — that is the repository's job
