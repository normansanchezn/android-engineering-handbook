# Bottom Sheet Template

## ModalBottomSheet

```kotlin
// presentation/feature/groups/ui/GroupOptionsBottomSheet.kt
@Composable
fun GroupOptionsBottomSheet(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
    onDismiss: () -> Unit,
) {
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true)

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
        containerColor = MaterialTheme.colorScheme.surface,
        dragHandle = { BottomSheetDefaults.DragHandle() },
    ) {
        GroupOptionsContent(
            uiState = uiState,
            onEvent = onEvent,
        )
    }
}

@Composable
private fun GroupOptionsContent(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = SpacingL)
            .padding(bottom = SpacingXl)
            .navigationBarsPadding(),
    ) {
        Text(
            text = uiState.screenUiModel.optionsTitle,
            style = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(vertical = SpacingM),
        )
        GroupOptionItem(
            label = uiState.screenUiModel.editLabel,
            icon = Icons.Outlined.Edit,
            onClick = { onEvent(GroupContract.Event.OnEditClick) },
        )
        GroupOptionItem(
            label = uiState.screenUiModel.deleteLabel,
            icon = Icons.Outlined.Delete,
            onClick = { onEvent(GroupContract.Event.OnDeleteClick) },
        )
    }
}
```

---

## Showing and Hiding

Control visibility through `UiState` — not local `remember` state.

```kotlin
// In UiState
data class UiState(
    val isOptionsSheetVisible: Boolean = false,
    ...
)

// In Event
sealed interface Event {
    data object OnOptionsClick : Event
    data object OnSheetDismissed : Event
}

// In ViewModel
is Event.OnOptionsClick  -> updateState { copy(isOptionsSheetVisible = true) }
is Event.OnSheetDismissed -> updateState { copy(isOptionsSheetVisible = false) }
```

Render from screen root:

```kotlin
@Composable
private fun GroupBody(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
) {
    // ... main content ...

    if (uiState.isOptionsSheetVisible) {
        GroupOptionsBottomSheet(
            uiState = uiState,
            onEvent = onEvent,
            onDismiss = { onEvent(GroupContract.Event.OnSheetDismissed) },
        )
    }
}
```

---

## Programmatic Dismiss with Animation

When dismiss must be triggered by an event (e.g., after a delete completes):

```kotlin
@Composable
fun GroupOptionsBottomSheet(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
    onDismiss: () -> Unit,
) {
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = true)
    val scope = rememberCoroutineScope()

    LaunchedEffect(uiState.isDeleteComplete) {
        if (uiState.isDeleteComplete) {
            scope.launch { sheetState.hide() }.invokeOnCompletion { onDismiss() }
        }
    }

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState,
    ) {
        GroupOptionsContent(uiState = uiState, onEvent = onEvent)
    }
}
```

---

## Rules

Always:

- Use `skipPartiallyExpanded = true` unless the design requires a half-expanded state
- Add `navigationBarsPadding()` to the content to avoid gesture navigation overlap
- Control visibility through `UiState` — not `remember { mutableStateOf(false) }`
- Pass `onDismiss` as a callback — let the parent (screen) update state

Never:

- Call `hiltViewModel()` inside a bottom sheet composable
- Put the `ModalBottomSheet` inside a child composable that also owns other content
- Use `SheetState.show()` directly from outside the sheet's composition scope
