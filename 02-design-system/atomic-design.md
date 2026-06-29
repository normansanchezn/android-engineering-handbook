# Atomic Design

## Hierarchy

The design-system follows a simplified atomic model: three levels, not five.

```
Tokens
  └── Primitives
        └── Components
```

| Level | Examples | Location |
|-------|---------|---------|
| **Tokens** | colors, typography, shapes, spacing | `theme/AppColors.kt`, `theme/AppTypography.kt`, `theme/AppShape.kt` |
| **Primitives** | `AppButton`, `AppTextField`, `AppText` | `ui/primitives/` |
| **Components** | `AppStatusDialog`, `AppCard`, `AppToolbar` | `ui/components/` |

---

## Level 1 — Tokens

Raw design values. Not composable. Defined once, referenced everywhere.

Rules:
- Never used directly in screens — always via `MaterialTheme.xxx`
- Define new colors in `AppColors.kt`, not in component files

---

## Level 2 — Primitives

Single-purpose composables that wrap one Material3 component with app-specific defaults.

Characteristics:
- Take only primitive params (`String`, `Boolean`, lambdas)
- No internal state (stateless)
- No direct token reference inline — always via `MaterialTheme`

```kotlin
// Good: one responsibility, no domain dependency
@Composable
fun AppButton(text: String, onClick: () -> Unit, enabled: Boolean = true) { ... }

// Bad: two responsibilities merged
@Composable
fun AppSaveButton(groupName: String, onSave: (String) -> Unit) { ... }
```

---

## Level 3 — Components

Composed patterns built from two or more primitives (or one primitive + layout).

Characteristics:
- Solve a recurring UI problem that appears in multiple features
- Still accept only primitive params — no domain models
- May have internal layout logic (e.g., Row/Column arrangement)

```kotlin
// Good: combines layout + primitive + optional content slot
@Composable
fun AppCard(onClick: () -> Unit, content: @Composable ColumnScope.() -> Unit) { ... }

// Bad: domain object leaking into design-system
@Composable
fun GroupCard(group: GroupModel, onClick: () -> Unit) { ... }
```

---

## What Does NOT Fit the Hierarchy

Feature-specific composables belong in `presentation`, not `design-system`.

| Use case | Where it goes |
|----------|-------------|
| A button used in every feature | `design-system/ui/primitives/AppButton.kt` |
| A card that shows a Group | `presentation/feature/groups/ui/GroupCard.kt` |
| A form used only in the login screen | `presentation/feature/login/ui/LoginForm.kt` |
| The dialog shared across all features | `design-system/ui/components/AppStatusDialog.kt` |

---

## Growth Rule

When a UI element appears in **three or more features** with the same visual form and different data, it is a candidate for promotion to design-system.

Process:
1. Extract the data into primitive params (`String`, enum, lambda)
2. Remove domain-model dependency from the composable
3. Move to `design-system/ui/components/`
4. Add `@Preview` for all key states
