# Coroutine & Flow Testing

## Test Dispatchers

Two dispatcher options for `runTest`:

| Dispatcher | Behavior |
|------------|---------|
| `StandardTestDispatcher()` | Coroutines are queued — you control advancement with `advanceUntilIdle()` |
| `UnconfinedTestDispatcher()` | Coroutines run eagerly — useful for simple Flow collection |

Use `StandardTestDispatcher` (the default in `MainDispatcherRule`). It gives explicit control over coroutine execution and makes test sequences easier to reason about.

---

## Testing StateFlow

`StateFlow` holds the current value — read it directly after `advanceUntilIdle()`:

```kotlin
@Test
fun `OnNameChanged updates name in state`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()

    viewModel.event(GroupContract.Event.OnNameChanged("Algebra I"))
    advanceUntilIdle()

    assertEquals("Algebra I", viewModel.state.value.groupName)
}
```

---

## Testing SharedFlow (Effects)

Collect effects into a list using a background job:

```kotlin
@Test
fun `OnCancelClick emits NavigateBack effect`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()
    val effects = mutableListOf<GroupContract.Effect>()

    val collectJob = launch(UnconfinedTestDispatcher()) {
        viewModel.effect.collect { effects += it }
    }

    viewModel.event(GroupContract.Event.OnCancelClick)
    advanceUntilIdle()

    assertTrue(effects.contains(GroupContract.Effect.NavigateBack))
    collectJob.cancel()
}
```

Use `UnconfinedTestDispatcher()` for the collection job — it starts collecting immediately before any events are fired.

---

## Turbine

Turbine simplifies Flow testing with a structured collection API.

```kotlin
// build.gradle.kts (test)
testImplementation(libs.turbine)
```

### Testing StateFlow with Turbine

```kotlin
@Test
fun `loading completes after groups refresh`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()

    viewModel.state.test {
        val initial = awaitItem()
        assertTrue(initial.isLoading)

        advanceUntilIdle()

        val loaded = awaitItem()
        assertFalse(loaded.isLoading)
        assertEquals(3, loaded.groups.size)

        cancelAndIgnoreRemainingEvents()
    }
}
```

### Testing Effects with Turbine

```kotlin
@Test
fun `OnSubmitClick success emits NavigateBack`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()
    fillValidForm(viewModel)

    viewModel.effect.test {
        viewModel.event(GroupContract.Event.OnSubmitClick)
        advanceUntilIdle()

        assertEquals(GroupContract.Effect.NavigateBack, awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Testing a Repository Flow

```kotlin
@Test
fun `observeGroups emits updated list after upsert`() = runTest {
    val dao = FakeGroupDao()
    val repository = GroupRepositoryImpl(remoteDataSource = FakeGroupRemoteDataSource(), localDataSource = GroupLocalDataSourceImpl(dao))

    repository.observeGroups().test {
        assertEquals(emptyList(), (awaitItem() as Either.Success).value)

        dao.upsertGroups(listOf(groupEntity("1"), groupEntity("2")))

        val updated = (awaitItem() as Either.Success).value
        assertEquals(2, updated.size)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## Testing Delays and Timeouts

Use `advanceTimeBy` to simulate time without real delay:

```kotlin
@Test
fun `search debounce waits 300ms before querying`() = runTest(mainDispatcherRule.dispatcher) {
    val viewModel = buildViewModel()

    viewModel.event(SearchContract.Event.OnQueryChanged("al"))
    advanceTimeBy(200)
    assertEquals(0, fakeRepository.searchCallCount)

    advanceTimeBy(150)
    advanceUntilIdle()
    assertEquals(1, fakeRepository.searchCallCount)
}
```

---

## Testing Cold vs Hot Flows

| Flow type | Testing approach |
|-----------|-----------------|
| `StateFlow` | Read `.value` after `advanceUntilIdle()` |
| `SharedFlow` (effects) | Collect in background job or use Turbine `.test {}` |
| Cold `Flow` (repository) | Use Turbine `.test {}` to step through emissions |

---

## Rules

Always:

- Use `StandardTestDispatcher` for ViewModel tests via `MainDispatcherRule`
- Use `UnconfinedTestDispatcher()` for the collection `launch` job in effect tests
- Call `cancelAndIgnoreRemainingEvents()` at end of Turbine blocks
- Call `advanceUntilIdle()` after triggering any coroutine-launching event

Never:

- Use `Thread.sleep` — use `advanceTimeBy` or `advanceUntilIdle`
- Collect from `SharedFlow` without a background job — it suspends indefinitely
- Use `withTimeout` in tests — Turbine's `awaitItem()` has a built-in timeout
