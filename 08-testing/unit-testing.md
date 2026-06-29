# Unit Testing

## Purpose

Unit tests verify that ViewModels behave correctly in isolation.

Every ViewModel should have tests covering: initial state, each event, all success paths, all failure paths.

---

## Test Infrastructure

### MainDispatcherRule

Replaces `Dispatchers.Main` with a test dispatcher so coroutines run synchronously.

```kotlin
// test/base/MainDispatcherRule.kt
class MainDispatcherRule(
    val dispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

### BaseTest

Provides `beforeTest()` and `afterTest()` hooks. Every ViewModel test extends this.

```kotlin
// test/base/BaseTest.kt
abstract class BaseTest {
    @Before fun baseBefore() = beforeTest()
    @After fun baseAfter() = afterTest()
    protected open fun beforeTest() = Unit
    protected open fun afterTest() = Unit
}
```

---

## ViewModel Test Structure

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class GroupViewModelTest : BaseTest() {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var repository: FakeGroupRepository

    override fun beforeTest() {
        repository = FakeGroupRepository()
    }

    // ─── Initial state ────────────────────────────────────────────────

    @Test
    fun `initial state has empty fields and disabled button`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()
        val state = viewModel.state.value

        assertEquals("", state.groupName)
        assertEquals("", state.groupLabel)
        assertEquals(false, state.enabledButton)
        assertIs<GroupContract.DialogState.None>(state.dialogState)
    }

    // ─── Events ───────────────────────────────────────────────────────

    @Test
    fun `OnNameChanged with valid value should update name and clear error`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()

        viewModel.event(GroupContract.Event.OnNameChanged("Algebra I"))

        val state = viewModel.state.value
        assertEquals("Algebra I", state.groupName)
        assertNull(state.groupNameError)
    }

    @Test
    fun `OnNameChanged with blank value should set error`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()

        viewModel.event(GroupContract.Event.OnNameChanged(""))

        val state = viewModel.state.value
        assertNotNull(state.groupNameError)
        assertEquals(false, state.enabledButton)
    }

    @Test
    fun `OnSubmitClick with valid form should show loading then success`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()
        fillValidForm(viewModel)

        viewModel.event(GroupContract.Event.OnSubmitClick)
        advanceUntilIdle()

        assertIs<GroupContract.DialogState.Success>(viewModel.state.value.dialogState)
    }

    @Test
    fun `OnSubmitClick when repository fails should show error dialog`() = runTest(mainDispatcherRule.dispatcher) {
        repository.shouldFail = true
        val viewModel = buildViewModel()
        fillValidForm(viewModel)

        viewModel.event(GroupContract.Event.OnSubmitClick)
        advanceUntilIdle()

        assertIs<GroupContract.DialogState.Error>(viewModel.state.value.dialogState)
    }

    // ─── Effects ─────────────────────────────────────────────────────

    @Test
    fun `OnDialogDismissed should emit NavigateBack effect`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()

        viewModel.event(GroupContract.Event.OnDialogDismissed)

        val effect = withTimeout(1_000) { viewModel.effect.first() }
        assertEquals(GroupContract.Effect.NavigateBack, effect)
    }

    @Test
    fun `OnCancelClick should emit NavigateBack effect`() = runTest(mainDispatcherRule.dispatcher) {
        val viewModel = buildViewModel()

        viewModel.event(GroupContract.Event.OnCancelClick)

        val effect = withTimeout(1_000) { viewModel.effect.first() }
        assertEquals(GroupContract.Effect.NavigateBack, effect)
    }

    // ─── Helpers ─────────────────────────────────────────────────────

    private fun buildViewModel() = GroupViewModel(
        context = mockk(relaxed = true),
        saveGroupUseCase = SaveGroupUseCase(repository),
    )

    private fun fillValidForm(viewModel: GroupViewModel) {
        viewModel.event(GroupContract.Event.OnNameChanged("Algebra I"))
        viewModel.event(GroupContract.Event.OnLabelChanged("A1"))
    }
}
```

---

## Coroutine Test Helpers

| Helper | When to use |
|--------|------------|
| `advanceUntilIdle()` | After triggering a coroutine, wait for it to finish |
| `advanceTimeBy(ms)` | Simulate time passing (timeouts, delays) |
| `runTest(mainDispatcherRule.dispatcher)` | Always — standard test coroutine scope |
| `withTimeout(1_000) { flow.first() }` | Collect a single effect with a timeout |

---

## Effect Collection in Tests

Collect multiple effects during a test:

```kotlin
val emittedEffects = mutableListOf<GroupContract.Effect>()
val collectJob = launch { viewModel.effect.collect { emittedEffects += it } }

// ... trigger events ...
advanceUntilIdle()

assertTrue(emittedEffects.contains(GroupContract.Effect.NavigateBack))
collectJob.cancel()
```

---

## Test Location

```
presentation/src/test/
└── feature/
    └── groups/
        └── GroupViewModelTest.kt
```

---

## Rules

Always:

- Use `FakeXxxRepository` — not mocks — for repositories (see [fake-repositories.md](fake-repositories.md))
- Use `MainDispatcherRule` — every ViewModel test needs it
- `advanceUntilIdle()` after every event that triggers a coroutine
- Test initial state, every event path, every failure path

Never:

- Test implementation details (private methods, internal state changes mid-coroutine)
- Use real network or database in unit tests
- Sleep or use `Thread.sleep` — use `advanceTimeBy` instead
