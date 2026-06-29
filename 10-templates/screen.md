# Screen Template

```kotlin
// presentation/feature/xxx/ui/XxxScreen.kt
package com.myapp.presentation.feature.xxx.ui

import androidx.activity.compose.BackHandler
import androidx.compose.foundation.layout.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.myapp.presentation.feature.xxx.viewModel.XxxContract
import com.myapp.presentation.feature.xxx.viewModel.XxxViewModel

@Composable
fun XxxScreen(
    viewModel: XxxViewModel = hiltViewModel(),
    onNavigateBack: () -> Unit,
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()

    // Collect one-time effects
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                XxxContract.Effect.NavigateBack -> onNavigateBack()
            }
        }
    }

    BackHandler { onNavigateBack() }

    // Content
    XxxBody(
        uiState = uiState,
        onEvent = viewModel::event,
    )

    // Dialogs as overlay
    XxxDialogState(
        uiState = uiState,
        onDismiss = { viewModel.event(XxxContract.Event.OnDialogDismissed) },
    )
}

@Composable
private fun XxxBody(
    uiState: XxxContract.UiState,
    onEvent: (XxxContract.Event) -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .systemBarsPadding()
            .navigationBarsPadding()
            .imePadding()
    ) {
        // Toolbar
        XxxToolbar(
            title = uiState.screenUiModel.toolbarTitle,
            onBackClick = { onEvent(XxxContract.Event.OnCancelClick) },
        )

        // Form content
        XxxFormContent(
            uiState = uiState,
            onEvent = onEvent,
        )
    }
}

@Composable
private fun XxxFormContent(
    uiState: XxxContract.UiState,
    onEvent: (XxxContract.Event) -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 24.dp, vertical = 16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        AppTextField(
            value = uiState.fieldValue,
            onValueChange = { onEvent(XxxContract.Event.OnFieldChanged(it)) },
            label = uiState.screenUiModel.fieldHint,
            errorMessage = uiState.fieldError,
            modifier = Modifier.fillMaxWidth(),
        )

        AppButton(
            text = uiState.screenUiModel.saveButtonText,
            onClick = { onEvent(XxxContract.Event.OnSubmitClick) },
            enabled = uiState.enabledButton,
            modifier = Modifier.fillMaxWidth(),
        )
    }
}

@Composable
private fun XxxDialogState(
    uiState: XxxContract.UiState,
    onDismiss: () -> Unit,
) {
    when (val dialog = uiState.dialogState) {
        XxxContract.DialogState.None -> Unit

        XxxContract.DialogState.Loading -> AppStatusDialog(
            type = StatusDialogType.LOADING,
            title = uiState.screenUiModel.dialogLoadingTitle,
            message = uiState.screenUiModel.dialogLoadingMessage,
        )

        is XxxContract.DialogState.Error -> AppStatusDialog(
            type = StatusDialogType.ERROR,
            title = uiState.screenUiModel.dialogErrorTitle,
            message = dialog.msg,
            onDismiss = onDismiss,
        )

        is XxxContract.DialogState.Success -> AppStatusDialog(
            type = StatusDialogType.SUCCESS,
            title = uiState.screenUiModel.dialogSuccessTitle,
            message = uiState.screenUiModel.dialogSuccessMessage.format(dialog.name),
            onDismiss = onDismiss,
        )
    }
}
```

---

## Structure Rules

| Composable | Visibility | Responsibility |
|-----------|-----------|---------------|
| `XxxScreen` | `public` | ViewModel injection, effect collection, BackHandler |
| `XxxBody` | `private` | Main layout structure |
| `XxxFormContent` | `private` | Form fields and submit button |
| `XxxDialogState` | `private` | Dialog rendering as overlay |
| Sub-views | `private` | Individual sections of the screen |

---

## Rules

Always:

- Root composable (`XxxScreen`) collects effects via `LaunchedEffect(Unit)`
- Root composable passes `onEvent = viewModel::event` to stateless children
- Dialog is rendered as a separate overlay composable, not inside the body
- `collectAsStateWithLifecycle()` — never `collectAsState()`

Never:

- Access `viewModel` from a child composable — pass lambdas down instead
- Call `viewModel.event()` inside `remember`, `LaunchedEffect`, or `DisposableEffect`
- Import domain models in a Screen — work with `UiState` and `UiModel` only
- Collect effects inside a child composable — only in the root `XxxScreen`
