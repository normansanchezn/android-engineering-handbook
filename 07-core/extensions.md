# Extensions

## Purpose

The `core` module provides small, generic extension functions reused across the project.

These extensions must be framework-light — no Android, no Compose, no domain knowledge.

---

## Nullable Helpers

```kotlin
// core/functional/Ext.kt

fun Boolean?.orDefault(default: Boolean = false): Boolean = this ?: default

fun <T, R> List<T>?.mapOrDefault(
    defaultListValue: List<R> = emptyList(),
    transform: (T) -> R,
): List<R> = this?.map { transform(it) } ?: defaultListValue
```

Usage:

```kotlin
val isActive = user?.isActive.orDefault(false)
val items = response.data?.mapOrDefault { it.toDomain() } ?: emptyList()
```

---

## Inline Field Validation

```kotlin
// core/InlineFunctions.kt

inline fun <STATE> STATE.validateRequiredTextField(
    value: String,
    errorMessage: String,
    crossinline onValid: STATE.(String) -> STATE,
    crossinline onInvalid: STATE.(String) -> STATE,
): STATE = if (value.isBlank()) onInvalid(errorMessage) else onValid(value)
```

Usage in ViewModel:

```kotlin
updateState { ui ->
    ui.validateRequiredTextField(
        value = event.value,
        errorMessage = ui.screenUiModel.nameRequiredError,
        onValid = { text -> ui.copy(name = text, nameError = null) },
        onInvalid = { error -> ui.copy(name = event.value, nameError = error) }
    ).recalculateFormValidity()
}
```

---

## Where Extensions Belong

| Extension type | Module |
|---------------|--------|
| Generic Kotlin (nullable, collections) | `core` |
| Android-specific (`Context`, `Resources`) | `presentation` or `data` |
| Compose (`Modifier`, `Color`) | `design-system` |
| ViewModel/scope helpers | `presentation` |
| DTO/entity mapping | next to the DTO file in `data` |
| Domain model helpers | next to the model file in `domain` |

---

## Rules

Always:

- Extensions in `core` compile without any project-specific imports
- Name extensions after what they do, not who calls them

Never:

- Add an extension to `core` that imports Android, Compose, or domain classes
- Add feature-specific logic to `core` — it must stay generic
- Create an extension just to save a few characters — clarity over brevity
