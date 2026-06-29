# Recomposition

## What Triggers Recomposition

Compose re-runs a composable when a `State` it **reads** changes.

If a composable does not read a piece of state, it does not recompose when that state changes.

```kotlin
// GroupBody reads the full UiState — recomposes on any UiState change
@Composable
private fun GroupBody(uiState: GroupContract.UiState, onEvent: ...) { ... }

// GroupNameField reads only groupName and groupNameError
// With stability, it recomposes only when those two fields change
@Composable
private fun GroupNameField(value: String, errorText: String?, onValueChange: (String) -> Unit) { ... }
```

---

## Stability

Compose skips recomposition of a composable if its parameters haven't changed — but only when those parameters are **stable**.

| Type | Stable? |
|------|--------|
| Primitives (`String`, `Int`, `Boolean`) | ✅ Always |
| Kotlin `data class` with only stable fields | ✅ If all fields are stable |
| `List<T>`, `Map<K,V>` (standard library) | ❌ Considered unstable |
| `ImmutableList<T>` (Kotlinx Immutable Collections) | ✅ |
| Lambdas (captured variables change) | ❌ Unstable unless remembered |
| `@Stable` / `@Immutable` annotated class | ✅ If you guarantee stability |

UiState is a `data class` with primitive and nested `data class` fields — stable by default.

---

## Avoiding Lambda Instability

Lambdas that capture changing values create new instances on each recomposition, causing child recompositions.

```kotlin
// ❌ New lambda on every recomposition of GroupBody
GroupNameField(
    onValueChange = { value -> onEvent(GroupContract.Event.OnNameChanged(value)) }
)

// ✅ Stable — passes the reference directly
GroupNameField(
    onValueChange = { value -> onEvent(GroupContract.Event.OnNameChanged(value)) }
)
// Note: for this specific pattern, wrapping in remember is not needed because
// onEvent itself is a stable function reference (viewModel::event).
// Only remember lambdas that capture local variables that change.
```

When the lambda captures a local variable that changes:

```kotlin
val currentFilter = uiState.activeFilter

// ❌ New lambda whenever currentFilter changes
Button(onClick = { onEvent(Event.OnFilterChanged(currentFilter)) })

// ✅ Read uiState inside the lambda — lambda itself doesn't capture changing var
Button(onClick = { onEvent(Event.OnFilterChanged(uiState.activeFilter)) })
```

---

## Splitting Composables

Large composables read more state and recompose more often. Split into focused children that read only what they need.

```kotlin
// ❌ One big composable — any UiState change recomposes everything
@Composable
private fun GroupBody(uiState: GroupContract.UiState, onEvent: ...) {
    Column {
        Text(uiState.screenUiModel.toolbarTitle)
        AppTextField(value = uiState.groupName, ...)
        AppButton(enabled = uiState.enabledButton, ...)
        AppStatusDialog(...)
    }
}

// ✅ Split — toolbar only recomposes if title changes
@Composable
private fun GroupBody(uiState: GroupContract.UiState, onEvent: ...) {
    Scaffold(topBar = { AppToolbar(title = uiState.screenUiModel.toolbarTitle) }) { _ ->
        GroupFormContent(uiState = uiState, onEvent = onEvent)
    }
    GroupDialogState(uiState = uiState, onEvent = onEvent)
}
```

---

## `key` and `remember`

`key` restarts `remember` blocks and `LaunchedEffect` when it changes.

```kotlin
// Resets the remembered value when groupId changes
val scrollState = rememberScrollState()

// Replays the effect when the id changes
LaunchedEffect(groupId) {
    viewModel.loadGroup(groupId)
}
```

---

## What NOT to Optimize

Do not add `@Stable` annotations or wrap everything in `remember` proactively.

Profile first with Layout Inspector → Recomposition Counts. Only optimize composables that show unnecessary recompositions in practice.

Premature stability annotations that lie about actual mutability cause silent bugs.

---

## Rules

Always:

- Pass primitive params to leaf composables when possible
- Use `key` in `LazyColumn`/`LazyRow`
- Split large composables into focused children

Never:

- Annotate a class `@Stable` or `@Immutable` without verifying it is actually stable/immutable
- Add `remember` around lambdas without measuring first
- Use mutable state inside a composable that the ViewModel should own
