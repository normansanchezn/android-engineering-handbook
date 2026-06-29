# Theming

## AppTheme

`AppTheme` is the single `MaterialTheme` wrapper used throughout the app.

It applies the color scheme, typography, and shapes. It supports light/dark mode automatically.

```kotlin
// design-system/theme/AppTheme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme

    // Sync status bar with theme
    val view = LocalView.current
    val inspectionMode = LocalInspectionMode.current
    if (!inspectionMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = colorScheme.background.toArgb()
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = !darkTheme
        }
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content,
    )
}
```

Applied at the root (in `MainActivity` or `NavHost`):

```kotlin
AppTheme {
    AppNavHost(navController = rememberNavController())
}
```

---

## Color Scheme

Two schemes: light and dark. Both defined in `AppColors.kt`.

```kotlin
// design-system/theme/AppColors.kt

// Brand palette
val GreenPrimary = Color(0xFF49B583)
val OnGreenPrimary = Color(0xFF052E24)
val GreenContainer = Color(0xFFCFF3E3)

val PeriwinkleSecondary = Color(0xFF7F86FF)
val AmberTertiary = Color(0xFFFFBD69)
val ErrorRed = Color(0xFFFF4171)

val LightColorScheme = lightColorScheme(
    primary = GreenPrimary,
    onPrimary = OnGreenPrimary,
    primaryContainer = GreenContainer,
    secondary = PeriwinkleSecondary,
    // ... full Material3 scheme
)

val DarkColorScheme = darkColorScheme(
    primary = Color(0xFF6DD6A8),
    // ... full dark scheme
)
```

---

## Typography Scale

Defined in `AppTypography.kt`. Based on Material 3 type scale.

```kotlin
val AppTypography = Typography(
    displayLarge  = TextStyle(fontFamily = AppFontFamily, fontWeight = Black, fontSize = 57.sp),
    displayMedium = TextStyle(fontFamily = AppFontFamily, fontWeight = Black, fontSize = 45.sp),
    displaySmall  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 36.sp),

    headlineLarge  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 32.sp),
    headlineMedium = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 28.sp),
    headlineSmall  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 24.sp),

    titleLarge   = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 22.sp),
    titleMedium  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 16.sp),
    titleSmall   = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 14.sp),

    bodyLarge  = TextStyle(fontFamily = AppFontFamily, fontWeight = Normal, fontSize = 16.sp),
    bodyMedium = TextStyle(fontFamily = AppFontFamily, fontWeight = Normal, fontSize = 14.sp),
    bodySmall  = TextStyle(fontFamily = AppFontFamily, fontWeight = Normal, fontSize = 12.sp),

    labelLarge  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 14.sp),
    labelMedium = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 12.sp),
    labelSmall  = TextStyle(fontFamily = AppFontFamily, fontWeight = Bold, fontSize = 11.sp),
)
```

Usage in components:

```kotlin
Text(text = title, style = MaterialTheme.typography.headlineMedium)
Text(text = body, style = MaterialTheme.typography.bodyMedium)
Text(text = buttonLabel, style = MaterialTheme.typography.labelLarge)
```

---

## Shape Scale

```kotlin
// design-system/theme/AppShape.kt
val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(8.dp),
    small      = RoundedCornerShape(16.dp),
    medium     = RoundedCornerShape(24.dp),
    large      = RoundedCornerShape(36.dp),
    extraLarge = RoundedCornerShape(42.dp),
)
```

Usage:

```kotlin
Surface(shape = MaterialTheme.shapes.medium) { ... }
Card(shape = MaterialTheme.shapes.large) { ... }
```

---

## Rules

Always:

- Apply `AppTheme` once at the root — not inside individual screens
- Wrap Compose Previews with `AppTheme { }` — ensures correct visual output
- Access colors via `MaterialTheme.colorScheme.xxx` — never hardcode `Color(0xFF...)`
- Access typography via `MaterialTheme.typography.xxx` — never hardcode `TextStyle(...)`
- Access shapes via `MaterialTheme.shapes.xxx` — never hardcode `RoundedCornerShape(...)`

Never:

- Call `MaterialTheme { }` directly in a feature screen
- Define a secondary theme for a specific feature
- Use hardcoded colors, text sizes, or shapes outside `AppColors.kt`, `AppTypography.kt`, `AppShape.kt`
