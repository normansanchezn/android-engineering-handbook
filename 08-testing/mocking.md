# Mocking

## When to Mock vs Fake

| Dependency type | Preferred approach |
|-----------------|--------------------|
| Repository (domain interface) | **Fake** — in-memory implementation |
| Simple value provider (`Context`) | **MockK** with `relaxed = true` |
| Complex SDK (auth, analytics) | **Fake gateway** or **MockK** |
| Use case | **Real class** with a fake repository |

Repositories should always be faked, not mocked. See [fake-repositories.md](fake-repositories.md).

---

## MockK Usage

```kotlin
// Relaxed mock — all methods return default values automatically
val context: Context = mockk(relaxed = true)

// Stub a specific return value
every { context.getString(R.string.group_name) } returns "Group Name"
every { context.getString(any()) } returns ""  // catch-all stub
```

---

## Context Mocking for ViewModel Strings

ViewModels load strings via `context.getString()`. Mock Context with a relaxed mock so all string calls return `""` by default, then stub only what you need to verify.

```kotlin
private fun buildViewModel(): GroupViewModel {
    val context: Context = mockk(relaxed = true)
    // If testing specific string content:
    every { context.getString(R.string.group_name_required_error) } returns "Name is required"
    
    return GroupViewModel(
        context = context,
        saveGroupUseCase = SaveGroupUseCase(repository),
    )
}
```

For tests that don't care about string content (state transitions, effect emissions), `mockk(relaxed = true)` with no stubs is sufficient.

---

## Verifying Interactions (when needed)

```kotlin
val analytics: AnalyticsTracker = mockk(relaxed = true)

viewModel.event(GroupContract.Event.OnSubmitClick)
advanceUntilIdle()

verify { analytics.track("group_saved") }
```

Prefer asserting on state and effects over verifying mock calls. State assertions are more resilient to implementation changes.

---

## Rules

Always:

- `relaxed = true` for Context mocks — avoids boilerplate stubs for every string
- Use real use case instances with fake repositories, not mocked use cases
- Reset mocks if reused across tests (or create new ones in `beforeTest()`)

Never:

- Mock repository interfaces in ViewModel tests — use Fake implementations instead
- Mock data classes or value objects
- Verify mock calls when you can assert on state instead
