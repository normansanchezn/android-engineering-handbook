# Previews

## Strategy

Every component in `design-system` needs a `@Preview`.

Previews serve two purposes: visual regression check during development, and living documentation of a component's states.

---

## Preview for Primitives

Show the default state and every meaningful variant.

```kotlin
// design-system/ui/primitives/AppButton.kt

@Preview(showBackground = true)
@Composable
private fun AppButtonPreview() {
    AppTheme {
        Column(
            modifier = Modifier.padding(SpacingM),
            verticalArrangement = Arrangement.spacedBy(SpacingS),
        ) {
            AppButton(text = "Save", onClick = {})
            AppButton(text = "Save", onClick = {}, enabled = false)
        }
    }
}
```

```kotlin
// design-system/ui/primitives/AppTextField.kt

@Preview(showBackground = true)
@Composable
private fun AppTextFieldPreview() {
    AppTheme {
        Column(
            modifier = Modifier.padding(SpacingM),
            verticalArrangement = Arrangement.spacedBy(SpacingS),
        ) {
            AppTextField(value = "Hello", onValueChange = {}, label = "Name")
            AppTextField(value = "", onValueChange = {}, label = "Name", errorText = "Name is required")
        }
    }
}
```

---

## Preview for Components

Show every `DialogState` variant independently.

```kotlin
// design-system/ui/components/AppStatusDialog.kt

@Preview
@Composable
private fun AppStatusDialogLoadingPreview() {
    AppTheme {
        AppStatusDialog(
            title = "Saving...",
            message = "Please wait",
            onDismiss = {},
            isLoading = true,
        )
    }
}

@Preview
@Composable
private fun AppStatusDialogErrorPreview() {
    AppTheme {
        AppStatusDialog(
            title = "Error",
            message = "Something went wrong. Please try again.",
            onDismiss = {},
            confirmButtonText = "OK",
        )
    }
}

@Preview
@Composable
private fun AppStatusDialogSuccessPreview() {
    AppTheme {
        AppStatusDialog(
            title = "Done",
            message = "Group saved successfully.",
            onDismiss = {},
            confirmButtonText = "Continue",
        )
    }
}
```

---

## Dark Mode Previews

Use `uiMode` to show the dark variant alongside the light one.

```kotlin
@Preview(showBackground = true, name = "Light")
@Preview(showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES, name = "Dark")
@Composable
private fun AppButtonDarkPreview() {
    AppTheme {
        AppButton(text = "Save", onClick = {})
    }
}
```

---

## Preview Placement

| Preview target | Where to put it |
|---------------|----------------|
| Primitive or component | Same file, private, at the bottom |
| Screen body (`XxxBody`) | Same file as the screen composable |
| Full screen (with ViewModel) | `@PreviewParameterProvider` or separate preview file |

---

## Rules

Always:

- Wrap every preview in `AppTheme { }`
- Mark preview functions `private`
- One preview function per state — not one big function with multiple conditional branches
- Name previews clearly: `AppButtonDisabledPreview`, not `Preview2`

Never:

- Preview a composable without `AppTheme` — colors and shapes will be wrong
- Pass real ViewModel instances to previews — use `UiState` directly
- Use `@Preview` on `internal` composables you intend to keep hidden — make it `private`
