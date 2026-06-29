# Example: Repository Implementation

Complete data layer for a `Group` resource: API service, DTO, mapper, repository interface, implementation, DI binding, and fake.

---

## File Structure

```
domain/
├── models/
│   └── GroupModel.kt
└── repository/
    └── GroupRepository.kt

data/
├── dto/
│   └── GroupDto.kt
├── mapper/
│   └── GroupMapper.kt
├── remote/
│   └── GroupApiService.kt
└── repository/
    └── GroupRepositoryImpl.kt

di/
└── GroupModule.kt

test/fake/
└── FakeGroupRepository.kt
```

---

## Domain Model

```kotlin
// domain/models/GroupModel.kt
data class GroupModel(
    val id: String,
    val name: String,
    val label: String,
    val createdAt: LocalDateTime,
)
```

---

## Repository Interface

```kotlin
// domain/repository/GroupRepository.kt
interface GroupRepository {
    suspend fun getGroups(): Either<Failure, List<GroupModel>>
    suspend fun getGroupById(id: String): Either<Failure, GroupModel>
    suspend fun saveGroup(group: GroupModel): Either<Failure, Unit>
    suspend fun deleteGroup(id: String): Either<Failure, Unit>
}
```

---

## DTO

```kotlin
// data/dto/GroupDto.kt
@Serializable
data class GroupDto(
    @SerialName("id") val id: String,
    @SerialName("name") val name: String,
    @SerialName("label") val label: String,
    @SerialName("created_at") val createdAt: String,
)

@Serializable
data class CreateGroupRequest(
    @SerialName("name") val name: String,
    @SerialName("label") val label: String,
)
```

---

## Mapper

```kotlin
// data/mapper/GroupMapper.kt
fun GroupDto.toDomain(): GroupModel = GroupModel(
    id = id,
    name = name,
    label = label,
    createdAt = LocalDateTime.parse(createdAt),
)

fun GroupModel.toDto(): CreateGroupRequest = CreateGroupRequest(
    name = name,
    label = label,
)
```

---

## API Service

```kotlin
// data/remote/GroupApiService.kt
interface GroupApiService {
    @GET("groups")
    suspend fun getGroups(): List<GroupDto>

    @GET("groups/{id}")
    suspend fun getGroupById(@Path("id") id: String): GroupDto

    @POST("groups")
    suspend fun createGroup(@Body request: CreateGroupRequest): GroupDto

    @DELETE("groups/{id}")
    suspend fun deleteGroup(@Path("id") id: String)
}
```

---

## Repository Implementation

```kotlin
// data/repository/GroupRepositoryImpl.kt
class GroupRepositoryImpl @Inject constructor(
    private val apiService: GroupApiService,
) : GroupRepository {

    override suspend fun getGroups(): Either<Failure, List<GroupModel>> =
        safeApiCall { apiService.getGroups().map { it.toDomain() } }

    override suspend fun getGroupById(id: String): Either<Failure, GroupModel> =
        safeApiCall { apiService.getGroupById(id).toDomain() }

    override suspend fun saveGroup(group: GroupModel): Either<Failure, Unit> =
        safeApiCall { apiService.createGroup(group.toDto()) }

    override suspend fun deleteGroup(id: String): Either<Failure, Unit> =
        safeApiCall { apiService.deleteGroup(id) }
}
```

---

## DI Binding

```kotlin
// di/GroupModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class GroupModule {

    @Binds
    @Singleton
    abstract fun bindGroupRepository(impl: GroupRepositoryImpl): GroupRepository
}
```

---

## Fake Repository (for tests)

```kotlin
// test/fake/FakeGroupRepository.kt
class FakeGroupRepository : GroupRepository {

    var shouldFail = false
    val savedGroups = mutableListOf<GroupModel>()
    val deletedIds = mutableListOf<String>()

    private val groups = mutableListOf(
        GroupModel(id = "1", name = "Algebra I", label = "A1", createdAt = LocalDateTime.now()),
        GroupModel(id = "2", name = "Geometry", label = "G1", createdAt = LocalDateTime.now()),
    )

    override suspend fun getGroups(): Either<Failure, List<GroupModel>> {
        if (shouldFail) return Either.Error(Failure.NetworkError)
        return Either.Success(groups.toList())
    }

    override suspend fun getGroupById(id: String): Either<Failure, GroupModel> {
        if (shouldFail) return Either.Error(Failure.NetworkError)
        return groups.find { it.id == id }
            ?.let { Either.Success(it) }
            ?: Either.Error(Failure.EmptyData)
    }

    override suspend fun saveGroup(group: GroupModel): Either<Failure, Unit> {
        if (shouldFail) return Either.Error(Failure.NetworkError)
        savedGroups += group
        groups += group
        return Either.Success(Unit)
    }

    override suspend fun deleteGroup(id: String): Either<Failure, Unit> {
        if (shouldFail) return Either.Error(Failure.NetworkError)
        deletedIds += id
        groups.removeAll { it.id == id }
        return Either.Success(Unit)
    }
}
```

---

## Testing the Repository

```kotlin
class GroupRepositoryImplTest {

    private val apiService = mockk<GroupApiService>()
    private val repository = GroupRepositoryImpl(apiService)

    @Test
    fun `getGroups maps DTOs to domain models`() = runTest {
        val dto = GroupDto(id = "1", name = "Algebra I", label = "A1", createdAt = "2024-01-01T10:00:00")
        coEvery { apiService.getGroups() } returns listOf(dto)

        val result = repository.getGroups()

        assertIs<Either.Success<List<GroupModel>>>(result)
        assertEquals("Algebra I", result.value.first().name)
    }

    @Test
    fun `getGroups returns NetworkError on IOException`() = runTest {
        coEvery { apiService.getGroups() } throws IOException("timeout")

        val result = repository.getGroups()

        assertIs<Either.Error<Failure.NetworkError>>(result)
    }
}
```
