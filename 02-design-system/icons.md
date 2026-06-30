# Icons

## Strategy

Icons use `ImageVector` for Material symbols and `painterResource` for custom vector drawables.

Never reference drawable resources directly in feature modules. Route all icons through the design system's `AppIcons` object.

---

## Material Icons

```kotlin
Icon(
    imageVector = Icons.Outlined.Add,
    contentDescription = stringResource(R.string.add_group),
)

Icon(
    imageVector = Icons.Filled.ArrowBack,
    contentDescription = stringResource(R.string.navigate_back),
)
```

Prefer `Icons.Outlined` over `Icons.Filled` unless the design spec requires filled.

---

## Custom Icons

Custom icons are declared in the design system module:

```
design-system/src/main/kotlin/designsystem/icon/AppIcons.kt
design-system/src/main/res/drawable/
├── ic_calendar.xml
├── ic_student.xml
└── ic_group.xml
```

```kotlin
// design-system/icon/AppIcons.kt
object AppIcons {
    val Calendar = R.drawable.ic_calendar
    val Student  = R.drawable.ic_student
    val Group    = R.drawable.ic_group
    val Logo     = R.drawable.ic_logo
}
```

Usage in any module:

```kotlin
Icon(
    painter = painterResource(AppIcons.Calendar),
    contentDescription = stringResource(R.string.calendar),
    tint = MaterialTheme.colorScheme.onSurface,
)
```

---

## ImageVector vs Painter

| Use case | Type |
|----------|------|
| Material Design icons | `ImageVector` via `Icons.Xxx.Name` |
| Custom single-color icons (tintable) | `painterResource(AppIcons.Xxx)` with `tint` |
| Logos, multi-color illustrations | `painterResource(AppIcons.Logo)` without `tint` |

---

## Vector Drawable Requirements

All custom icons must be vector drawables (`<vector>` XML). No PNG icons.

For tintable icons, use a single-color path:

```xml
<!-- res/drawable/ic_group.xml -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="@color/black"
        android:pathData="M12,12c2.21,0 4,-1.79 4,-4s-1.79,-4 -4,-4..." />
</vector>
```

Single black path — the `tint` modifier overrides color at runtime.

---

## Naming Convention

```
ic_[noun].xml
ic_[noun]_[variant].xml

ic_arrow_back.xml
ic_calendar.xml
ic_student_outline.xml
ic_logo.xml
```

---

## Content Descriptions

Every `Icon` must have a `contentDescription`.

```kotlin
// ✅ Describes the action
Icon(imageVector = Icons.Outlined.Delete, contentDescription = "Delete group")

// ❌ Describes the shape
Icon(imageVector = Icons.Outlined.Delete, contentDescription = "Trash icon")

// ✅ Decorative icon next to visible label — null is correct
Row {
    Icon(imageVector = Icons.Outlined.Add, contentDescription = null)
    Text("Add group")
}
```

---

## Rules

Always:

- Declare custom icons in `AppIcons` inside the design system module
- Use `24dp` for standard action icons
- Supply `contentDescription` on every interactive icon
- Use single-color vector drawables for tintable icons

Never:

- Reference `R.drawable.ic_xxx` directly in feature modules
- Use PNG icons
- Hardcode icon colors — use `tint = MaterialTheme.colorScheme.xxx`
- Duplicate a Material icon as a custom drawable
