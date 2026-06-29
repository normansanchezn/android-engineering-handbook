# New Feature Checklist

## Files to Create

For a feature named `Groups`, create:

### presentation/feature/groups/
```
groups/
├── GroupRouter.kt                   # Route constant
├── viewModel/
│   ├── GroupContract.kt             # MVI Contract
│   └── GroupViewModel.kt            # ViewModel implementing Contract
├── ui/
│   ├── GroupScreen.kt               # Root composable + effect collector
│   └── GroupFormContent.kt          # Private sub-composable (optional)
└── model/
    └── GroupUiModel.kt              # UI-layer model (if domain model differs)
```

### domain/
```
usecases/groups/
├── SaveGroupUseCase.kt
└── GetGroupsUseCase.kt

repository/
└── GroupRepository.kt               # Interface
```

### data/
```
repository/
└── GroupRepositoryImpl.kt           # Implementation

remote/dto/
└── GroupDto.kt                      # DTO + toDomain() + toDto()

di/
└── GroupModule.kt                   # @Binds interface → implementation
```

---

## Step-by-Step

### 1. Domain model
```kotlin
// domain/models/GroupModel.kt
data class GroupModel(
    val id: String,
    val name: String,
    val label: String,
    val teacherId: String,
)
```

### 2. Repository interface
```kotlin
// domain/repository/GroupRepository.kt
interface GroupRepository {
    suspend fun getAll(): Either<Failure, List<GroupModel>>
    suspend fun save(group: GroupModel): Either<Failure, Unit>
}
```

### 3. DTO + mapping
```kotlin
// data/remote/dto/GroupDto.kt
data class GroupDto(val id: String, val name: String, val label: String, val teacherId: String)
fun GroupDto.toDomain() = GroupModel(id = id, name = name, label = label, teacherId = teacherId)
fun GroupModel.toDto() = GroupDto(id = id, name = name, label = label, teacherId = teacherId)
```

### 4. Repository implementation
```kotlin
// data/repository/GroupRepositoryImpl.kt
class GroupRepositoryImpl @Inject constructor(private val api: GroupApi) : GroupRepository {
    override suspend fun getAll(): Either<Failure, List<GroupModel>> =
        safeApiCall { api.getGroups().map { it.toDomain() } }
    override suspend fun save(group: GroupModel): Either<Failure, Unit> =
        safeApiCall { api.saveGroup(group.toDto()); Unit }
}
```

### 5. DI module
```kotlin
@Module @InstallIn(SingletonComponent::class)
abstract class GroupModule {
    @Binds @Singleton
    abstract fun bindGroupRepository(impl: GroupRepositoryImpl): GroupRepository
}
```

### 6. Use cases
```kotlin
class SaveGroupUseCase @Inject constructor(private val repo: GroupRepository) {
    suspend operator fun invoke(group: GroupModel) = repo.save(group)
}
class GetGroupsUseCase @Inject constructor(private val repo: GroupRepository) {
    suspend operator fun invoke() = repo.getAll()
}
```

### 7. Contract
→ See [contract.md](contract.md)

### 8. ViewModel
→ See [viewmodel.md](viewmodel.md)

### 9. Router
```kotlin
object GroupRouter {
    const val ROUTE = "Groups"
}
```

### 10. Screen
→ See [screen.md](screen.md)

### 11. Register in NavHost
```kotlin
composable(route = GroupRouter.ROUTE) {
    GroupScreen(onNavigateBack = { navController.popBackStack() })
}
```

### 12. String resources
Add all entries referenced in `createUiStrings()` to `res/values/strings.xml`.

---

## Completion Checklist

**Domain**
- [ ] Domain model — pure Kotlin, no framework annotations
- [ ] Repository interface — returns `Either<Failure, T>`
- [ ] Use cases — one per operation, `suspend operator fun invoke()`

**Data**
- [ ] DTO with `toDomain()` and `toDto()` extensions
- [ ] Repository implementation — uses `safeApiCall`
- [ ] DI module — `@Binds` interface → implementation

**Presentation**
- [ ] Contract — UiState, ScreenUiModel, Effect, Event, DialogState
- [ ] ViewModel — strings loaded in constructor, all Failure types handled
- [ ] Router — `const val ROUTE`
- [ ] Screen — effects in `LaunchedEffect(Unit)`, dialog as overlay

**App**
- [ ] Route registered in NavHost

**Resources**
- [ ] All strings added to `strings.xml`

**Quality**
- [ ] No hardcoded strings in Compose
- [ ] No DTOs in domain or presentation
- [ ] No business logic in ViewModel or Screen
