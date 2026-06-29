# Example: Home Feature

Read-only list screen. Shows how to load data on init, display a list, handle empty/error states, and navigate to a detail screen.

---

## File Structure

```
presentation/feature/home/
├── HomeRouter.kt
├── HomeNavGraph.kt
├── contract/
│   └── HomeContract.kt
├── viewModel/
│   └── HomeViewModel.kt
└── ui/
    └── HomeScreen.kt

domain/usecases/
└── GetGroupsUseCase.kt
```

---

## Contract

```kotlin
// presentation/feature/home/contract/HomeContract.kt
interface HomeContract : MVIContract<HomeContract.UiState, HomeContract.Effect, HomeContract.Event> {

    data class UiState(
        val groups: List<GroupUiItem> = emptyList(),
        val isLoading: Boolean = false,
        val errorMessage: String? = null,
        val screenUiModel: HomeScreenUiModel = HomeScreenUiModel(),
    )

    data class HomeScreenUiModel(
        val toolbarTitle: String = "",
        val emptyStateMessage: String = "",
        val addButtonDescription: String = "",
    )

    data class GroupUiItem(
        val id: String,
        val name: String,
        val label: String,
    )

    sealed interface Effect {
        data class NavigateToDetail(val groupId: String) : Effect
        data object NavigateToCreate : Effect
    }

    sealed interface Event {
        data class OnGroupClick(val groupId: String) : Event
        data object OnAddClick : Event
        data object OnRetryClick : Event
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val getGroupsUseCase: GetGroupsUseCase,
) : ViewModel(), HomeContract {

    private val _state = MutableStateFlow(HomeContract.UiState())
    override val state: StateFlow<HomeContract.UiState> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<HomeContract.Effect>()
    override val effect: SharedFlow<HomeContract.Effect> = _effect.asSharedFlow()

    init {
        updateState { copy(screenUiModel = createUiStrings()) }
        loadGroups()
    }

    override fun event(event: HomeContract.Event) {
        when (event) {
            is HomeContract.Event.OnGroupClick -> sendEffect(HomeContract.Effect.NavigateToDetail(event.groupId))
            is HomeContract.Event.OnAddClick -> sendEffect(HomeContract.Effect.NavigateToCreate)
            is HomeContract.Event.OnRetryClick -> loadGroups()
        }
    }

    private fun loadGroups() {
        updateState { copy(isLoading = true, errorMessage = null) }
        viewModelScope.launch {
            getGroupsUseCase()
                .fold(
                    onSuccess = { groups ->
                        updateState {
                            copy(
                                isLoading = false,
                                groups = groups.map { it.toUiItem() },
                            )
                        }
                    },
                    onError = { failure ->
                        updateState {
                            copy(
                                isLoading = false,
                                errorMessage = context.getString(R.string.error_loading_groups),
                            )
                        }
                    },
                )
        }
    }

    private fun GroupModel.toUiItem() = HomeContract.GroupUiItem(
        id = id,
        name = name,
        label = label,
    )

    private fun createUiStrings() = HomeContract.HomeScreenUiModel(
        toolbarTitle = context.getString(R.string.home_title),
        emptyStateMessage = context.getString(R.string.home_empty_state),
        addButtonDescription = context.getString(R.string.home_add_button),
    )

    private fun updateState(update: HomeContract.UiState.() -> HomeContract.UiState) {
        _state.update { it.update() }
    }

    private fun sendEffect(effect: HomeContract.Effect) {
        viewModelScope.launch { _effect.emit(effect) }
    }
}
```

---

## Screen

