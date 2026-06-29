# State Management

## Source of Truth

All UI state lives in the ViewModel. Never split state between the ViewModel and `remember { }` in a composable.

```
ViewModel (StateFlow<UiState>)
    ↓  collectAsStateWithLifecycle()
Screen composable
    ↓  UiState passed as param
Stateless children
```

---

## Collecting State

Use `collectAsStateWithLifecycle()` — it respects the Compose lifecycle and stops collecting when the screen is in the background.

```kotlin
@Composable
fun GroupScreen(
    viewModel: GroupViewModel = hiltViewModel(),
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()
    ...
}
```

Never use `collectAsState()` in feature screens — it ignores lifecycle.

---

## Updating State

State updates happen in the ViewModel via `updateState`:

```kotlin
// In ViewModel
private fun updateState(update: GroupContract.UiState.() -> GroupContract.UiState) {
    _state.update { it.update() }
}

// Usage
fun handleEvent(event: GroupContract.Event) {
    when (event) {
        is GroupContract.Event.OnNameChanged -> updateState {
            copy(
                groupName = event.value,
                groupNameError = null,
            ).recalculateFormValidity()
        }
    }
}
```

Each `updateState` call is a single atomic transition. No partial updates.

---

## Local UI State

Use `remember` only for UI-only ephemeral state that does not need to survive recomposition across navigation or process death.

| State | Where |
|-------|-------|
| User data (name, email, etc.) | ViewModel `UiState` |
| Loading / error / success | ViewModel `DialogState` |
| Form validity, button enabled | ViewModel `UiState` |
| Dropdown expanded | `remember { mutableStateOf(false) }` in composable |
| Scroll position | `rememberScrollState()` / `rememberLazyListState()` |
| Soft keyboard open | `LocalSoftwareKeyboardController.current` |

```kotlin
// ✅ OK: purely visual, ephemeral, no business meaning
var dropdownExpanded by remember { mutableStateOf(false) }

// ❌ Wrong: filter selection has business meaning — belongs in ViewModel
var selectedFilter by remember { mutableStateOf(Filter.All) }
```

---

## Derived State

When a value is derived from another piece of state, use `derivedStateOf` to avoid redundant recompositions.

```kotlin
val listState = rememberLazyListState()

// Only recomposes when the derived boolean changes, not on every scroll event
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}

if (showScrollToTop) {
    ScrollToTopButton(onClick = { /* scroll to top */ })
}
```

---

## State in Lists

Use `key` in `LazyColumn` to help Compose track items correctly during updates.

```kotlin
LazyColumn {
    items(
        items = uiState.groups,
        key = { group -> group.id },
    ) { group ->
        GroupCard(
            name = group.name,
            onClick = { onEvent(GroupContract.Event.OnGroupClick(group.id)) },
        )
    }
}
```

Without `key`, Compose re-renders the full list on any change. With `key`, it only recomposes changed items.

---

## Rules

Always:

- Collect state with `collectAsStateWithLifecycle()`
- Keep business state in the ViewModel
- Use `key` in every `LazyColumn`/`LazyRow`
- Use `derivedStateOf` when computing a value from scroll or list state

Never:

- Use `remember { mutableStateOf(...) }` for data the ViewModel owns
- Call `collectAsState()` instead of `collectAsStateWithLifecycle()`
- Hold a copy of ViewModel state in a local variable and mutate it independently
