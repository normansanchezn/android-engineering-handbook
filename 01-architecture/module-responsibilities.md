# Module Responsibilities

## Summary

| Module | Owns | Can use | Cannot use |
|--------|------|---------|-----------|
| `app` | NavHost, DI wiring, Application class | All modules | Business logic, reusable UI |
| `presentation` | Screens, ViewModels, Contracts | `domain`, `design-system`, `core` | `data`, DTOs, network, DB |
| `domain` | Use cases, domain models, repository interfaces | `core` | Android, network, DB, Compose |
| `data` | Repository implementations, DTOs, mappers, network/db access | `domain`, `core` | `presentation`, `design-system` |
| `design-system` | MVIContract, components, theme | `core` (only when needed) | `presentation`, `domain`, `data` |
| `core` | Either, Failure, safeApiCall, extensions | Nothing project-specific | Feature code, Android UI, domain models |

---

## app

**Owns:** `MainActivity`, `NavHost`, application-level DI setup, startup logic.

**Rule:** The only module that knows about all other modules. It wires the graph but contains no feature logic.

```
app/
├── MainActivity.kt
├── navigation/
│   └── AppNavHost.kt        ← registers routes from XxxRouter.ROUTE
└── MyApplication.kt         ← @HiltAndroidApp
```

---

## presentation

**Owns:** Screens, ViewModels, MVI Contracts, navigation routes, UI models.

**Depends on:** `domain` (use cases), `design-system` (components), `core` (utilities).

**Rule:** ViewModels call use cases. Screens render UiState. Effects trigger navigation. Nothing else.

---

## domain

**Owns:** Business logic (use cases), domain models, repository interfaces.

**Depends on:** `core` only — for `Either`, `Failure`, and generic abstractions.

**Rule:** This module is pure Kotlin. It has no Android dependencies. It is the center of the dependency graph.

---

## data

**Owns:** Repository implementations, DTOs, network clients (Retrofit, SDKs), Room entities and DAOs, DataStore, mappers.

**Depends on:** `domain` (implements its interfaces), `core` (`safeApiCall`, `Either`).

**Rule:** Maps DTOs to domain models before returning. Never exposes DTOs or entities outside this module.

---

## design-system

**Owns:** `MVIContract` interface, reusable Compose components, `AppTheme`, typography, color tokens.

**Depends on:** `core` only when needed for utilities.

**Rule:** Components are fully generic — accept strings, lambdas, and primitives. No ViewModel references. No domain models.

---

## core

**Owns:** `Either`, `Failure`, `safeApiCall`, `Mapper<D, E>`, extension functions, `validateRequiredTextField`.

**Depends on:** Nothing project-specific.

**Rule:** Framework-light. Must compile without any domain, data, or presentation imports. It is the foundation everything else builds on.

---

## Dependency Direction

```text
app ──► presentation ──► domain ◄── data
         │                 ▲           │
         ▼                 │           │
    design-system       core ◄─────────┘
         │                 ▲
         └─────────────────┘
```

All arrows point toward stable abstractions. No cycles. No upward dependencies.