```kotlin
@Composable
fun HomeScreen(
    onNavigateToDetail: (String) -> Unit,
    onNavigateToCreate: () -> Unit,
    viewModel: HomeViewModel = hiltViewModel(),
) {
    val uiState by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is HomeContract.Effect.NavigateToDetail -> onNavigateToDetail(effect.groupId)
                is HomeContract.Effect.NavigateToCreate -> onNavigateToCreate()
            }
        }
    }

    HomeBody(uiState = uiState, onEvent = viewModel::event)
}

@Composable
private fun HomeBody(
    uiState: HomeContract.UiState,
    onEvent: (HomeContract.Event) -> Unit,
) {
    Scaffold(
        topBar = { AppToolbar(title = uiState.screenUiModel.toolbarTitle) },
        floatingActionButton = {
            FloatingActionButton(onClick = { onEvent(HomeContract.Event.OnAddClick) }) {
                Icon(
                    imageVector = Icons.Default.Add,
                    contentDescription = uiState.screenUiModel.addButtonDescription,
                )
            }
        },
    ) { padding ->
        HomeContent(
            uiState = uiState,
            onEvent = onEvent,
            modifier = Modifier.padding(padding),
        )
    }
}

@Composable
private fun HomeContent(
    uiState: HomeContract.UiState,
    onEvent: (HomeContract.Event) -> Unit,
    modifier: Modifier = Modifier,
) {
    Box(modifier = modifier.fillMaxSize()) {
        when {
            uiState.isLoading -> CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))

            uiState.errorMessage != null -> ErrorState(
                message = uiState.errorMessage,
                onRetry = { onEvent(HomeContract.Event.OnRetryClick) },
                modifier = Modifier.align(Alignment.Center),
            )

            uiState.groups.isEmpty() -> Text(
                text = uiState.screenUiModel.emptyStateMessage,
                modifier = Modifier.align(Alignment.Center),
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
            )

            else -> GroupList(
                groups = uiState.groups,
                onGroupClick = { id -> onEvent(HomeContract.Event.OnGroupClick(id)) },
            )
        }
    }
}

@Composable
private fun GroupList(
    groups: List<HomeContract.GroupUiItem>,
    onGroupClick: (String) -> Unit,
) {
    LazyColumn(
        contentPadding = PaddingValues(SpacingM),
        verticalArrangement = Arrangement.spacedBy(SpacingS),
    ) {
        items(
            items = groups,
            key = { it.id },
        ) { group ->
            AppCard(onClick = { onGroupClick(group.id) }) {
                Text(text = group.name, style = MaterialTheme.typography.titleMedium)
                Text(text = group.label, style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}

@Composable
private fun ErrorState(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier.padding(SpacingL),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(SpacingM),
    ) {
        Text(text = message, style = MaterialTheme.typography.bodyMedium)
        AppButton(text = "Retry", onClick = onRetry, modifier = Modifier.fillMaxWidth(0.5f))
    }
}
```

---

## Test

```kotlin
class HomeViewModelTest : BaseTest() {

    private val fakeGroupRepository = FakeGroupRepository()
    private lateinit var viewModel: HomeViewModel

    @Before
    fun setup() {
        val context = mockk<Context>(relaxed = true)
        viewModel = HomeViewModel(context, GetGroupsUseCase(fakeGroupRepository))
    }

    @Test
    fun `initial load shows groups on success`() = runTest {
        advanceUntilIdle()
        assertTrue(viewModel.state.value.groups.isNotEmpty())
        assertFalse(viewModel.state.value.isLoading)
    }

    @Test
    fun `initial load shows error on failure`() = runTest {
        fakeGroupRepository.shouldFail = true
        val context = mockk<Context>(relaxed = true)
        viewModel = HomeViewModel(context, GetGroupsUseCase(fakeGroupRepository))

        advanceUntilIdle()
        assertNotNull(viewModel.state.value.errorMessage)
    }

    @Test
    fun `OnGroupClick emits NavigateToDetail effect`() = runTest {
        val effects = mutableListOf<HomeContract.Effect>()
        backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) {
            viewModel.effect.toList(effects)
        }

        advanceUntilIdle()
        val firstGroupId = viewModel.state.value.groups.first().id
        viewModel.event(HomeContract.Event.OnGroupClick(firstGroupId))

        assertContains(effects, HomeContract.Effect.NavigateToDetail(firstGroupId))
    }

    @Test
    fun `OnRetryClick reloads groups`() = runTest {
        fakeGroupRepository.shouldFail = true
        viewModel = HomeViewModel(mockk(relaxed = true), GetGroupsUseCase(fakeGroupRepository))
        advanceUntilIdle()
        assertNotNull(viewModel.state.value.errorMessage)

        fakeGroupRepository.shouldFail = false
        viewModel.event(HomeContract.Event.OnRetryClick)
        advanceUntilIdle()

        assertNull(viewModel.state.value.errorMessage)
        assertTrue(viewModel.state.value.groups.isNotEmpty())
    }
}
```
