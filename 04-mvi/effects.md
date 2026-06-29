# Effects

## Purpose

Effects are one-time commands from the ViewModel to the Screen.

They represent actions that cannot be expressed as state: navigation, toasts, scroll commands, system dialogs.

They flow in one direction: **ViewModel → Screen**, and are never replayed.

---

## Definition

Effects are a `sealed interface` nested inside the Contract:

```kotlin
sealed interface Effect {
    data object NavigateBack : Effect
    data object NavigateToHome : Effect
    data class NavigateToDetail(val id: String) : Effect
    data class ShowToast(val message: String) : Effect
}
```

---

## ViewModel — Emitting Effects

```kotlin
private val uiEffect = MutableSharedFlow<Effect>()
override val effect: SharedFlow<Effect> = uiEffect.asSharedFlow()

private fun sendEffect(effect: Effect) {
    viewModelScope.launch { uiEffect.emit(effect) }
}
```

Usage:

```kotlin
Event.OnCancelClick -> sendEffect(Effect.NavigateBack)

Event.OnDialogDismissed -> {
    updateState { it.copy(dialogState = DialogState.None) }
    sendEffect(Effect.NavigateToHome)
}
```

---

## Screen — Collecting Effects

```kotlin
LaunchedEffect(Unit) {
    viewModel.effect.collect { effect ->
        when (effect) {
            Effect.NavigateBack -> onNavigateBack()
            Effect.NavigateToHome -> onNavigateToHome()
            is Effect.NavigateToDetail -> onNavigateToDetail(effect.id)
            is Effect.ShowToast -> Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

---

## State vs Effect — When to Use Which

| Scenario | Use |
|----------|-----|
| Show or hide a dialog | State (`dialogState`) |
| Enable or disable a button | State (`enabledButton`) |
| Display an error message | State (`errorMessage`) or `DialogState.Error` |
| Navigate to another screen | Effect |
| Show a toast or snackbar | Effect |
| Clear all fields after success | State (`copy(name = "", label = "")`) |
| Scroll to top | Effect |
| Open a system picker or sheet | Effect |

---

## Rules

Always:

- Effects use `MutableSharedFlow()` — default, no replay
- `sendEffect` wraps emission in `viewModelScope.launch`
- Effect collection lives in `LaunchedEffect(Unit)` in the root composable

Never:

- Use `replay = 1` for navigation effects — this re-triggers on recomposition
- Navigate using state (`val shouldNavigate: Boolean`) — use an Effect instead
- Collect effects inside a child composable — only in the root screen
- Emit effects outside of `viewModelScope`
