# State

## Purpose

`UiState` is the complete, immutable snapshot of what the screen should render at any moment.

It lives as a nested `data class` inside the Contract interface.

---

## Structure

```kotlin
data class UiState(
    // Form fields
    val groupName: String = "",
    val groupLabel: String = "",

    // Inline validation errors (null = no error)
    val groupNameError: String? = null,
    val groupLabelError: String? = null,
    val scheduleError: String? = null,

    // Derived UI state
    val enabledButton: Boolean = false,
    val isLoading: Boolean = false,

    // Dialog visibility and content
    val dialogState: DialogState = DialogState.None,

    // All screen strings — loaded from resources in the ViewModel
    val screenUiModel: GroupScreenUiModel = GroupScreenUiModel(),
)
```

---

## ScreenUiModel

Every string the screen displays lives here. No hardcoded strings in Compose.

```kotlin
data class GroupScreenUiModel(
    val toolbarTitle: String = "",
    val screenTitle: String = "",
    val screenSubtitle: String = "",
    val nameHint: String = "",
    val saveButtonText: String = "",
    val cancelButtonText: String = "",
    val nameRequiredError: String = "",
    val networkErrorMessage: String = "",
    val serverErrorMessageTemplate: String = "",
    val unknownErrorMessage: String = "",
    val emptyDataErrorMessage: String = "",
    val unauthorizedMessage: String = "",
    val dialogSuccessTitle: String = "",
    val dialogSuccessMessageTemplate: String = "",
    val dialogErrorTitle: String = "",
    val dialogLoadingTitle: String = "",
    val dialogLoadingMessage: String = "",
)
```

The ViewModel loads all strings once in its constructor and stores them in the initial state:

```kotlin
private val uiState = MutableStateFlow(
    UiState(screenUiModel = createUiStrings())
)

private fun createUiStrings() = GroupScreenUiModel(
    toolbarTitle = context.getString(R.string.group_toolbar_title),
    screenTitle = context.getString(R.string.group_screen_title),
    nameRequiredError = context.getString(R.string.group_name_required_error),
    networkErrorMessage = context.getString(R.string.error_network),
    // ...
)
```

Screen reads strings from state:
```kotlin
Text(text = uiState.screenUiModel.toolbarTitle)
PkButton(text = uiState.screenUiModel.saveButtonText, enabled = uiState.enabledButton)
```

---

## DialogState

One sealed class per screen controls all dialog visibility. Replaces multiple boolean flags.

```kotlin
sealed class DialogState {
    data object None : DialogState()
    data object Loading : DialogState()
    data class Error(val msg: String) : DialogState()
    data class Success(val groupName: String) : DialogState()
}
```

Screen renders the dialog based on the current state:

```kotlin
when (val dialog = uiState.dialogState) {
    DialogState.None -> Unit
    DialogState.Loading -> LoadingDialog(title = uiState.screenUiModel.dialogLoadingTitle)
    is DialogState.Error -> ErrorDialog(msg = dialog.msg, onDismiss = onDismiss)
    is DialogState.Success -> SuccessDialog(name = dialog.groupName, onDismiss = onDismiss)
}
```

---

## Form Validation Pattern

`recalculateFormValidity()` is a private extension on `UiState` inside the ViewModel.

Called after every field update. Drives `enabledButton`.

```kotlin
private fun UiState.recalculateFormValidity(): UiState {
    val hasName = groupName.isNotBlank()
    val hasLabel = groupLabel.isNotBlank()
    val noErrors = groupNameError.isNullOrBlank()
        && groupLabelError.isNullOrBlank()
        && scheduleError.isNullOrBlank()
    return copy(enabledButton = hasName && hasLabel && noErrors)
}
```

---

## Rules

Always:

- `UiState` is a `data class` — always produce new instances via `copy()`
- All fields have default values — initial state is valid without arguments
- Error fields are `String?` — `null` means no error, a string means an error message
- Strings are in `screenUiModel` — never hardcoded in a Composable
- One `dialogState` field per screen — not multiple boolean flags
- Map domain lists to `List<UiModel>` before storing in state

Never:

- Store domain models directly in `UiState`
- Store DTOs in `UiState`
- Put rendering logic inside `UiState`
- Use empty string `""` as a sentinel for "no error" — use `null`
