# DataSource Template

## Purpose

A DataSource is an optional abstraction layer beneath the repository.

Use it when the repository would become too large, or when the same network resource or table is accessed by multiple repositories.

In most features, the repository talks directly to the network client or DAO — no DataSource is needed.

---

## Remote DataSource

```kotlin
// data/remote/datasource/GroupRemoteDataSource.kt
interface GroupRemoteDataSource {
    suspend fun fetchAll(): List<GroupDto>
    suspend fun insert(dto: GroupDto)
    suspend fun delete(id: String)
}

class GroupRemoteDataSourceImpl @Inject constructor(
    private val api: GroupApi,
) : GroupRemoteDataSource {
    override suspend fun fetchAll(): List<GroupDto> = api.getGroups()
    override suspend fun insert(dto: GroupDto) { api.saveGroup(dto) }
    override suspend fun delete(id: String) { api.deleteGroup(id) }
}
```

Repository wraps calls in `safeApiCall`:

```kotlin
class GroupRepositoryImpl @Inject constructor(
    private val dataSource: GroupRemoteDataSource,
) : GroupRepository {
    override suspend fun getAll(): Either<Failure, List<GroupModel>> =
        safeApiCall { dataSource.fetchAll().map { it.toDomain() } }
}
```

---

## Local DataSource (Room DAO)

```kotlin
// data/local/dao/GroupDao.kt
@Dao
interface GroupDao {
    @Query("SELECT * FROM groups")
    fun observeAll(): Flow<List<GroupEntity>>

    @Upsert
    suspend fun upsert(entity: GroupEntity)

    @Delete
    suspend fun delete(entity: GroupEntity)
}
```

---

## When to Add a DataSource

| Add DataSource | Skip DataSource |
|---------------|----------------|
| Repository has 5+ methods | Repository has 1-3 simple calls |
| Same endpoint used by 2+ repositories | Single repository owns the resource |
| Complex query logic | Simple CRUD |

---

## Rules

- DataSource interfaces live in `data/` — unlike repository interfaces which live in `domain/`
- DataSources return DTOs or entities — never domain models
- `safeApiCall` wraps the DataSource call in the Repository, not inside the DataSource
- DataSources are not mandatory — add them when complexity justifies it
