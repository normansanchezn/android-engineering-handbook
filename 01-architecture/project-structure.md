# Project Structure

## Module Layout

```
root/
├── app/                            # Composition root, NavHost, Application class
├── presentation/                   # UI layer — screens, ViewModels, MVI Contracts
│   └── src/main/
│       └── com/myapp/presentation/
│           ├── feature/
│           │   ├── groups/
│           │   │   ├── GroupRouter.kt
│           │   │   ├── viewModel/
│           │   │   │   ├── GroupContract.kt
│           │   │   │   └── GroupViewModel.kt
│           │   │   ├── ui/
│           │   │   │   └── GroupScreen.kt
│           │   │   └── model/
│           │   │       └── GroupUiModel.kt
│           │   ├── signin/
│           │   ├── onboarding/
│           │   └── ...
│           ├── main/               # MainActivity
│           ├── di/                 # Presentation DI modules
│           └── utils/              # Presentation utilities
│
├── domain/                         # Business layer — use cases, models, repository interfaces
│   └── src/main/
│       └── com/myapp/domain/
│           ├── models/
│           │   ├── GroupModel.kt
│           │   └── StudentModel.kt
│           ├── usecases/
│           │   ├── group/
│           │   │   ├── SaveGroupUseCase.kt
│           │   │   └── GetGroupsUseCase.kt
│           │   └── student/
│           └── repository/
│               ├── GroupRepository.kt
│               ├── StudentRepository.kt
│               └── UserPreferencesRepository.kt
│
├── data/                           # Infrastructure layer — implementations, DTOs, network
│   └── src/main/
│       └── com/myapp/data/
│           ├── repository/
│           │   ├── GroupRepositoryImpl.kt
│           │   └── UserPreferencesRepositoryImpl.kt
│           ├── remote/
│           │   ├── api/            # Retrofit interfaces or SDK clients
│           │   └── dto/            # DTOs + toDomain() / toDto() extensions
│           ├── local/
│           │   ├── dao/            # Room DAOs
│           │   └── entity/         # Room entities
│           ├── di/                 # Hilt modules (@Binds)
│           └── mapper/             # Mapper classes (for complex mappings)
│
├── design-system/                  # Reusable UI foundations
│   └── src/main/
│       └── com/myapp/designsystem/
│           ├── mvi/                # MVIContract, MVIContractEvent
│           ├── theme/              # AppTheme, colors, typography, shapes
│           └── ui/
│               ├── primitives/     # AppButton, AppTextField, AppText
│               └── components/     # AppStatusDialog, AppCard, AppToolbar
│
├── core/                           # Generic utilities
│   └── src/main/
│       └── com/myapp/core/
│           ├── error/              # Failure sealed class
│           ├── functional/         # Either, fold, Ext
│           ├── remote/handler/     # safeApiCall, Throwable.toEither()
│           ├── mapper/             # Mapper<Domain, Entity> interface
│           └── validation/         # validateRequiredTextField
│
├── gradle/
│   └── libs.versions.toml          # Version catalog
├── build-logic/                    # Convention plugins (optional)
├── build.gradle.kts
└── settings.gradle.kts
```

---

## Feature Package Structure

```
feature/groups/
├── GroupRouter.kt                  # Route constant
├── viewModel/
│   ├── GroupContract.kt            # MVI Contract (State/Effect/Event)
│   └── GroupViewModel.kt           # ViewModel implementing Contract
├── ui/
│   ├── GroupScreen.kt              # Root composable, effect collector
│   └── GroupFormContent.kt         # Stateless content composable (optional)
└── model/
    └── GroupUiModel.kt             # UI-layer model (when domain model differs)
```

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Feature package | `snake_case` | `feature/student_report/` |
| Kotlin classes | `PascalCase` | `GroupViewModel` |
| Contract | `XxxContract` | `GroupContract` |
| ViewModel | `XxxViewModel` | `GroupViewModel` |
| Screen | `XxxScreen` | `GroupScreen` |
| Router | `XxxRouter` | `GroupRouter` |
| UseCase | `VerbXxxUseCase` | `SaveGroupUseCase` |
| Repository interface | `XxxRepository` | `GroupRepository` |
| Repository impl | `XxxRepositoryImpl` | `GroupRepositoryImpl` |
| DTO | `XxxDto` | `GroupDto` |
| Domain model | `XxxModel` | `GroupModel` |
| UiModel | `XxxUiModel` | `GroupUiModel` |
| DI module | `XxxModule` | `GroupModule` |
