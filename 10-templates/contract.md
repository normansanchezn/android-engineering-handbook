# Contract Template

## Full Template

Replace `Xxx` with the feature name in PascalCase.

```kotlin
// presentation/feature/xxx/viewModel/XxxContract.kt
package com.myapp.presentation.feature.xxx.viewModel

import com.myapp.designsystem.mvi.MVIContract

interface XxxContract :
    MVIContract<XxxContract.UiState, XxxContract.Effect, XxxContract.Event> {

    data class UiState(
        val isLoading: Boolean = false,
        val fieldValue: String = "",
        val fieldError: String? = null,
        val enabledButton: Boolean = false,
        val screenUiModel: XxxScreenUiModel = XxxScreenUiModel(),
        val dialogState: DialogState = DialogState.None,
    )

    data class XxxScreenUiModel(
        val toolbarTitle: String = "",
        val screenTitle: String = "",
        val screenSubtitle: String = "",
        val fieldHint: String = "",
        val fieldPlaceholder: String = "",
        val saveButtonText: String = "",
        val cancelButtonText: String = "",
        val fieldRequiredError: String = "",
        val networkErrorMessage: String = "",
        val serverErrorMessageTemplate: String = "",
        val unknownErrorMessage: String = "",
        val emptyDataErrorMessage: String = "",
        val unauthorizedMessage: String = "",
        val dialogSuccessTitle: String = "",
        val dialogSuccessMessage: String = "",
        val dialogErrorTitle: String = "",
        val dialogLoadingTitle: String = "",
        val dialogLoadingMessage: String = "",
    )

    sealed interface Effect {
        data object NavigateBack : Effect
    }

    sealed interface Event {
        data class OnFieldChanged(val value: String) : Event
        data object OnSubmitClick : Event
        data object OnCancelClick : Event
        data object OnDialogDismissed : Event
    }

    sealed class DialogState {
        data object None : DialogState()
        data object Loading : DialogState()
        data class Error(val msg: String) : DialogState()
        data class Success(val name: String) : DialogState()
    }
}
```

---

## Minimal Template (read-only screen)

For screens that display data without form input:

```kotlin
interface XxxContract :
    MVIContract<XxxContract.UiState, XxxContract.Effect, XxxContract.Event> {

    data class UiState(
        val isLoading: Boolean = false,
        val items: List<XxxUiModel> = emptyList(),
        val errorMessage: String? = null,
        val screenUiModel: XxxScreenUiModel = XxxScreenUiModel(),
    )

    data class XxxScreenUiModel(
        val toolbarTitle: String = "",
        val emptyStateMessage: String = "",
    )

    sealed interface Effect {
        data class NavigateToDetail(val id: String) : Effect
    }

    sealed interface Event {
        data object LoadData : Event
        data class OnItemTapped(val id: String) : Event
    }
}
```

---

## Rules

- Contract is a pure `interface` — no Android or coroutine imports
- `UiState` is a `data class` — all fields have default values
- `ScreenUiModel` contains every string the screen displays
- `DialogState` is a `sealed class` (not interface) — avoids the need for `else` in `when`
- `Effect` and `Event` are `sealed interface` — allows extension if needed
