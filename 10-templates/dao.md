# DAO Template

## Standard DAO

```kotlin
// data/local/dao/GroupDao.kt
@Dao
interface GroupDao {

    @Query("SELECT * FROM groups ORDER BY name ASC")
    fun observeGroups(): Flow<List<GroupEntity>>

    @Query("SELECT * FROM groups WHERE id = :id")
    suspend fun getGroupById(id: String): GroupEntity?

    @Query("SELECT * FROM groups WHERE teacher_id = :teacherId")
    fun observeGroupsByTeacher(teacherId: String): Flow<List<GroupEntity>>

    @Upsert
    suspend fun upsertGroups(groups: List<GroupEntity>)

    @Upsert
    suspend fun upsertGroup(group: GroupEntity)

    @Query("DELETE FROM groups WHERE id = :id")
    suspend fun deleteGroupById(id: String)

    @Query("DELETE FROM groups")
    suspend fun deleteAll()
}
```

---

## Key Decisions

| Decision | Rule |
|----------|------|
| Observe vs suspend | Use `Flow<T>` for reactive reads; `suspend` for one-shot reads and writes |
| Insert vs Upsert | Always use `@Upsert` — avoids replacing rows with foreign key constraints |
| Return type for writes | `suspend` with no return value — repositories wrap errors via `safeApiCall` |
| Nullable single reads | `suspend fun getXxx(): XxxEntity?` — null means not found |

---

## Query with Parameters

```kotlin
@Query("SELECT * FROM groups WHERE name LIKE '%' || :query || '%'")
fun searchGroups(query: String): Flow<List<GroupEntity>>

@Query("SELECT * FROM groups LIMIT :limit OFFSET :offset")
suspend fun getGroupsPaged(limit: Int, offset: Int): List<GroupEntity>

@Query("UPDATE groups SET is_active = :isActive WHERE id = :id")
suspend fun setGroupActive(id: String, isActive: Boolean)
```

---

## Transaction

Use `@Transaction` when an operation spans multiple DAO calls that must succeed atomically:

```kotlin
@Transaction
suspend fun replaceAllGroups(groups: List<GroupEntity>) {
    deleteAll()
    upsertGroups(groups)
}
```

---

## DI

```kotlin
// AppDatabase.kt
@Database(
    entities = [GroupEntity::class, StudentEntity::class],
    version = 1,
    exportSchema = true,
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun groupDao(): GroupDao
    abstract fun studentDao(): StudentDao
}

// data/di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .addMigrations(MIGRATION_1_2)
            .build()

    @Provides
    fun provideGroupDao(database: AppDatabase): GroupDao = database.groupDao()
}
```

---

## Rules

Always:

- Return `Flow` for reactive list queries
- Use `@Upsert` for insertions — never `@Insert(onConflict = OnConflictStrategy.REPLACE)`
- Make single-entity reads return nullable (`XxxEntity?`)
- Keep DAOs free of business logic — queries only

Never:

- Map to domain models inside a DAO
- Wrap DAO calls in `safeApiCall` — that belongs in the repository
- Perform multiple related writes outside a `@Transaction`
