# Pagination

## Strategy

Use Paging 3 when a list can grow unbounded and loading all items at once would impact performance or memory.

For lists with a known small upper bound (< 200 items), skip Paging and load all at once.

---

## Setup

```toml
# libs.versions.toml
paging = "3.3.2"
paging-compose = "3.3.2"

[libraries]
paging-runtime = { group = "androidx.paging", name = "paging-runtime", version.ref = "paging" }
paging-compose = { group = "androidx.paging", name = "paging-compose", version.ref = "paging-compose" }
```

---

## PagingSource

```kotlin
// data/paging/GroupPagingSource.kt
class GroupPagingSource @Inject constructor(
    private val apiService: GroupApiService,
) : PagingSource<Int, GroupModel>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, GroupModel> {
        val page = params.key ?: 0
        return try {
            val response = apiService.getGroupsPaged(page = page, size = params.loadSize)
            LoadResult.Page(
                data = response.items.map { it.toDomain() },
                prevKey = if (page == 0) null else page - 1,
                nextKey = if (response.items.isEmpty()) null else page + 1,
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, GroupModel>): Int? =
        state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
}
```

---

## Repository

```kotlin
// domain/repository/GroupRepository.kt (add)
interface GroupRepository {
    fun getGroupsPaged(): Flow<PagingData<GroupModel>>
}

// data/repository/GroupRepositoryImpl.kt
class GroupRepositoryImpl @Inject constructor(
    private val pagingSourceFactory: GroupPagingSourceFactory,
) : GroupRepository {

    override fun getGroupsPaged(): Flow<PagingData<GroupModel>> =
        Pager(
            config = PagingConfig(
                pageSize = 20,
                prefetchDistance = 5,
                enablePlaceholders = false,
            ),
            pagingSourceFactory = { pagingSourceFactory.create() },
        ).flow
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class GroupListViewModel @Inject constructor(
    getGroupsPaged: GetGroupsPagedUseCase,
) : ViewModel() {

    val groups: Flow<PagingData<GroupUiItem>> = getGroupsPaged()
        .map { pagingData -> pagingData.map { it.toUiItem() } }
        .cachedIn(viewModelScope)
}
```

`cachedIn(viewModelScope)` — caches pages across recompositions and screen rotations.

---

## Screen

```kotlin
@Composable
private fun GroupListContent(
    viewModel: GroupListViewModel = hiltViewModel(),
) {
    val groups = viewModel.groups.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = groups.itemCount,
            key = groups.itemKey { it.id },
        ) { index ->
            val group = groups[index]
            if (group != null) {
                GroupCard(name = group.name, onClick = { ... })
            }
        }

        // Loading / error states
        item {
            when {
                groups.loadState.append is LoadState.Loading -> {
                    CircularProgressIndicator(modifier = Modifier.fillMaxWidth().wrapContentWidth())
                }
                groups.loadState.append is LoadState.Error -> {
                    AppButton(text = "Retry", onClick = { groups.retry() })
                }
            }
        }
    }
}
```

---

## Rules

Always:

- `cachedIn(viewModelScope)` in ViewModel — prevents re-fetching on recomposition
- Handle `LoadState.Loading` and `LoadState.Error` in the UI for both `refresh` and `append`
- Map to `UiItem` in the ViewModel before exposing `PagingData` to the screen

Never:

- Collect `PagingData` outside a composable context — always use `collectAsLazyPagingItems()`
- Use `PagingData` for lists with a known small bound
- Skip `cachedIn` — without it, the list re-fetches from page 0 on every recomposition
