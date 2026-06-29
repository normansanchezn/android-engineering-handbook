# Utilities

## What Belongs in core

The `core` module contains utilities that are used across multiple modules and have no dependency on Android framework or domain models.

| Type | Examples |
|------|---------|
| `Either` and `Failure` | `Either<Failure, T>`, `safeApiCall` |
| Extension functions | `String.orDefault()`, `List.orDefault()` |
| Dispatcher interfaces | `AppDispatchers` |
| Logging | `AppLogger` |
| Constants | Timeout values, regex patterns |

---

## Extension Functions

Place extensions in `core/extensions/`. Group by receiver type.

```kotlin
// core/extensions/StringExtensions.kt

fun String?.orDefault(default: String = ""): String = this ?: default

fun String?.isNotNullOrBlank(): Boolean = !this.isNullOrBlank()

fun String.capitalizeWords(): String =
    split(" ").joinToString(" ") { it.replaceFirstChar { c -> c.uppercase() } }
```

```kotlin
// core/extensions/ListExtensions.kt

fun <T> List<T>?.orDefault(): List<T> = this ?: emptyList()

fun <T> List<T>.orDefault(default: List<T>): List<T> = if (isEmpty()) default else this
```

```kotlin
// core/extensions/FlowExtensions.kt

fun <T> Flow<T>.onEachLog(tag: String): Flow<T> = onEach { AppLogger.d("$tag: $it") }
```

---

## validateRequiredTextField

This is a core utility used by ViewModel form validation.

```kotlin
// core/extensions/ValidationExtensions.kt

inline fun validateRequiredTextField(value: String): Boolean = value.isNotBlank()
```

Used in `UiState.recalculateFormValidity()`:

```kotlin
private fun LoginContract.UiState.recalculateFormValidity(): LoginContract.UiState {
    val emailValid = validateRequiredTextField(email)
    val passwordValid = validateRequiredTextField(password)
    return copy(enabledButton = emailValid && passwordValid)
}
```

---

## Constants

```kotlin
// core/constants/AppConstants.kt
object AppConstants {
    const val DATE_FORMAT = "MMM d, yyyy"
    const val DATETIME_FORMAT = "MMM d, yyyy HH:mm"
    const val DEBOUNCE_MS = 300L
    const val PAGE_SIZE = 20
}
```

---

## Where Not to Put Utilities

| Utility | Correct location |
|---------|----------------|
| Screen-specific helper | `presentation/feature/xxx/ui/` |
| Domain calculation | `domain/usecases/` |
| DB query helper | `data/local/` |
| String resource formatter | ViewModel (requires `Context`) |

---

## Rules

Always:

- Put utilities in `core` only if used by 2+ modules
- Group extensions by receiver type in separate files
- Keep extension functions pure — no side effects, no DI, no context

Never:

- Import `android.*` in `core/extensions/` — keep it pure Kotlin
- Add business logic to an extension function
- Put utility functions directly in a feature package if they could be reused
