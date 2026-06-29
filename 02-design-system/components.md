# Components

## Catalog

The design-system ships two tiers of UI building blocks.

**Primitives** — atomic, single-purpose. Wrap a Material3 component with app-specific defaults.

**Components** — composed patterns built from primitives. Cover recurring UI structures that appear across features.

---

## Primitives

### AppButton

```kotlin
// design-system/ui/primitives/AppButton.kt
@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
) {
    Button(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        enabled = enabled,
        shape = MaterialTheme.shapes.small,
    ) {
        Text(
            text = text,
            style = MaterialTheme.typography.labelLarge,
        )
    }
}
```

### AppTextField

```kotlin
// design-system/ui/primitives/AppTextField.kt
@Composable
fun AppTextField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    modifier: Modifier = Modifier,
    errorText: String? = null,
    singleLine: Boolean = true,
    keyboardOptions: KeyboardOptions = KeyboardOptions.Default,
) {
    Column(modifier = modifier) {
        OutlinedTextField(
            value = value,
            onValueChange = onValueChange,
            label = { Text(label) },
            isError = errorText != null,
            singleLine = singleLine,
            keyboardOptions = keyboardOptions,
            modifier = Modifier.fillMaxWidth(),
            shape = MaterialTheme.shapes.extraSmall,
        )
        if (errorText != null) {
            Text(
                text = errorText,
                color = MaterialTheme.colorScheme.error,
                style = MaterialTheme.typography.labelSmall,
                modifier = Modifier.padding(start = SpacingS, top = 4.dp),
            )
        }
    }
}
```

### AppText

```kotlin
// design-system/ui/primitives/AppText.kt
@Composable
fun AppText(
    text: String,
    modifier: Modifier = Modifier,
    style: TextStyle = MaterialTheme.typography.bodyMedium,
    color: Color = MaterialTheme.colorScheme.onSurface,
    textAlign: TextAlign? = null,
    maxLines: Int = Int.MAX_VALUE,
) {
    Text(
        text = text,
        modifier = modifier,
        style = style,
        color = color,
        textAlign = textAlign,
        maxLines = maxLines,
        overflow = if (maxLines < Int.MAX_VALUE) TextOverflow.Ellipsis else TextOverflow.Clip,
    )
}
```

---

## Components

### AppStatusDialog

Used in every feature that has async operations. Covers Loading, Error, and Success states.

```kotlin
// design-system/ui/components/AppStatusDialog.kt
@Composable
fun AppStatusDialog(
    title: String,
    message: String,
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
    isLoading: Boolean = false,
    confirmButtonText: String? = null,
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        modifier = modifier,
        title = { Text(title, style = MaterialTheme.typography.titleMedium) },
        text = {
            if (isLoading) {
                Row(
                    horizontalArrangement = Arrangement.spacedBy(SpacingM),
                    verticalAlignment = Alignment.CenterVertically,
                ) {
                    CircularProgressIndicator(modifier = Modifier.size(24.dp))
                    Text(message, style = MaterialTheme.typography.bodyMedium)
                }
            } else {
                Text(message, style = MaterialTheme.typography.bodyMedium)
            }
        },
        confirmButton = {
            if (!isLoading && confirmButtonText != null) {
                TextButton(onClick = onDismiss) {
                    Text(confirmButtonText)
                }
            }
        },
    )
}
```

Usage from a screen's `XxxDialogState` composable:

```kotlin
@Composable
private fun XxxDialogState(
    uiState: XxxContract.UiState,
    onEvent: (XxxContract.Event) -> Unit,
) {
    when (val dialog = uiState.dialogState) {
        is XxxContract.DialogState.Loading -> AppStatusDialog(
            title = uiState.screenUiModel.dialogLoadingTitle,
            message = uiState.screenUiModel.dialogLoadingMessage,
            onDismiss = {},
            isLoading = true,
        )
        is XxxContract.DialogState.Error -> AppStatusDialog(
            title = uiState.screenUiModel.dialogErrorTitle,
            message = dialog.message,
            onDismiss = { onEvent(XxxContract.Event.OnDialogDismissed) },
            confirmButtonText = uiState.screenUiModel.dialogConfirmButton,
        )
        is XxxContract.DialogState.Success -> AppStatusDialog(
            title = uiState.screenUiModel.dialogSuccessTitle,
            message = uiState.screenUiModel.dialogSuccessMessage,
            onDismiss = { onEvent(XxxContract.Event.OnDialogDismissed) },
            confirmButtonText = uiState.screenUiModel.dialogConfirmButton,
        )
        is XxxContract.DialogState.None -> Unit
    }
}
```

### AppCard

```kotlin
// design-system/ui/components/AppCard.kt
@Composable
fun AppCard(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    content: @Composable ColumnScope.() -> Unit,
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        shape = MaterialTheme.shapes.medium,
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant,
        ),
    ) {
        Column(
            modifier = Modifier.padding(SpacingM),
            verticalArrangement = Arrangement.spacedBy(SpacingS),
            content = content,
        )
    }
}
```

### AppToolbar

```kotlin
// design-system/ui/components/AppToolbar.kt
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppToolbar(
    title: String,
    modifier: Modifier = Modifier,
    onNavigateBack: (() -> Unit)? = null,
) {
    TopAppBar(
        title = {
            Text(
                text = title,
                style = MaterialTheme.typography.titleLarge,
            )
        },
        modifier = modifier,
        navigationIcon = {
            if (onNavigateBack != null) {
                IconButton(onClick = onNavigateBack) {
                    Icon(
                        imageVector = Icons.AutoMirrored.Filled.ArrowBack,
                        contentDescription = "Navigate back",
                    )
                }
            }
        },
        colors = TopAppBarDefaults.topAppBarColors(
            containerColor = MaterialTheme.colorScheme.surface,
        ),
    )
}
```

---

## Adding a New Component

1. Decide tier: primitive (single M3 component wrapped) or component (two+ primitives composed)?
2. Place in `ui/primitives/` or `ui/components/` accordingly
3. Accept only primitives, enums, lambdas — no domain models
4. Add `@Preview` for every visual state the component can be in
5. Export it — no `internal` visibility in design-system unless truly private helper

---

## Rules

Always:

- Primitives accept only primitive types (`String`, `Boolean`, lambdas, `Modifier`)
- Components are composable wrappers — no ViewModel, no UseCase, no repository
- Every primitive and component has at least one `@Preview`
- `modifier: Modifier = Modifier` as second-to-last param (before lambdas)
- Trailing-lambda convention: content lambdas last

Never:

- Add business logic to a component
- Import domain models into design-system
- Put a feature-specific variant inside a shared component — add it to the feature's own UI package
- Duplicate a component that already exists in design-system
