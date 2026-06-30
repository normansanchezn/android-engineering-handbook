# DTO Template

## Standard DTO

```kotlin
// data/remote/dto/GroupDto.kt
@Serializable
data class GroupDto(
    @SerialName("id")          val id: String,
    @SerialName("name")        val name: String,
    @SerialName("label")       val label: String,
    @SerialName("teacher_id")  val teacherId: String,
    @SerialName("is_active")   val isActive: Boolean = true,
    @SerialName("created_at")  val createdAt: String,
    @SerialName("avatar_url")  val avatarUrl: String? = null,
)
```

---

## Mapping Extensions

Mapping functions are extension functions on the DTO — not methods on domain models.

```kotlin
// Extension: DTO → Domain
fun GroupDto.toDomain() = GroupModel(
    id        = id,
    name      = name,
    label     = label,
    teacherId = teacherId,
    isActive  = isActive,
    createdAt = Instant.parse(createdAt),
    avatarUrl = avatarUrl,
)

// Extension: DTO → Entity (for local persistence)
fun GroupDto.toEntity() = GroupEntity(
    id        = id,
    name      = name,
    label     = label,
    teacherId = teacherId,
    isActive  = isActive,
    createdAt = createdAt,
)

// Extension: Domain → DTO (for requests)
fun GroupModel.toDto() = GroupDto(
    id        = id,
    name      = name,
    label     = label,
    teacherId = teacherId,
    isActive  = isActive,
    createdAt = createdAt.toString(),
    avatarUrl = avatarUrl,
)
```

---

## Request Body DTO

For POST/PUT request bodies, create a separate DTO:

```kotlin
@Serializable
data class CreateGroupRequest(
    @SerialName("name")       val name: String,
    @SerialName("label")      val label: String,
    @SerialName("teacher_id") val teacherId: String,
)

fun GroupModel.toCreateRequest() = CreateGroupRequest(
    name      = name,
    label     = label,
    teacherId = teacherId,
)
```

Separate request and response DTOs — they rarely have identical shapes.

---

## Nested DTO

```kotlin
@Serializable
data class CourseDto(
    @SerialName("id")      val id: String,
    @SerialName("name")    val name: String,
    @SerialName("teacher") val teacher: TeacherDto,
    @SerialName("groups")  val groups: List<GroupDto> = emptyList(),
)

fun CourseDto.toDomain() = CourseModel(
    id      = id,
    name    = name,
    teacher = teacher.toDomain(),
    groups  = groups.map { it.toDomain() },
)
```

---

## Rules

Always:

- Annotate DTOs with `@Serializable`
- Use `@SerialName` on every field — never rely on property name matching
- Provide `= null` default for optional nullable fields
- Provide `= emptyList()` for optional list fields
- Keep mapping extensions in the same file as the DTO

Never:

- Let DTOs appear in the domain or presentation layers
- Put domain logic inside mapping functions — map field to field only
- Reuse DTOs as request and response bodies when their shapes differ
- Parse dates inside the DTO — parse in `toDomain()`
