# Feature Completion Checklist

Run this before marking any feature as done.

---

## Architecture

- [ ] Domain model is pure Kotlin — no framework annotations
- [ ] Repository interface is in `domain/` — returns `Either<Failure, T>`
- [ ] Repository implementation is in `data/` — uses `safeApiCall`
- [ ] DI module binds interface → implementation via `@Binds`
- [ ] Use case has a single responsibility and `suspend operator fun invoke()`
- [ ] DTOs do not appear in domain or presentation layers
- [ ] Domain models do not appear in the data layer (only via the repository interface)
- [ ] No business logic in ViewModel or Screen

---

## MVI Contract

- [ ] Contract interface extends `MVIContract`
- [ ] `UiState` is a `data class` — all fields have default values
- [ ] `ScreenUiModel` holds every string the screen displays
- [ ] `DialogState` sealed class: `None`, `Loading`, `Error`, `Success`
- [ ] `Effect` sealed interface for all navigation and one-time actions
- [ ] `Event` sealed interface for all user interactions

---

## ViewModel

- [ ] `@HiltViewModel` annotation
- [ ] Implements the feature Contract
- [ ] `@ApplicationContext` used for string loading
- [ ] All strings loaded in `createUiStrings()` — called once in the initial state
- [ ] Private `updateState { }` wraps `MutableStateFlow.update`
- [ ] Private `sendEffect { }` wraps `viewModelScope.launch { uiEffect.emit() }`
- [ ] All `Failure` types handled in `mapFailureToMessage()`
- [ ] Form validity recalculated after every field update via `recalculateFormValidity()`
- [ ] Field validation uses `validateRequiredTextField()` on every keystroke

---

## Screen

- [ ] Root composable injects ViewModel via `hiltViewModel()`
- [ ] `collectAsStateWithLifecycle()` — not `collectAsState()`
- [ ] `LaunchedEffect(Unit) { effect.collect { } }` for effects
- [ ] Child composables are `private` and stateless
- [ ] No ViewModel reference in child composables — lambdas only
- [ ] Dialog rendered as a separate overlay composable
- [ ] `BackHandler` added where needed

---

## Navigation

- [ ] `XxxRouter` object with `const val ROUTE`
- [ ] Route registered in `NavHost`
- [ ] Navigation callbacks passed as lambdas to the screen

---

## Resources

- [ ] All strings in `res/values/strings.xml`
- [ ] No hardcoded strings in Kotlin code or Composables
- [ ] String keys follow `feature_element_description` pattern

---

## Quality

- [ ] No `TODO` or `FIXME` comments left
- [ ] No unused imports
- [ ] Preview exists for the screen body composable
- [ ] Preview exists for any new reusable component
