# Validation

## Where Validation Lives

| Type | Location | Example |
|------|----------|---------|
| Required field | ViewModel, `validateRequiredTextField` | "Name is required" |
| Format validation | ViewModel, custom field validator | "Invalid email format" |
| Time range | ViewModel event handler | "End time must be after start time" |
| Business rules | UseCase | "Student already assigned to this group" |
| Server-side errors | `Failure.ServerError` from repository | "Group name already taken" |

---

## Field Validation (ViewModel)

The `validateRequiredTextField` inline function from `core` validates a single field and returns an updated state:

```kotlin
is Event.OnNameChanged -> {
    val errorMsg = uiState.value.screenUiModel.nameRequiredError
    updateState { ui ->
        ui.validateRequiredTextField(
            value = event.value,
            errorMessage = errorMsg,
            onValid = { text -> ui.copy(name = text, nameError = null) },
            onInvalid = { error -> ui.copy(name = event.value, nameError = error) }
        ).recalculateFormValidity()
    }
}
```

---

## Form-Wide Validity

`recalculateFormValidity()` is a private extension on `UiState`.

It recalculates `enabledButton` after every field update.

```kotlin
private fun UiState.recalculateFormValidity(): UiState {
    val hasName = name.isNotBlank()
    val hasLabel = label.isNotBlank()
    val noErrors = nameError.isNullOrBlank()
        && labelError.isNullOrBlank()
        && scheduleError.isNullOrBlank()
    return copy(enabledButton = hasName && hasLabel && noErrors)
}
```

---

## Time Range Validation

```kotlin
is Event.OnEndTimeSelected -> {
    updateState { ui ->
        val error = if (uiState.value.startTime != null && event.value <= uiState.value.startTime) {
            ui.screenUiModel.scheduleError
        } else null
        ui.copy(endTime = event.value, scheduleError = error).recalculateFormValidity()
    }
}
```

---

## Business Rule Validation (UseCase)

When the rule is a business decision, not a UI concern:

```kotlin
class SaveGroupUseCase @Inject constructor(private val repository: GroupRepository) {
    suspend operator fun invoke(group: GroupModel): Either<Failure, Unit> {
        if (group.name.isBlank()) return Either.Error(Failure.EmptyData("Name is required"))
        if (group.endTime != null && group.startTime != null && group.endTime <= group.startTime) {
            return Either.Error(Failure.EmptyData("End time must be after start time"))
        }
        return repository.save(group)
    }
}
```

UseCase validation errors arrive in the ViewModel as `Failure.EmptyData` and are displayed via `DialogState.Error`.

---

## Error Display

Inline field errors: `String?` in `UiState` — `null` means no error.

```kotlin
// In UiState
val nameError: String? = null

// In Screen
AppTextField(
    value = uiState.name,
    onValueChange = { onEvent(Event.OnNameChanged(it)) },
    errorMessage = uiState.nameError,  // null = field is valid
)
```

---

## Rules

Always:

- Validate on every keystroke — not only on submit
- Disable the submit button until all required fields pass
- Field errors (`String?`) go in `UiState`
- Business rule errors from use case go to `DialogState.Error`
- All error strings come from `screenUiModel`

Never:

- Validate inside a Composable — send an Event and let the ViewModel validate
- Show submit-time inline field errors — real-time validation prevents this
- Hardcode error messages in the ViewModel
