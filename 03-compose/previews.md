# Compose Previews

## Purpose

Previews verify UI visually without running the app.

Every screen and significant component should have at least one preview.

---

## Screen Preview Strategy

Screens depend on `hiltViewModel()` — they cannot be previewed directly.

Preview the **stateless body composable** instead.

```kotlin
// GroupScreen.kt
@Composable
fun GroupScreen(
    viewModel: GroupViewModel = hiltViewModel(),
    onNavigateBack: () -> Unit,
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()
    LaunchedEffect(Unit) { viewModel.effect.collect { ... } }

    GroupBody(uiState = uiState, onEvent = viewModel::event)
    GroupDialogState(uiState = uiState, onDismiss = { viewModel.event(GroupContract.Event.OnDialogDismissed) })
}

// Private body — previewable
@Composable
private fun GroupBody(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
) { ... }

// Previews
@Preview(showBackground = true)
@Composable
private fun GroupBodyDefaultPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                screenUiModel = GroupContract.GroupScreenUiModel(
                    toolbarTitle = "New Group",
                    saveButtonText = "Save",
                    nameHint = "Group name",
                )
            ),
            onEvent = {},
        )
    }
}

@Preview(showBackground = true, name = "Filled + enabled")
@Composable
private fun GroupBodyFilledPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                name = "Algebra I",
                label = "A1",
                enabledButton = true,
            ),
            onEvent = {},
        )
    }
}

@Preview(showBackground = true, name = "Field error")
@Composable
private fun GroupBodyErrorPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                name = "",
                nameError = "Name is required",
                enabledButton = false,
            ),
            onEvent = {},
        )
    }
}

@Preview(showBackground = true, name = "Loading dialog")
@Composable
private fun GroupBodyLoadingPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                dialogState = GroupContract.DialogState.Loading,
            ),
            onEvent = {},
        )
    }
}
```

---

## Component Preview

```kotlin
@Preview(showBackground = true, name = "Enabled")
@Preview(showBackground = true, name = "Disabled")
@Composable
private fun AppButtonPreview() {
    AppTheme {
        AppButton(
            text = "Save",
            onClick = {},
            enabled = true,
        )
    }
}
```

---

## Dark / Light Preview

```kotlin
@Preview(showBackground = true, uiMode = UI_MODE_NIGHT_NO, name = "Light")
@Preview(showBackground = true, uiMode = UI_MODE_NIGHT_YES, name = "Dark")
@Composable
private fun GroupBodyThemePreview() {
    AppTheme {
        GroupBody(uiState = GroupContract.UiState(), onEvent = {})
    }
}
```

---

## Rules

Always:

- Wrap every preview in `AppTheme { }` — ensures correct colors and typography
- Preview only **stateless** composables — no `hiltViewModel()`, no `StateFlow`
- Use realistic data — not "text1", "test", or Lorem Ipsum
- Cover key states: empty/default, filled, validation error, loading, success

Never:

- Preview composables that require a ViewModel
- Suppress preview compilation errors — fix the root cause
- Skip previews for new reusable components in `design-system`
