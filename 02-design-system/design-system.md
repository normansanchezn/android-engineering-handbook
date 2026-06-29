# Design System

## Purpose

The design-system module is the single source of visual truth for the application.

It provides:
- **MVIContract** — the shared interface every ViewModel implements
- **Theme** — colors, typography, shapes
- **Primitives** — foundational components (`AppButton`, `AppTextField`, `AppText`)
- **Components** — composed UI patterns (`AppStatusDialog`, `AppCard`, `AppToolbar`)

All other modules consume these foundations. None of them re-implement them.

---

## Module Structure

```
design-system/
└── src/main/
    └── com/myapp/designsystem/
        ├── mvi/
        │   ├── MVIContract.kt          # Core MVI interface
        │   └── MVIContractEvent.kt
        ├── theme/
        │   ├── AppTheme.kt             # MaterialTheme wrapper
        │   ├── AppColors.kt            # Color palette + light/dark schemes
        │   ├── AppTypography.kt        # Typography scale
        │   └── AppShape.kt             # Shape tokens
        └── ui/
            ├── primitives/             # Foundational components
            │   ├── AppButton.kt
            │   ├── AppTextField.kt
            │   └── AppText.kt
            └── components/             # Composed patterns
                ├── AppStatusDialog.kt
                ├── AppCard.kt
                └── AppToolbar.kt
```

---

## What Belongs in design-system

| Belongs | Reason |
|---------|--------|
| `MVIContract` | Shared by every ViewModel — needs to live in a module that `presentation` can depend on |
| `AppTheme` | Applied at the root — must be accessible to all screens |
| `AppButton`, `AppTextField`, `AppText` | Reused across many features |
| `AppStatusDialog` | Standard dialog used in all features |
| Color palette, typography scale, shape tokens | Visual consistency across the app |

---

## What Does NOT Belong in design-system

| Does not belong | Belongs in |
|-----------------|-----------|
| Feature-specific screens | `presentation/feature/xxx/ui/` |
| Business logic | `domain/usecases/` |
| API calls | `data/` |
| Domain models | `domain/models/` |
| ViewModels | `presentation/feature/xxx/viewModel/` |

---

## Rules

Always:

- Components in `design-system` accept only primitives, enums, and lambdas
- Every component has a `@Preview` covering its key states
- `AppTheme` is the single `MaterialTheme` wrapper — never call `MaterialTheme { }` directly in features

Never:

- Import domain models, repository interfaces, or ViewModels in `design-system`
- Duplicate components — if something exists in `design-system`, use it
- Hardcode colors — always use `MaterialTheme.colorScheme.xxx`
- Hardcode text styles — always use `MaterialTheme.typography.xxx`
