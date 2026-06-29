# DataSource

## When to Use

A DataSource is an optional layer between the Repository and the network/database.

Use a DataSource when:
- The repository would otherwise have two direct dependencies (API + DAO) and caching logic that belongs extracted
- The same remote endpoint is used by multiple repositories
- You want to swap implementations (e.g., mock remote vs. real remote) without touching the repository

For simple CRUD features with one data source, skip the DataSource and call the API service or DAO directly from the repository.

---

## Decision Table

| Scenario | Pattern |
|----------|---------|
| One repository, one API service | Repository calls `apiService` directly |
| Repository needs both remote and local | Add `RemoteDataSource` and `LocalDataSource` |
| Two repositories share the same API endpoint | Extract a `RemoteDataSource` |
| Offline-first with cache refresh | Repository orchestrates `RemoteDataSource` + `LocalDataSource` |

---

## Remote DataSource

```kotlin
// data/datasource/GroupRemoteDataSource.kt
interface GroupRemoteDataSource {
    suspend fun getGroups(): List<GroupDto>
    suspend fun createGroup(request: CreateGroupRequest): GroupDto
    suspend fun deleteGroup(id: String)
}

class GroupRemoteDataSourceImpl @Inject constructor(
    private val apiService: GroupApiService,
) : GroupRemoteDataSource {

    override suspend fun getGroups(): List<GroupDto> =
        apiService.getGroups()

    override suspend fun createGroup(request: CreateGroupRequest): GroupDto =
        apiService.createGroup(request)

    override suspend fun deleteGroup(id: String) =
        apiService.deleteGroup(id)
}
```

---

## Local DataSource

```kotlin
// data/datasource/GroupLocalDataSource.kt
interface GroupLocalDataSource {
    fun observeGroups(): Flow<List<GroupEntity>>
    suspend fun upsertGroups(groups: List<GroupEntity>)
    suspend fun deleteGroup(id: String)
}

class GroupLocalDataSourceImpl @Inject constructor(
    private val dao: GroupDao,
) : GroupLocalDataSource {

    override fun observeGroups(): Flow<List<GroupEntity>> = dao.observeGroups()

    override suspend fun upsertGroups(groups: List<GroupEntity>) = dao.upsertGroups(groups)

    override suspend fun deleteGroup(id: String) = dao.deleteGroupById(id)
}
```

---

## Repository Using DataSources

```kotlin
class GroupRepositoryImpl @Inject constructor(
    private val remoteDataSource: GroupRemoteDataSource,
    private val localDataSource: GroupLocalDataSource,
) : GroupRepository {

    override fun observeGroups(): Flow<Either<Failure, List<GroupModel>>> =
        localDataSource.observeGroups().map { entities ->
            Either.Success(entities.map { it.toDomain() })
        }

    override suspend fun refreshGroups(): Either<Failure, Unit> =
        safeApiCall {
            val dtos = remoteDataSource.getGroups()
            localDataSource.upsertGroups(dtos.map { it.toEntity() })
        }
}
```

---

## DI Binding

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class DataSourceModule {

    @Binds
    abstract fun bindGroupRemoteDataSource(impl: GroupRemoteDataSourceImpl): GroupRemoteDataSource

    @Binds
    abstract fun bindGroupLocalDataSource(impl: GroupLocalDataSourceImpl): GroupLocalDataSource
}
```

---

## Rules

Always:

- Define DataSource as an interface — implementations are injected
- Keep DataSources thin: no business logic, no `Either` wrapping
- `safeApiCall` wrapping happens in the Repository, not the DataSource

Never:

- Add a DataSource just to add a layer — only extract when there's a clear reason
- Let the DataSource call another DataSource
- Return domain models from a DataSource — it deals in DTOs and entities
