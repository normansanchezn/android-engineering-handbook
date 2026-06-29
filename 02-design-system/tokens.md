# Design Tokens

## Purpose

Design tokens are the named values that define the visual language: colors, typography, spacing, shapes.

They ensure consistency across the app and make design changes propagate automatically.

---

## Color Tokens

Colors are accessed through `MaterialTheme.colorScheme`. Never use raw hex values in Composables.

| Token | Role | Example use |
|-------|------|-------------|
| `primary` | Brand color, main CTAs | Primary buttons, active tabs |
| `onPrimary` | Text/icons on primary | Button label on filled button |
| `primaryContainer` | Subtle primary background | Tag backgrounds, chips |
| `onPrimaryContainer` | Text on primaryContainer | Tag text |
| `secondary` | Accent color | Secondary actions |
| `surface` | Default card/sheet background | Cards, bottom sheets |
| `surfaceVariant` | Elevated surface background | Form inputs, section backgrounds |
| `onSurface` | Primary text | Body text, labels |
| `onSurfaceVariant` | Secondary text | Hints, captions |
| `outline` | Borders, dividers | Input field borders |
| `error` | Destructive/invalid | Error messages, delete actions |

Usage:

```kotlin
// ✅ Token reference
Box(modifier = Modifier.background(MaterialTheme.colorScheme.primaryContainer))
Text(text = label, color = MaterialTheme.colorScheme.onSurface)

// ❌ Hardcoded color
Box(modifier = Modifier.background(Color(0xFFCFF3E3)))
```

---

## Typography Tokens

Typography is accessed through `MaterialTheme.typography`. See [theming.md](theming.md) for the full scale.

| Token | Use case |
|-------|---------|
| `displaySmall` – `displayLarge` | Hero headings on main screens |
| `headlineSmall` – `headlineLarge` | Section titles, screen titles |
| `titleSmall` – `titleLarge` | Card titles, toolbar titles |
| `bodySmall` – `bodyLarge` | Body text, descriptions |
| `labelSmall` – `labelLarge` | Buttons, chips, tags, captions |

---

## Shape Tokens

Shapes are accessed through `MaterialTheme.shapes`. See [theming.md](theming.md) for values.

| Token | Approx radius | Use case |
|-------|--------------|---------|
| `extraSmall` | 8dp | Small chips, badges |
| `small` | 16dp | Text fields, small cards |
| `medium` | 24dp | Cards, bottom sheets |
| `large` | 36dp | Full-width cards, dialogs |
| `extraLarge` | 42dp | Pill buttons, floating elements |

---

## Spacing

Spacing is defined by the 8dp grid. All padding and spacing values must be multiples of 4dp (or 8dp for major gaps).

```kotlin
// Common spacing values
val SpacingXs = 4.dp
val SpacingS  = 8.dp
val SpacingM  = 16.dp
val SpacingL  = 24.dp
val SpacingXl = 32.dp
val SpacingXxl = 48.dp

// Usage
Column(
    modifier = Modifier.padding(horizontal = SpacingL, vertical = SpacingM),
    verticalArrangement = Arrangement.spacedBy(SpacingS)
) { ... }
```

---

## Rules

Always:

- Reference color, typography, and shape tokens via `MaterialTheme.xxx`
- Use 4dp/8dp grid for all spacing values
- Add new colors to the color scheme in `AppColors.kt` — not as raw values in component files

Never:

- Hardcode `Color(0xFF...)` in a composable
- Hardcode `TextStyle(fontSize = 16.sp)` in a composable
- Use `RoundedCornerShape(24.dp)` directly — use `MaterialTheme.shapes.medium`
- Use arbitrary spacing values like `15.dp` or `22.dp` — round to the nearest 4dp
