# Contract Interface

## Purpose

A Contract is the single source of truth for a feature's MVI types.

It is a Kotlin `interface` that extends `MVIContract` and nests `UiState`, `Effect`, `Event`, `ScreenUiModel`, and `DialogState` as inner types.

One Contract per feature screen.

---

## Anatomy

```kotlin
interface GroupContract :
    MVIContract<GroupContract.UiState, GroupContract.Effect, GroupContract.Event> {

    // ── State ──────────────────────────────────────────────────────
    data class UiState(
        val groupName: String = "",
        val groupLabel: String = "",
        val groupNameError: String? = null,
        val groupLabelError: String? = null,
        val enabledButton: Boolean = false,
        val screenUiModel: GroupScreenUiModel = GroupScreenUiModel(),
        val dialogState: DialogState = DialogState.None,
    )

    // ── Screen strings (loaded from resources in the ViewModel) ────
    data class GroupScreenUiModel(
        val toolbarTitle: String = "",
        val screenTitle: String = "",
        val screenSubtitle: String = "",
        val nameHint: String = "",
        val namePlaceholder: String = "",
        val labelHint: String = "",
        val saveButtonText: String = "",
        val cancelButtonText: String = "",
        val nameRequiredError: String = "",
        val labelRequiredError: String = "",
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

    // ── Effects (one-time commands to the UI) ──────────────────────
    sealed interface Effect {
        data object NavigateBack : Effect
    }

    // ── Events (user interactions) ─────────────────────────────────
    sealed interface Event {
        data class OnNameChanged(val value: String) : Event
        data class OnLabelChanged(val value: String) : Event
        data object OnSubmitClick : Event
        data object OnCancelClick : Event
        data object OnDialogDismissed : Event
    }

    // ── Dialog states ──────────────────────────────────────────────
    sealed class DialogState {
        data object None : DialogState()
        data object Loading : DialogState()
        data class Error(val msg: String) : DialogState()
        data class Success(val groupName: String) : DialogState()
    }
}
```

---

## File Location

```
presentation/feature/groups/
└── viewModel/
    └── GroupContract.kt
```

---

## Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Contract | `XxxContract` | `GroupContract` |
| UiState | nested `UiState` | `GroupContract.UiState` |
| Screen strings | `XxxScreenUiModel` | `GroupContract.GroupScreenUiModel` |
| Effect | nested `sealed interface Effect` | `GroupContract.Effect` |
| Event | nested `sealed interface Event` | `GroupContract.Event` |
| Dialog state | nested `sealed class DialogState` | `GroupContract.DialogState` |

---

## Rules

Always:

- Contract is a pure `interface` — no Android imports, no coroutine imports
- `UiState` is a `data class` — every field has a default value
- All UI strings go in `ScreenUiModel` — never hardcoded in the Screen
- `DialogState.None` is the default — screen shows nothing until state changes
- The ViewModel class implements the Contract: `class XxxViewModel : ViewModel(), XxxContract`

Never:

- Put logic inside the Contract
- Import `Context`, `ViewModel`, or Compose inside the Contract file
- Use `Boolean` flags for dialogs — use `DialogState` instead
- Share a Contract between two screens
