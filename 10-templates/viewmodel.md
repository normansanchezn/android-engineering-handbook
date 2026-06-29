# ViewModel Template

```kotlin
// presentation/feature/xxx/viewModel/XxxViewModel.kt
package com.myapp.presentation.feature.xxx.viewModel

import android.content.Context
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.myapp.core.error.Failure
import com.myapp.core.functional.fold
import com.myapp.core.validateRequiredTextField
import com.myapp.domain.models.XxxModel
import com.myapp.domain.usecases.SaveXxxUseCase
import com.myapp.presentation.R
import dagger.hilt.android.lifecycle.HiltViewModel
import dagger.hilt.android.qualifiers.ApplicationContext
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class XxxViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val saveXxxUseCase: SaveXxxUseCase,
) : ViewModel(), XxxContract {

    private val uiState: MutableStateFlow<XxxContract.UiState> =
        MutableStateFlow(XxxContract.UiState(screenUiModel = createUiStrings()))

    private val uiEffect: MutableSharedFlow<XxxContract.Effect> = MutableSharedFlow()

    override val state: StateFlow<XxxContract.UiState> = uiState.asStateFlow()
    override val effect: SharedFlow<XxxContract.Effect> = uiEffect.asSharedFlow()

    override fun event(event: XxxContract.Event) {
        when (event) {
            is XxxContract.Event.OnFieldChanged -> validateField(event.value)
            XxxContract.Event.OnSubmitClick -> submit()
            XxxContract.Event.OnCancelClick -> sendEffect(XxxContract.Effect.NavigateBack)
            XxxContract.Event.OnDialogDismissed -> {
                updateState { it.copy(dialogState = XxxContract.DialogState.None) }
                sendEffect(XxxContract.Effect.NavigateBack)
            }
        }
    }

    private fun validateField(value: String) {
        val errorMsg = uiState.value.screenUiModel.fieldRequiredError
        updateState { ui ->
            ui.validateRequiredTextField(
                value = value,
                errorMessage = errorMsg,
                onValid = { text -> ui.copy(fieldValue = text, fieldError = null) },
                onInvalid = { error -> ui.copy(fieldValue = value, fieldError = error) }
            ).recalculateFormValidity()
        }
    }

    private fun submit() {
        updateState { it.copy(dialogState = XxxContract.DialogState.Loading) }
        viewModelScope.launch {
            saveXxxUseCase(XxxModel(name = uiState.value.fieldValue))
                .fold(
                    onSuccess = {
                        updateState {
                            it.copy(dialogState = XxxContract.DialogState.Success(name = uiState.value.fieldValue))
                        }
                    },
                    onError = { failure ->
                        updateState { state ->
                            state.copy(
                                dialogState = XxxContract.DialogState.Error(
                                    msg = mapFailureToMessage(failure, state.screenUiModel)
                                )
                            )
                        }
                    }
                )
        }
    }

    private fun XxxContract.UiState.recalculateFormValidity(): XxxContract.UiState =
        copy(enabledButton = fieldValue.isNotBlank() && fieldError.isNullOrBlank())

    private fun updateState(transform: (XxxContract.UiState) -> XxxContract.UiState) =
        uiState.update(transform)

    private fun sendEffect(effect: XxxContract.Effect) =
        viewModelScope.launch { uiEffect.emit(effect) }

    private fun mapFailureToMessage(failure: Failure, uiModel: XxxContract.XxxScreenUiModel): String =
        when (failure) {
            is Failure.NetworkError -> uiModel.networkErrorMessage
            is Failure.ServerError -> uiModel.serverErrorMessageTemplate.format(failure.code, failure.message.orEmpty()).trim()
            is Failure.UnknownError -> uiModel.unknownErrorMessage
            is Failure.EmptyData -> uiModel.emptyDataErrorMessage
            is Failure.Unauthorized -> uiModel.unauthorizedMessage
        }

    private fun createUiStrings(): XxxContract.XxxScreenUiModel =
        XxxContract.XxxScreenUiModel(
            toolbarTitle = context.getString(R.string.xxx_toolbar_title),
            screenTitle = context.getString(R.string.xxx_screen_title),
            fieldHint = context.getString(R.string.xxx_field_hint),
            saveButtonText = context.getString(R.string.xxx_save),
            cancelButtonText = context.getString(R.string.xxx_cancel),
            fieldRequiredError = context.getString(R.string.xxx_field_required),
            networkErrorMessage = context.getString(R.string.error_network),
            serverErrorMessageTemplate = context.getString(R.string.error_server),
            unknownErrorMessage = context.getString(R.string.error_unknown),
            emptyDataErrorMessage = context.getString(R.string.error_empty_data),
            unauthorizedMessage = context.getString(R.string.error_unauthorized),
            dialogSuccessTitle = context.getString(R.string.xxx_dialog_success_title),
            dialogSuccessMessage = context.getString(R.string.xxx_dialog_success_message),
            dialogErrorTitle = context.getString(R.string.xxx_dialog_error_title),
            dialogLoadingTitle = context.getString(R.string.xxx_dialog_loading_title),
            dialogLoadingMessage = context.getString(R.string.xxx_dialog_loading_message),
        )
}
```

---

## Key Patterns

| Pattern | Implementation |
|---------|---------------|
| State update | `uiState.update { it.copy(...) }` via private `updateState { }` |
| Effect emission | `viewModelScope.launch { uiEffect.emit(effect) }` via private `sendEffect()` |
| String loading | `context.getString()` in `createUiStrings()`, called once in the initial state |
| Error handling | `result.fold(onSuccess = { ... }, onError = { ... })` |
| Form validity | `recalculateFormValidity()` extension on `UiState`, called after every field update |
| Field validation | `validateRequiredTextField()` inline function from `core` |

---

## Rules

- `@ApplicationContext` for context — never Activity context
- `createUiStrings()` is called **once** in the initial state — strings don't change at runtime
- All `Failure` types must be handled in `mapFailureToMessage()` — exhaustive `when`
- `updateState` uses `MutableStateFlow.update` — atomic, thread-safe
- `sendEffect` always uses `viewModelScope.launch` — never fire-and-forget coroutines outside the scope
