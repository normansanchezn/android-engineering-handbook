# MVI Pattern

## Purpose

Model-View-Intent (MVI) provides a predictable, unidirectional data flow for every screen.

Every feature follows the same three-type system:

| Type | Direction | What it represents |
|------|-----------|-------------------|
| **Event** | Screen → ViewModel | A user interaction or system trigger |
| **State** | ViewModel → Screen | A complete snapshot of what to render |
| **Effect** | ViewModel → Screen | A one-time command (navigate, show toast) |

---

## MVIContract Interface

Defined once in the **design-system** module. Never duplicated.

```kotlin
fun interface MVIContractEvent<EVENT> {
    fun event(event: EVENT)
}

interface MVIContract<STATE, EFFECT, EVENT> : MVIContractEvent<EVENT> {
    val state: StateFlow<STATE>
    val effect: SharedFlow<EFFECT>
}
```

---

## How the Three Types Connect

```
Screen
  ├── collectAsStateWithLifecycle()  ───────────────────► renders State
  ├── LaunchedEffect { effect.collect {} } ──────────────► handles Effect
  └── viewModel.event(XxxEvent) ─────────────────────────► ViewModel
                                                              │
                                                         event(event)
                                                              │
  ◄──── uiState.update { ... } ◄─── private updateState {} ─┘
  ◄──── uiEffect.emit(effect)  ◄─── private sendEffect {}
```

---

## Responsibilities

| Layer | Responsibility |
|-------|---------------|
| Screen | Render State. Send Events. React to Effects |
| ViewModel | Handle Events. Update State. Emit Effects |
| UseCase | Execute business logic. Return `Either<Failure, T>` |
| Repository | Access data. Return `Either<Failure, T>` |

---

## Rules

Always:

- State is a `data class` — immutable, all changes via `copy()`
- Effects use `MutableSharedFlow` with no replay — they fire once and are gone
- State is observed via `collectAsStateWithLifecycle()` — lifecycle-aware
- Effects are collected in `LaunchedEffect(Unit)` inside the root composable
- ViewModel implements the Contract interface directly

Never:

- Call use cases from a Composable
- Put business logic inside `when (uiState)` blocks in the UI
- Use `MutableSharedFlow(replay = 1)` for navigation effects
- Share state across two ViewModels
- Expose `MutableStateFlow` or `MutableSharedFlow` publicly
