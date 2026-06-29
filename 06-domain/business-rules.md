# Business Rules

## Where Business Rules Live

Business rules belong in UseCases — not in ViewModels, not in repositories.

| Rule type | Example | Where |
|-----------|---------|-------|
| Input validation | Name must not be blank | UseCase |
| Authorization logic | Only admin users can delete | UseCase |
| Domain constraints | Event end must be after start | UseCase |
| Computation | Calculate discount | UseCase |
| Data fetching | Get group by ID | Repository (called by UseCase) |
| UI formatting | Show "Algebra I (A1)" | ViewModel or UiItem mapper |

---

## Encoding Rules in a UseCase

```kotlin
// domain/usecases/SaveGroupUseCase.kt
class SaveGroupUseCase @Inject constructor(
    private val groupRepository: GroupRepository,
) {
    suspend operator fun invoke(group: GroupModel): Either<Failure, Unit> {
        if (group.name.isBlank()) return Either.Error(Failure.EmptyData)
        if (group.label.length > 10) return Either.Error(Failure.UnknownError)
        return groupRepository.saveGroup(group)
    }
}
```

Rules are explicit early returns — no nested conditionals.

---

## Multiple Rules: Extract to Private Functions

```kotlin
class CreateEventUseCase @Inject constructor(
    private val eventRepository: EventRepository,
    private val groupRepository: GroupRepository,
) {
    suspend operator fun invoke(event: EventModel): Either<Failure, Unit> {
        validateEvent(event)?.let { return it }
        val groupExists = groupRepository.getGroupById(event.groupId)
        if (groupExists is Either.Error) return Either.Error(Failure.UnknownError)
        return eventRepository.createEvent(event)
    }

    private fun validateEvent(event: EventModel): Either<Failure, Unit>? {
        if (event.title.isBlank()) return Either.Error(Failure.EmptyData)
        if (event.endTime <= event.startTime) return Either.Error(Failure.UnknownError)
        return null
    }
}
```

---

## Rule Failures and Error Mapping

Each rule violation returns a specific `Failure` type. The ViewModel maps failures to user-facing messages.

```kotlin
// ViewModel
private fun mapFailureToMessage(failure: Failure): String = when (failure) {
    is Failure.EmptyData -> context.getString(R.string.error_required_fields)
    is Failure.Unauthorized -> context.getString(R.string.error_not_authorized)
    is Failure.NetworkError -> context.getString(R.string.error_network)
    else -> context.getString(R.string.error_unknown)
}
```

The UseCase never formats strings or references Android context — that stays in the presentation layer.

---

## Testing Business Rules

Rules are the most valuable thing to test. One test per rule path.

```kotlin
class SaveGroupUseCaseTest {
    private val repository = FakeGroupRepository()
    private val useCase = SaveGroupUseCase(repository)

    @Test
    fun `blank name returns EmptyData failure`() = runTest {
        val result = useCase(GroupModel(id = "", name = "  ", label = "A1", createdAt = LocalDateTime.now()))
        assertIs<Either.Error<Failure.EmptyData>>(result)
    }

    @Test
    fun `label over 10 chars returns failure`() = runTest {
        val result = useCase(GroupModel(id = "", name = "Group", label = "TOOLONGLABEL", createdAt = LocalDateTime.now()))
        assertIs<Either.Error<*>>(result)
    }

    @Test
    fun `valid group delegates to repository`() = runTest {
        val result = useCase(GroupModel(id = "", name = "Algebra I", label = "A1", createdAt = LocalDateTime.now()))
        assertIs<Either.Success<Unit>>(result)
        assertEquals(1, repository.savedGroups.size)
    }
}
```

---

## Rules

Always:

- Express each business rule as an early return in the UseCase
- Use specific `Failure` types to signal different rule violations
- Test every rule branch independently

Never:

- Put business rules in a ViewModel (`if (name.isBlank()) { ... }` in an event handler is a UseCase job)
- Put business rules in a repository
- Catch exceptions in a UseCase — let `safeApiCall` in the repository handle network errors
