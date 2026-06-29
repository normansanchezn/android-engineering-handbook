# Compose Principles

## Mental Model

Compose is declarative: you describe **what** the UI looks like for a given state, not **how** to transition to it.

When state changes, Compose re-runs the composable functions that read that state and redraws only what changed. This is **recomposition**.

Your job: keep composables stateless and push state up.

---

## Composable Anatomy

```kotlin
@Composable
fun GroupScreen(
    viewModel: GroupViewModel = hiltViewModel(),
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is GroupContract.Effect.NavigateBack -> navController.navigateUp()
            }
        }
    }

    GroupBody(
        uiState = uiState,
        onEvent = viewModel::event,
    )
}

@Composable
private fun GroupBody(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
) {
    Scaffold(
        topBar = {
            AppToolbar(
                title = uiState.screenUiModel.toolbarTitle,
                onNavigateBack = { onEvent(GroupContract.Event.OnBackClick) },
            )
        }
    ) { padding ->
        GroupFormContent(
            uiState = uiState,
            onEvent = onEvent,
            modifier = Modifier.padding(padding),
        )
    }
    GroupDialogState(uiState = uiState, onEvent = onEvent)
}
```

---

## Stateless vs Stateful

| Composable | Holds state? | Role |
|-----------|-------------|------|
| `XxxScreen` | Yes — `collectAsStateWithLifecycle()` | Entry point; injects ViewModel |
| `XxxBody` | No | Renders UiState; testable |
| `XxxFormContent` | No | Form sub-section |
| `XxxDialogState` | No | Dialog rendering |

The root screen is the only stateful composable. Everything below it is stateless.

---

## State Hoisting

State and callbacks flow down. Events flow up.

```kotlin
// ✅ Hoisted — Body has no state
GroupBody(
    uiState = uiState,           // state flows down
    onEvent = viewModel::event,  // events flow up
)

// ❌ Stateful body — breaks testability
@Composable
private fun GroupBody(viewModel: GroupViewModel) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()
    ...
}
```

---

## Effect Handling

Effects are one-shot actions (navigate, show Snackbar). Collect them in `LaunchedEffect`.

```kotlin
LaunchedEffect(Unit) {
    viewModel.effect.collect { effect ->
        when (effect) {
            is GroupContract.Effect.NavigateBack -> navController.navigateUp()
            is GroupContract.Effect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
        }
    }
}
```

Use `LaunchedEffect(Unit)` not `LaunchedEffect(key)` — effects must be collected for the full screen lifetime, not restarted on state changes.

---

## Modifier Convention

- Always accept `modifier: Modifier = Modifier` as the second parameter (after required data params, before lambdas)
- Apply the modifier to the outermost layout of the composable
- Never apply multiple modifiers at the call site that conflict with the default — the caller overrides by passing their own `Modifier`

```kotlin
@Composable
fun AppCard(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,        // second-to-last, before content lambda
    content: @Composable ColumnScope.() -> Unit,
) {
    Card(modifier = modifier.fillMaxWidth()) { ... }  // modifier on outermost
}
```

---

## Rules

Always:

- One stateful composable per feature (the `XxxScreen` root)
- Pass `UiState` and `onEvent` lambda to stateless children
- Collect effects in `LaunchedEffect(Unit)` at the root screen level
- Apply `modifier` to the outermost layout element

Never:

- Call `hiltViewModel()` inside a child composable
- Store business state with `remember { mutableStateOf(...) }` in a screen — it lives in the ViewModel
- Use `DisposableEffect` for one-shot events — use `LaunchedEffect`
- Nest composables more than three levels without extracting a named function
