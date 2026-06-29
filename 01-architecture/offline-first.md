# Offline-First

## Strategy

Offline-first means the app shows cached data immediately and syncs with the network in the background.

The user sees content instantly — no loading spinner on every launch.

---

## Data Flow

```
UI observes local DB (Flow)
       ↓
Screen shows cached data immediately
       ↓
ViewModel triggers background refresh
       ↓
Repository fetches from API → writes to local DB
       ↓
DB Flow emits new data → UI updates automatically
```

---

## Repository Pattern

```kotlin
// domain/repository/GroupRepository.kt
interface GroupRepository {
    fun observeGroups(): Flow<Either<Failure, List<GroupModel>>>
    suspend fun refreshGroups(): Either<Failure, Unit>
}

// data/repository/GroupRepositoryImpl.kt
class GroupRepositoryImpl @Inject constructor(
    private val apiService: GroupApiService,
    private val groupDao: GroupDao,
) : GroupRepository {

    // Emit from local DB — never blocks, always fast
    override fun observeGroups(): Flow<Either<Failure, List<GroupModel>>> =
        groupDao.observeGroups().map { entities ->
            Either.Success(entities.map { it.toDomain() })
        }

    // Fetch from network, write to DB — DB Flow propagates changes to the UI
    override suspend fun refreshGroups(): Either<Failure, Unit> =
        safeApiCall {
            val groups = apiService.getGroups()
            groupDao.upsertGroups(groups.map { it.toEntity() })
        }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val groupRepository: GroupRepository,
) : ViewModel(), HomeContract {

    private val _state = MutableStateFlow(HomeContract.UiState())
    override val state: StateFlow<HomeContract.UiState> = _state.asStateFlow()

    init {
        observeGroups()
        refreshGroups()
    }

    private fun observeGroups() {
        viewModelScope.launch {
            groupRepository.observeGroups().collect { result ->
                result.fold(
                    onSuccess = { groups ->
                        updateState { copy(groups = groups.map { it.toUiItem() }, isLoading = false) }
                    },
                    onError = { failure ->
                        // Only show error if no cached data
                        if (_state.value.groups.isEmpty()) {
                            updateState { copy(isLoading = false, errorMessage = "...") }
                        }
                    },
                )
            }
        }
    }

    private fun refreshGroups() {
        viewModelScope.launch {
            groupRepository.refreshGroups()
                .fold(
                    onSuccess = { /* DB observer picks up the change */ },
                    onError = { failure ->
                        if (_state.value.groups.isEmpty()) {
                            updateState { copy(errorMessage = "...") }
                        }
                    },
                )
        }
    }
}
```

Key: `observeGroups()` and `refreshGroups()` run in parallel. The DB observer always has data — refresh just keeps it fresh.

---

## When Not to Use Offline-First

| Scenario | Reason to skip |
|----------|---------------|
| Auth / login | Session data changes on server; stale cache is a security risk |
| Payments / financial data | Must always reflect real-time server state |
| Feature that writes only | No reads to cache; write-then-navigate pattern is simpler |
| Prototype / MVP | Adds complexity; add it when the product demands it |

---

## Rules

Always:

- Observe from local DB — not from a network response directly
- Run refresh in parallel with the DB observation
- Use `@Upsert` (not `@Insert(REPLACE)`) to avoid deleting rows with foreign key constraints
- Emit the cached data even if refresh fails — stale data is better than no data

Never:

- Block UI on network response in an offline-first feature
- Clear the local DB before inserting new data — use upsert to preserve rows the network didn't return
- Show a full-screen loading state if cached data is already available
