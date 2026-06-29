# Room (Local Database)

## When to Use

Room is used for offline-first features where data must be available without a network connection, or for caching API responses to reduce network calls.

For features that are always online with no caching needs, skip Room.

---

## Entity

```kotlin
// data/local/entity/GroupEntity.kt
@Entity(tableName = "groups")
data class GroupEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "label") val label: String,
    @ColumnInfo(name = "created_at") val createdAt: String,
)
```

---

## DAO

```kotlin
// data/local/dao/GroupDao.kt
@Dao
interface GroupDao {

    @Query("SELECT * FROM groups ORDER BY created_at DESC")
    fun observeGroups(): Flow<List<GroupEntity>>

    @Query("SELECT * FROM groups WHERE id = :id")
    suspend fun getGroupById(id: String): GroupEntity?

    @Upsert
    suspend fun upsertGroups(groups: List<GroupEntity>)

    @Delete
    suspend fun deleteGroup(group: GroupEntity)

    @Query("DELETE FROM groups WHERE id = :id")
    suspend fun deleteGroupById(id: String)
}
```

Use `Flow` for queries that the UI should react to automatically.
Use `suspend` for one-shot reads and writes.

---

## Database

```kotlin
// data/local/AppDatabase.kt
@Database(
    entities = [GroupEntity::class],
    version = 1,
    exportSchema = true,
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun groupDao(): GroupDao
}
```

DI:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
            .fallbackToDestructiveMigration()
            .build()

    @Provides
    fun provideGroupDao(db: AppDatabase): GroupDao = db.groupDao()
}
```

---

## Mapper

```kotlin
fun GroupEntity.toDomain(): GroupModel = GroupModel(
    id = id,
    name = name,
    label = label,
    createdAt = LocalDateTime.parse(createdAt),
)

fun GroupModel.toEntity(): GroupEntity = GroupEntity(
    id = id,
    name = name,
    label = label,
    createdAt = createdAt.toString(),
)
```

---

## Cache-First Repository

```kotlin
class GroupRepositoryImpl @Inject constructor(
    private val apiService: GroupApiService,
    private val groupDao: GroupDao,
) : GroupRepository {

    override fun observeGroups(): Flow<Either<Failure, List<GroupModel>>> =
        groupDao.observeGroups().map { entities ->
            Either.Success(entities.map { it.toDomain() })
        }

    override suspend fun refreshGroups(): Either<Failure, Unit> =
        safeApiCall {
            val groups = apiService.getGroups()
            groupDao.upsertGroups(groups.map { it.toEntity() })
        }
}
```

See [offline-first.md](../01-architecture/offline-first.md) for the full cache-first flow.

---

## Rules

Always:

- Use `@Upsert` for writes that may insert or update
- Return `Flow` from DAO queries the UI observes; `suspend` for one-shot operations
- Export schema (`exportSchema = true`) for migration history
- Write migrations for schema changes in production — `fallbackToDestructiveMigration` is fine for dev only

Never:

- Call DAO methods on the main thread — Room enforces coroutine/thread checks
- Store domain models in the database — use entities and map at the repository boundary
- Access `AppDatabase` directly from the repository — inject the DAO
