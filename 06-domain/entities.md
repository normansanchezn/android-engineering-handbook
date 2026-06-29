# Domain Models

## Purpose

Domain models represent business concepts in pure Kotlin.

They are the language the application speaks internally — independent of network formats, database schemas, and UI requirements.

---

## Example

```kotlin
// domain/models/GroupModel.kt
data class GroupModel(
    val id: String,
    val name: String,
    val label: String,
    val teacherId: String,
    val startTime: LocalTime?,
    val endTime: LocalTime?,
)
```

```kotlin
// domain/models/StudentModel.kt
data class StudentModel(
    val id: String,
    val firstName: String,
    val lastName: String,
    val groupId: String,
)
```

```kotlin
// domain/models/AttendanceRecord.kt
data class AttendanceRecord(
    val studentId: String,
    val date: LocalDate,
    val status: AttendanceStatus,
)

enum class AttendanceStatus { PRESENT, ABSENT, LATE }
```

---

## Folder Structure

```
domain/models/
├── group/
│   └── GroupModel.kt
├── student/
│   ├── StudentModel.kt
│   └── AttendanceRecord.kt
├── teacher/
│   └── TeacherModel.kt
└── ...
```

---

## Rules

Always:

- Pure `data class` — no annotations from third-party frameworks
- Use `java.time` types (`LocalDate`, `LocalTime`) — not string dates
- Group by business concept, not by operation
- Use `null` for optional fields — not empty string sentinels

Never:

- Add `@SerialName`, `@ColumnInfo`, `@Entity`, or any ORM/serialization annotations
- Add UI-specific fields like `val displayName: String` or `val colorValue: Long`
- Add formatted strings — those belong in UiModels
- Depend on Android, Compose, network, or database classes

---

## What Does NOT Belong Here

| Belongs here | Belongs elsewhere |
|-------------|-----------------|
| `val name: String` | `val displayName: String` (UiModel) |
| `val startTime: LocalTime?` | `val scheduleText: String` (UiModel) |
| `val status: AttendanceStatus` | `@SerialName("status")` (DTO) |
| `val id: String` | `@PrimaryKey val id: String` (Entity) |
