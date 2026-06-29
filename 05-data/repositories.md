# Repositories

## Purpose

Repositories are the boundary between the domain and the infrastructure.

The domain defines the contract (interface). The data module implements it.

---

## Two-Part Structure

| Part | Module | Location |
|------|--------|----------|
| Interface | `domain` | `domain/repository/` |
| Implementation | `data` | `data/repository/` |

---

## Interface (domain)

```kotlin
// domain/repository/GroupRepository.kt
interface GroupRepository {
    suspend fun getAll(): Either<Failure, List<GroupModel>>
    suspend fun save(group: GroupModel): Either<Failure, Unit>
    suspend fun delete(id: String): Either<Failure, Unit>
}
```

Rules for the interface:

- Only domain models in the signature — no DTOs, no entities, no HTTP types
- Returns `Either<Failure, T>` — no exceptions, no nullable returns
- No Android imports — pure Kotlin

---

## Implementation (data)

```kotlin
// data/repository/GroupRepositoryImpl.kt
class GroupRepositoryImpl @Inject constructor(
    private val api: GroupApi,
) : GroupRepository {

    override suspend fun getAll(): Either<Failure, List<GroupModel>> =
        safeApiCall { api.getGroups().map { it.toDomain() } }

    override suspend fun save(group: GroupModel): Either<Failure, Unit> =
        safeApiCall { api.saveGroup(group.toDto()); Unit }

    override suspend fun delete(id: String): Either<Failure, Unit> =
        safeApiCall { api.deleteGroup(id); Unit }
}
```

---

## Local Repository (DataStore or Room)

```kotlin
// domain/repository/UserPreferencesRepository.kt
interface UserPreferencesRepository {
    suspend fun getTheme(): Either<Failure, AppTheme>
    suspend fun saveTheme(theme: AppTheme): Either<Failure, Unit>
}

// data/repository/UserPreferencesRepositoryImpl.kt
class UserPreferencesRepositoryImpl @Inject constructor(
    private val dataStore: DataStore<Preferences>,
) : UserPreferencesRepository {

    override suspend fun getTheme(): Either<Failure, AppTheme> =
        safeApiCall {
            val prefs = dataStore.data.first()
            AppTheme.fromString(prefs[PreferenceKeys.THEME] ?: AppTheme.SYSTEM.name)
        }

    override suspend fun saveTheme(theme: AppTheme): Either<Failure, Unit> =
        safeApiCall {
            dataStore.edit { it[PreferenceKeys.THEME] = theme.name }
            Unit
        }
}
```

---

## DI Binding

```kotlin
// data/di/GroupModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class GroupModule {

    @Binds
    @Singleton
    abstract fun bindGroupRepository(impl: GroupRepositoryImpl): GroupRepository
}
```

---

## Rules

Always:

- Interface in `domain/repository/` — use cases depend on this
- Implementation in `data/repository/` — this depends on the interface
- Wrap every call with `safeApiCall { }` — never use try/catch directly
- Map DTO/entity → domain model before returning
- Map domain model → DTO before inserting
- Bind with `@Binds` — not `@Provides`

Never:

- Expose DTOs outside the data module
- Put business rules inside a repository — those belong in use cases
- Make the domain interface aware of network clients, DAOs, or SDKs
- Return nullable types from a repository — use `Either.Error(Failure.EmptyData(...))` instead
