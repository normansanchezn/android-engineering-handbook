# Compose Component Template

## Reusable Component (design-system)

Components in `design-system` are feature-agnostic. They accept only primitives, enums, and lambdas.

```kotlin
// design-system/ui/primitives/AppButton.kt
@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    type: AppButtonType = AppButtonType.Filled,
) {
    Button(
        onClick = onClick,
        modifier = modifier
            .fillMaxWidth()
            .height(56.dp),
        enabled = enabled,
        shape = RoundedCornerShape(16.dp),
        colors = ButtonDefaults.buttonColors(
            containerColor = when (type) {
                AppButtonType.Filled -> MaterialTheme.colorScheme.primary
                AppButtonType.Outlined -> Color.Transparent
            }
        )
    ) {
        Text(text = text, style = MaterialTheme.typography.labelLarge)
    }
}

enum class AppButtonType { Filled, Outlined }
```

```kotlin
@Preview(showBackground = true, name = "Filled")
@Preview(showBackground = true, name = "Disabled")
@Composable
private fun AppButtonPreview() {
    AppTheme {
        Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
            AppButton(text = "Save", onClick = {}, enabled = true)
            AppButton(text = "Save", onClick = {}, enabled = false)
            AppButton(text = "Cancel", onClick = {}, type = AppButtonType.Outlined)
        }
    }
}
```

---

## Feature Sub-View (presentation)

Feature-specific composables live in `presentation/feature/xxx/ui/` and are `internal` or `private`.

```kotlin
// presentation/feature/groups/ui/GroupFormCard.kt
@Composable
internal fun GroupFormCard(
    name: String,
    nameError: String?,
    nameHint: String,
    onNameChanged: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    Surface(
        modifier = modifier,
        shape = RoundedCornerShape(24.dp),
        tonalElevation = 1.dp,
    ) {
        Column(
            modifier = Modifier.padding(24.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp),
        ) {
            AppTextField(
                value = name,
                onValueChange = onNameChanged,
                label = nameHint,
                errorMessage = nameError,
                modifier = Modifier.fillMaxWidth(),
            )
        }
    }
}

@Preview(showBackground = true)
@Composable
private fun GroupFormCardPreview() {
    AppTheme {
        GroupFormCard(
            name = "Algebra I",
            nameError = null,
            nameHint = "Group name",
            onNameChanged = {},
        )
    }
}

@Preview(showBackground = true, name = "Error state")
@Composable
private fun GroupFormCardErrorPreview() {
    AppTheme {
        GroupFormCard(
            name = "",
            nameError = "Name is required",
            nameHint = "Group name",
            onNameChanged = {},
        )
    }
}
```

---

## Rules

Always:

- Components in `design-system` accept only primitives, enums, lambdas — never domain models or UiModels
- Feature sub-views are `internal` or `private` — not exported to other features
- Every component has at least one `@Preview` with realistic data
- `Modifier = Modifier` as default parameter — allows callers to customize layout

Never:

- Import domain or ViewModel types inside a `design-system` component
- Build a reusable component that contains business rules
- Put a `hiltViewModel()` call inside a reusable component
- Skip previews for components — they are the only way to verify visually without running the app
