# Repository Template

## Interface (domain module)

```kotlin
// domain/repository/GroupRepository.kt
package com.myapp.domain.repository

import com.myapp.core.error.Failure
import com.myapp.core.functional.Either
import com.myapp.domain.models.GroupModel

interface GroupRepository {
    suspend fun getAll(): Either<Failure, List<GroupModel>>
    suspend fun save(group: GroupModel): Either<Failure, Unit>
    suspend fun delete(id: String): Either<Failure, Unit>
}
```

---

## Implementation (data module) — Generic network client

```kotlin
// data/repository/GroupRepositoryImpl.kt
package com.myapp.data.repository

import com.myapp.core.error.Failure
import com.myapp.core.functional.Either
import com.myapp.core.remote.handler.safeApiCall
import com.myapp.data.remote.api.GroupApi
import com.myapp.data.remote.dto.GroupDto
import com.myapp.data.remote.dto.toDomain
import com.myapp.data.remote.dto.toDto
import com.myapp.domain.models.GroupModel
import com.myapp.domain.repository.GroupRepository
import javax.inject.Inject

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

## Implementation — Retrofit (Response<T> wrapper)

```kotlin
class GroupRepositoryImpl @Inject constructor(
    private val api: GroupApi,
) : GroupRepository {

    override suspend fun getAll(): Either<Failure, List<GroupModel>> =
        safeApiCall(
            apiCall = { api.getGroups() },
            mapper = object : ResultMapper<GroupListResponse, List<GroupModel>> {
                override fun map(input: GroupListResponse) = input.items.map { it.toDomain() }
            }
        )
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

## DTO + Mapping Extensions

```kotlin
// data/remote/dto/GroupDto.kt
@Serializable
data class GroupDto(
    val id: String,
    val name: String,
    val label: String,
    val teacherId: String,
)

fun GroupDto.toDomain() = GroupModel(
    id = id,
    name = name,
    label = label,
    teacherId = teacherId,
)

fun GroupModel.toDto() = GroupDto(
    id = id,
    name = name,
    label = label,
    teacherId = teacherId,
)
```

---

## Rules

Always:

- Interface lives in `domain/repository/` — returns domain models
- Implementation lives in `data/repository/` — knows about DTOs and network clients
- Wrap every network call with `safeApiCall { }` — never try/catch directly
- Return `Either<Failure, T>` — never throw from a repository
- Map DTO → domain model inside the repository before returning
- Bind implementation via Hilt `@Binds`

Never:

- Expose DTOs outside the data module
- Return domain models directly from a DAO or API response — always map
- Put business rules inside a repository — those belong in use cases
- Make the domain interface import data classes (DTOs, entities)
