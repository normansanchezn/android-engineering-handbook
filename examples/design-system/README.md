# Example: Design System Usage

How to use the design-system module in a feature screen — theme, components, tokens, and previews.

---

## Setup: AppTheme at Root

```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                AppNavHost()
            }
        }
    }
}
```

`AppTheme` is applied once. Every composable in the tree inherits colors, typography, and shapes automatically.

---

## Using Primitives

```kotlin
@Composable
private fun GroupFormContent(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(horizontal = SpacingL, vertical = SpacingM),
        verticalArrangement = Arrangement.spacedBy(SpacingM),
    ) {
        // AppTextField — handles label + error display
        AppTextField(
            value = uiState.groupName,
            onValueChange = { onEvent(GroupContract.Event.OnNameChanged(it)) },
            label = uiState.screenUiModel.nameLabel,
            errorText = uiState.groupNameError,
        )

        AppTextField(
            value = uiState.label,
            onValueChange = { onEvent(GroupContract.Event.OnLabelChanged(it)) },
            label = uiState.screenUiModel.labelLabel,
            errorText = uiState.labelError,
        )

        Spacer(modifier = Modifier.weight(1f))

        // AppButton — disabled state driven by ViewModel
        AppButton(
            text = uiState.screenUiModel.saveButtonText,
            onClick = { onEvent(GroupContract.Event.OnSubmitClick) },
            enabled = uiState.enabledButton,
        )
    }
}
```

---

## Using AppStatusDialog

The `DialogState` sealed class drives which dialog variant appears. One `when` block handles all.

```kotlin
@Composable
private fun GroupDialogState(
    uiState: GroupContract.UiState,
    onEvent: (GroupContract.Event) -> Unit,
) {
    when (val dialog = uiState.dialogState) {
        is GroupContract.DialogState.Loading -> AppStatusDialog(
            title = uiState.screenUiModel.dialogLoadingTitle,
            message = uiState.screenUiModel.dialogLoadingMessage,
            onDismiss = {},
            isLoading = true,
        )
        is GroupContract.DialogState.Error -> AppStatusDialog(
            title = uiState.screenUiModel.dialogErrorTitle,
            message = dialog.message,
            onDismiss = { onEvent(GroupContract.Event.OnDialogDismissed) },
            confirmButtonText = uiState.screenUiModel.dialogConfirmButton,
        )
        is GroupContract.DialogState.Success -> AppStatusDialog(
            title = uiState.screenUiModel.dialogSuccessTitle,
            message = uiState.screenUiModel.dialogSuccessMessage,
            onDismiss = { onEvent(GroupContract.Event.OnDialogDismissed) },
            confirmButtonText = uiState.screenUiModel.dialogConfirmButton,
        )
        is GroupContract.DialogState.None -> Unit
    }
}
```

---

## Using Tokens Directly

When the predefined components don't fit, use tokens via `MaterialTheme`:

```kotlin
// Color token
Box(
    modifier = Modifier
        .fillMaxWidth()
        .background(MaterialTheme.colorScheme.primaryContainer)
        .padding(SpacingM),
) {
    Text(
        text = "Active",
        color = MaterialTheme.colorScheme.onPrimaryContainer,
        style = MaterialTheme.typography.labelLarge,
    )
}

// Shape token
Surface(
    shape = MaterialTheme.shapes.medium,
    color = MaterialTheme.colorScheme.surfaceVariant,
) {
    // content
}

// Spacing
Column(
    modifier = Modifier.padding(SpacingL),
    verticalArrangement = Arrangement.spacedBy(SpacingS),
) {
    // content
}
```

---

## AppToolbar in a Scaffold

```kotlin
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
        },
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

## Previews

Every screen's stateless body composable gets a preview. Wrap in `AppTheme`.

```kotlin
@Preview(showBackground = true, name = "Light — button enabled")
@Composable
private fun GroupBodyEnabledPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                groupName = "Algebra I",
                label = "A1",
                enabledButton = true,
                screenUiModel = GroupContract.GroupScreenUiModel(
                    toolbarTitle = "New Group",
                    nameLabel = "Group name",
                    labelLabel = "Label",
                    saveButtonText = "Save",
                ),
            ),
            onEvent = {},
        )
    }
}

@Preview(showBackground = true, name = "Light — with error")
@Composable
private fun GroupBodyErrorPreview() {
    AppTheme {
        GroupBody(
            uiState = GroupContract.UiState(
                groupName = "",
                groupNameError = "Name is required",
                enabledButton = false,
                screenUiModel = GroupContract.GroupScreenUiModel(
                    toolbarTitle = "New Group",
                    nameLabel = "Group name",
                    saveButtonText = "Save",
                ),
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
                screenUiModel = GroupContract.GroupScreenUiModel(
                    toolbarTitle = "New Group",
                    dialogLoadingTitle = "Saving...",
                    dialogLoadingMessage = "Please wait",
                ),
            ),
            onEvent = {},
        )
    }
}
```

---

## Anti-Patterns to Avoid

```kotlin
// ❌ Hardcoded color
Text(text = label, color = Color(0xFF49B583))

// ✅ Token
Text(text = label, color = MaterialTheme.colorScheme.primary)

// ❌ Hardcoded shape
Surface(shape = RoundedCornerShape(24.dp)) { ... }

// ✅ Token
Surface(shape = MaterialTheme.shapes.medium) { ... }

// ❌ Hardcoded spacing
Column(modifier = Modifier.padding(16.dp)) { ... }

// ✅ Named constant
Column(modifier = Modifier.padding(SpacingM)) { ... }

// ❌ Calling MaterialTheme directly in a feature
MaterialTheme(colorScheme = myScheme) { FeatureScreen() }

// ✅ AppTheme applied once at root — never inside a feature
AppTheme { AppNavHost() }
```
