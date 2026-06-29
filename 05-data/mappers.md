# Mappers

## Purpose

Mappers translate between layer-specific models:

```
DTO / Entity  ──toDomain()──►  Domain Model  ──toUiModel()──►  UiModel
(data layer)                   (domain layer)                  (presentation layer)
```

Each direction is explicit and unidirectional. No layer imports another layer's models.

---

## Mapper Interface (core)

```kotlin
// core/mapper/Mapper.kt
interface Mapper<Domain, Entity> {
    fun toEntity(domain: Domain): Entity
    fun toDomain(entity: Entity): Domain
}
```

Use the interface when the mapping requires injected dependencies (Context, formatters, etc.).

For simple field-to-field mappings, use extension functions instead.

---

## Extension Function Pattern (preferred)

```kotlin
// data/remote/dto/GroupDto.kt
@Serializable
data class GroupDto(
    val id: String,
    val name: String,
    val label: String,
    val teacherId: String,
    val startSchedule: String?,
    val endSchedule: String?,
)

// DTO → Domain
fun GroupDto.toDomain(): GroupModel = GroupModel(
    id = id,
    name = name,
    label = label,
    teacherId = teacherId,
    startTime = startSchedule?.let { LocalTime.parse(it) },
    endTime = endSchedule?.let { LocalTime.parse(it) },
)

// Domain → DTO (for writes)
fun GroupModel.toDto(): GroupDto = GroupDto(
    id = id,
    name = name,
    label = label,
    teacherId = teacherId,
    startSchedule = startTime?.toString(),
    endSchedule = endTime?.toString(),
)
```

---

## UiModel Mapping (presentation)

Domain → UiModel mapping happens in the ViewModel, not in the repository.

```kotlin
// presentation/feature/groups/model/GroupUiModel.kt
data class GroupUiModel(
    val id: String,
    val displayName: String,
    val scheduleText: String,
    val studentCountText: String,
    val colorValue: Long,
)

// Extension in the same file
fun GroupModel.toUiModel(context: Context): GroupUiModel = GroupUiModel(
    id = id,
    displayName = "$name — $label",
    scheduleText = formatSchedule(startTime, endTime),
    studentCountText = context.getString(R.string.students_count, studentCount),
    colorValue = generateColor(label).value,
)
```

Used in ViewModel:

```kotlin
onSuccess = { groups ->
    updateState { it.copy(groups = groups.map { it.toUiModel(context) }) }
}
```

---

## Mapper Class Pattern (for complex cases)

When mapping needs injected dependencies:

```kotlin
class AttendanceMapper @Inject constructor(
    private val dateFormatter: DateFormatter,
) : Mapper<AttendanceModel, AttendanceDto> {

    override fun toEntity(domain: AttendanceModel): AttendanceDto = AttendanceDto(
        id = domain.id,
        studentId = domain.studentId,
        date = dateFormatter.format(domain.date),
        status = domain.status.name,
    )

    override fun toDomain(entity: AttendanceDto): AttendanceModel = AttendanceModel(
        id = entity.id,
        studentId = entity.studentId,
        date = dateFormatter.parse(entity.date),
        status = AttendanceStatus.valueOf(entity.status),
    )
}
```

---

## Rules

Always:

- `toDomain()` lives in the same file or folder as the DTO — it belongs to the data layer
- `toUiModel()` lives in the presentation layer — the domain model never knows about UI
- Map in the ViewModel for domain → UiModel, not in the repository

Never:

- Map in a Composable
- Put a `toUiModel()` function in the domain layer
- Import presentation types from the data layer
- Skip the domain model layer — DTO → UiModel directly
