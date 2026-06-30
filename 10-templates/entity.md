# Room Entity Template

## Standard Entity

```kotlin
// data/local/entity/GroupEntity.kt
@Entity(tableName = "groups")
data class GroupEntity(
    @PrimaryKey
    @ColumnInfo(name = "id")         val id: String,
    @ColumnInfo(name = "name")       val name: String,
    @ColumnInfo(name = "label")      val label: String,
    @ColumnInfo(name = "teacher_id") val teacherId: String,
    @ColumnInfo(name = "is_active")  val isActive: Boolean = true,
    @ColumnInfo(name = "created_at") val createdAt: String,
)
```

---

## Mapping Extensions

```kotlin
// Entity → Domain
fun GroupEntity.toDomain() = GroupModel(
    id        = id,
    name      = name,
    label     = label,
    teacherId = teacherId,
    isActive  = isActive,
    createdAt = Instant.parse(createdAt),
)

// Domain → Entity
fun GroupModel.toEntity() = GroupEntity(
    id        = id,
    name      = name,
    label     = label,
    teacherId = teacherId,
    isActive  = isActive,
    createdAt = createdAt.toString(),
)
```

---

## Entity with Foreign Key

```kotlin
@Entity(
    tableName = "students",
    foreignKeys = [
        ForeignKey(
            entity = GroupEntity::class,
            parentColumns = ["id"],
            childColumns = ["group_id"],
            onDelete = ForeignKey.CASCADE,
        )
    ],
    indices = [Index(value = ["group_id"])],
)
data class StudentEntity(
    @PrimaryKey
    @ColumnInfo(name = "id")       val id: String,
    @ColumnInfo(name = "name")     val name: String,
    @ColumnInfo(name = "group_id") val groupId: String,
)
```

Always index foreign key columns — Room warns if you don't and queries will be slow.

---

## Embedded Object

```kotlin
data class AddressEmbedded(
    @ColumnInfo(name = "street") val street: String,
    @ColumnInfo(name = "city")   val city: String,
)

@Entity(tableName = "schools")
data class SchoolEntity(
    @PrimaryKey
    @ColumnInfo(name = "id")   val id: String,
    @ColumnInfo(name = "name") val name: String,
    @Embedded val address: AddressEmbedded,
)
```

---

## Relation (Junction)

```kotlin
data class GroupWithStudents(
    @Embedded val group: GroupEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "group_id",
    )
    val students: List<StudentEntity>,
)

// DAO
@Transaction
@Query("SELECT * FROM groups WHERE id = :groupId")
fun observeGroupWithStudents(groupId: String): Flow<GroupWithStudents?>
```

---

## Database Registration

Every entity must be registered in `AppDatabase`:

```kotlin
@Database(
    entities = [
        GroupEntity::class,
        StudentEntity::class,
        SchoolEntity::class,
    ],
    version = 1,
    exportSchema = true,
)
abstract class AppDatabase : RoomDatabase()
```

`exportSchema = true` is required. Commit schema JSON files to version control.

---

## Migrations

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE groups ADD COLUMN is_active INTEGER NOT NULL DEFAULT 1")
    }
}
```

Never use `fallbackToDestructiveMigration()` in production builds. Always write explicit migrations.

---

## Rules

Always:

- Use `@ColumnInfo(name = "snake_case")` on every field
- Index every foreign key column
- Export schema and commit the JSON
- Write explicit migrations for every schema change

Never:

- Use `fallbackToDestructiveMigration()` in production
- Store complex objects as JSON blobs — normalize them or use `@Embedded`
- Use `@Entity` for domain models
- Let entity types appear in the domain or presentation layers
