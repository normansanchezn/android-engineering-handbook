# UseCase Template

## Single Repository Call

```kotlin
// domain/usecases/group/SaveGroupUseCase.kt
package com.myapp.domain.usecases.group

import com.myapp.core.error.Failure
import com.myapp.core.functional.Either
import com.myapp.domain.models.GroupModel
import com.myapp.domain.repository.GroupRepository
import javax.inject.Inject

class SaveGroupUseCase @Inject constructor(
    private val repository: GroupRepository,
) {
    suspend operator fun invoke(group: GroupModel): Either<Failure, Unit> =
        repository.save(group)
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

## Read Use Case

```kotlin
class GetGroupsUseCase @Inject constructor(
    private val repository: GroupRepository,
) {
    suspend operator fun invoke(): Either<Failure, List<GroupModel>> =
        repository.getAll()
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

## Called from ViewModel

```kotlin
private fun loadGroups() {
    updateState { it.copy(isLoading = true) }
    viewModelScope.launch {
        getGroupsUseCase().fold(
            onSuccess = { groups ->
                updateState { it.copy(isLoading = false, groups = groups.map { it.toUiModel() }) }
            },
            onError = { failure ->
                updateState { state ->
                    state.copy(
                        isLoading = false,
                        dialogState = DialogState.Error(mapFailureToMessage(failure, state.screenUiModel))
                    )
                }
            }
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
| Send (email, push) | `SendXxxUseCase` |
| Validate | `ValidateXxxUseCase` |

---

## Rules

Always:

- `operator fun invoke()` — called as `useCase(params)` not `useCase.execute(params)`
- `suspend` — always async, never blocking
- Returns `Either<Failure, T>` — never throws
- `@Inject constructor` — injectable without a factory
- One responsibility per class — if doing two things, split into two use cases

Never:

- Import Android classes — no `Context`, no `ViewModel`, no Compose
- Call other use cases from a use case — share a repository instead
- Put UI formatting or string logic inside a use case
