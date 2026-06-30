# Integration Testing

## Purpose

Integration tests verify that components work together correctly. They test real implementations against real dependencies — no fakes, no mocks at the boundary being tested.

The most valuable integration tests are data layer tests: DAOs against an in-memory Room database, and repositories against real DAOs and fake remote data sources.

---

## Room In-Memory Database

```kotlin
// androidTest/data/db/GroupDaoTest.kt
@RunWith(AndroidJUnit4::class)
class GroupDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var dao: GroupDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            context = ApplicationProvider.getApplicationContext(),
            klass = AppDatabase::class.java,
        )
            .allowMainThreadQueries()  // test only
            .build()
        dao = database.groupDao()
    }

    @After
    fun tearDown() {
        database.close()
    }

    @Test
    fun upsertAndObserveGroups() = runTest {
        val groups = listOf(
            GroupEntity(id = "1", name = "Algebra I",  label = "A1", teacherId = "t1"),
            GroupEntity(id = "2", name = "Geometry",   label = "G1", teacherId = "t1"),
        )

        dao.upsertGroups(groups)

        val result = dao.observeGroups().first()
        assertEquals(2, result.size)
        assertEquals("Algebra I", result.first().name)
    }

    @Test
    fun upsertReplacesExistingRow() = runTest {
        dao.upsertGroups(listOf(GroupEntity(id = "1", name = "Old Name", label = "A1", teacherId = "t1")))
        dao.upsertGroups(listOf(GroupEntity(id = "1", name = "New Name", label = "A1", teacherId = "t1")))

        val result = dao.observeGroups().first()
        assertEquals(1, result.size)
        assertEquals("New Name", result.first().name)
    }

    @Test
    fun deleteGroupById() = runTest {
        dao.upsertGroups(listOf(GroupEntity(id = "1", name = "Algebra I", label = "A1", teacherId = "t1")))
        dao.deleteGroupById("1")

        val result = dao.observeGroups().first()
        assertTrue(result.isEmpty())
    }
}
```

---

## Repository Integration Test

Tests the repository implementation with a real in-memory database and a fake remote data source:

```kotlin
@RunWith(AndroidJUnit4::class)
class GroupRepositoryImplTest {

    private lateinit var database: AppDatabase
    private lateinit var repository: GroupRepositoryImpl
    private lateinit var fakeRemote: FakeGroupRemoteDataSource

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java,
        ).allowMainThreadQueries().build()

        fakeRemote = FakeGroupRemoteDataSource()

        repository = GroupRepositoryImpl(
            remoteDataSource = fakeRemote,
            localDataSource = GroupLocalDataSourceImpl(database.groupDao()),
        )
    }

    @After
    fun tearDown() = database.close()

    @Test
    fun refreshGroupsWritesToDatabase() = runTest {
        fakeRemote.groups = listOf(
            GroupDto(id = "1", name = "Algebra I", label = "A1", teacherId = "t1"),
        )

        repository.refreshGroups()

        repository.observeGroups().test {
            val result = (awaitItem() as Either.Success).value
            assertEquals(1, result.size)
            assertEquals("Algebra I", result.first().name)
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun refreshGroupsReturnsNetworkErrorOnFailure() = runTest {
        fakeRemote.shouldFail = true

        val result = repository.refreshGroups()

        assertIs<Either.Error<Failure.NetworkError>>(result)
    }

    @Test
    fun observeGroupsEmitsCachedDataWhileRefreshFails() = runTest {
        // Seed the database with stale data
        database.groupDao().upsertGroups(listOf(GroupEntity(id = "1", name = "Stale", label = "S1", teacherId = "t1")))
        fakeRemote.shouldFail = true

        repository.observeGroups().test {
            val result = (awaitItem() as Either.Success).value
            assertEquals(1, result.size)  // stale data still visible
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## Fake Remote Data Source for Integration Tests

```kotlin
class FakeGroupRemoteDataSource : GroupRemoteDataSource {
    var groups: List<GroupDto> = emptyList()
    var shouldFail = false

    override suspend fun getGroups(): List<GroupDto> {
        if (shouldFail) throw IOException("Network error")
        return groups
    }

    override suspend fun createGroup(request: CreateGroupRequest): GroupDto {
        if (shouldFail) throw IOException("Network error")
        val dto = GroupDto(id = UUID.randomUUID().toString(), name = request.name, label = request.label, teacherId = request.teacherId)
        groups = groups + dto
        return dto
    }
}
```

---

## Hilt Integration Tests

For components that use Hilt DI, use `@HiltAndroidTest`:

```kotlin
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class GroupRepositoryHiltTest {

    @get:Rule
    val hiltRule = HiltAndroidRule(this)

    @Inject
    lateinit var repository: GroupRepository

    @Before
    fun setup() = hiltRule.inject()

    @Test
    fun repositoryIsInjected() {
        assertNotNull(repository)
    }
}
```

Requires `@AndroidEntryPoint` on the test application or a `CustomTestRunner`.

---

## Test Location

```
data/src/androidTest/kotlin/data/
├── db/
│   ├── GroupDaoTest.kt
│   └── StudentDaoTest.kt
└── repository/
    └── GroupRepositoryImplTest.kt
```

Integration tests live in `androidTest/` — they require an Android context and run on a device or emulator.

---

## Rules

Always:

- Use `Room.inMemoryDatabaseBuilder` — never a real file-backed database in tests
- Close the database in `@After`
- Allow main thread queries in integration tests only (`allowMainThreadQueries()`)
- Use fake remote data sources, not mocked ones, in repository tests

Never:

- Run integration tests with a real network — all remote calls use fakes
- Share database instances between tests — create a fresh instance per test class
- Use `@Before` to run data setup in parallel — Room's test builder is synchronous
