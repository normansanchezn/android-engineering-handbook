# Motion & Animations

## Philosophy

Animations communicate state transitions. They should be fast, purposeful, and invisible when the user is focused on content.

Default: no animation. Add animation when it reduces cognitive load or clarifies what changed.

---

## AnimatedVisibility

Content appears or disappears:

```kotlin
AnimatedVisibility(
    visible = uiState.isErrorVisible,
    enter = fadeIn() + expandVertically(),
    exit = fadeOut() + shrinkVertically(),
) {
    ErrorBanner(text = uiState.errorMessage)
}
```

Default transitions (fade + expand/shrink) cover most cases. Only specify custom transitions when the design spec requires it.

---

## AnimatedContent

Content changes in place:

```kotlin
AnimatedContent(
    targetState = uiState.dialogState,
    transitionSpec = { fadeIn() togetherWith fadeOut() },
    label = "DialogStateTransition",
) { dialogState ->
    when (dialogState) {
        is DialogState.Loading -> LoadingContent()
        is DialogState.Success -> SuccessContent(dialogState)
        is DialogState.Error   -> ErrorContent(dialogState)
        is DialogState.None    -> Unit
    }
}
```

Always provide `label` — used by Android Studio's Animation Inspector.

---

## Crossfade

Simple content swap:

```kotlin
Crossfade(
    targetState = uiState.isLoading,
    label = "ContentCrossfade",
) { isLoading ->
    if (isLoading) LoadingIndicator() else ContentList(uiState.items)
}
```

---

## Value Animations

Animate a single property change:

```kotlin
val elevation by animateDpAsState(
    targetValue = if (isScrolled) 4.dp else 0.dp,
    label = "ToolbarElevation",
)

val alpha by animateFloatAsState(
    targetValue = if (uiState.isEnabled) 1f else 0.38f,
    animationSpec = tween(durationMillis = 200),
    label = "ButtonAlpha",
)
```

---

## List Item Animations

```kotlin
LazyColumn {
    items(items = uiState.groups, key = { it.id }) { group ->
        GroupItem(
            group = group,
            modifier = Modifier.animateItem(),
        )
    }
}
```

`animateItem()` handles placement animations automatically. No custom spec needed for standard reordering.

---

## Spring vs Tween

| Spec | When to use |
|------|------------|
| `spring()` | Physics-based: draggable elements, gesture-driven motion |
| `tween()` | Time-based: fades, reveals, transitions defined by design |
| Default | Compose spring defaults — use when no spec is specified |

---

## Duration Guidelines

| Transition | Duration |
|------------|---------|
| Fade in / out | 150–200ms |
| Expand / collapse | 200–300ms |
| Content switch | 200–250ms |
| Screen-level transition | 300–400ms |
| Loading state appearing | Immediate — no animation |

---

## Rules

Always:

- Provide `label` to every `animateXxxAsState`, `AnimatedContent`, `Crossfade`
- Keep animations under 400ms
- Use `key` with `animateItem()` in lazy lists

Never:

- Animate loading indicators appearing — show them instantly
- Nest `AnimatedVisibility` inside `AnimatedContent`
- Add animation to pure decoration with no state meaning
- Use `InfiniteTransition` for loading states — use `CircularProgressIndicator`
