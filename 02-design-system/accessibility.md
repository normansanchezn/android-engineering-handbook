# Accessibility

## Principles

Every screen must be usable without relying solely on color or visual position.

Three areas to cover: **semantic descriptions**, **touch targets**, **contrast**.

---

## Content Descriptions

Icon-only elements need a `contentDescription`. Text elements do not.

```kotlin
// ✅ Icon needs description
IconButton(onClick = onClose) {
    Icon(
        imageVector = Icons.Default.Close,
        contentDescription = "Close dialog",
    )
}

// ✅ Decorative icon — explicitly suppress
Icon(
    imageVector = Icons.Default.CheckCircle,
    contentDescription = null, // decorative, adjacent text conveys meaning
)

// ✅ Text is self-describing — no description needed
Text("Save changes")
```

For dynamic content, use string resources from `screenUiModel` — never hardcoded.

```kotlin
Icon(
    imageVector = Icons.Default.ArrowBack,
    contentDescription = uiState.screenUiModel.navigateBackDescription,
)
```

---

## Semantic Roles

Use `Modifier.semantics` to add meaning the accessibility tree can't infer from layout alone.

```kotlin
// Mark a custom container as a button
Box(
    modifier = Modifier
        .clickable(onClick = onClick)
        .semantics { role = Role.Button },
)

// Add a state description for a toggle
Switch(
    checked = isEnabled,
    onCheckedChange = onToggle,
    modifier = Modifier.semantics {
        contentDescription = if (isEnabled) "Notifications enabled" else "Notifications disabled"
    },
)

// Merge descendants so TalkBack reads the card as a single item
Card(
    modifier = Modifier.semantics(mergeDescendants = true) {},
) { ... }
```

---

## Touch Targets

Minimum touch target: **48dp × 48dp**.

If the visual element is smaller, expand the touch area with `minimumInteractiveComponentSize` or padding.

```kotlin
// ✅ Correct: padding ensures 48dp minimum
IconButton(
    onClick = onDelete,
    modifier = Modifier.size(48.dp),
) {
    Icon(
        imageVector = Icons.Default.Delete,
        modifier = Modifier.size(24.dp),
        contentDescription = "Delete item",
    )
}

// ❌ Wrong: 24dp icon with no padding — fails accessibility
Icon(
    imageVector = Icons.Default.Delete,
    modifier = Modifier
        .size(24.dp)
        .clickable { onDelete() },
    contentDescription = "Delete",
)
```

---

## Color Contrast

- Normal text: minimum **4.5:1** contrast ratio
- Large text (≥18sp or ≥14sp bold): minimum **3:1**
- Icons conveying meaning: minimum **3:1**

Using Material3 color pairs guarantees contrast automatically:

| Surface | Text on it | Ratio guaranteed |
|---------|-----------|-----------------|
| `primary` | `onPrimary` | ✅ |
| `surface` | `onSurface` | ✅ |
| `error` | `onError` | ✅ |
| `surfaceVariant` | `onSurfaceVariant` | ✅ |

Never convey state with color alone:

```kotlin
// ❌ Color is the only indicator
Text(text = status, color = if (active) Color.Green else Color.Red)

// ✅ Text meaning + color
Row {
    Icon(if (active) Icons.Default.CheckCircle else Icons.Default.Cancel, contentDescription = null)
    Text(text = if (active) "Active" else "Inactive")
}
```

---

## Compose-Specific Notes

- `LazyColumn` items: set `key = { item.id }` — helps accessibility tree track items during scrolls
- `AnimatedVisibility`: accessibility tree updates automatically; no extra action needed
- `Dialog`: Compose `AlertDialog` traps focus correctly — don't override `onDismissRequest` to be a no-op unless the dialog is a blocking loading state

```kotlin
// Loading dialog: block dismiss so user can't dismiss mid-operation
AppStatusDialog(
    isLoading = true,
    onDismiss = {}, // intentional no-op
)
```

---

## Rules

Always:

- `contentDescription` for every non-decorative icon
- `contentDescription = null` for explicitly decorative icons
- Touch targets ≥ 48dp
- Use M3 color pairs (`primary`/`onPrimary`) for automatic contrast

Never:

- Hardcode accessibility strings in composables — load via `screenUiModel`
- Convey state through color alone
- Override dismiss behavior on non-loading dialogs
