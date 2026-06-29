# Code Review

## What to Look For

Code review is about correctness, architecture consistency, and catching issues before they reach production. Not about style preferences.

---

## Architecture Checklist

- [ ] Does the ViewModel implement the Contract interface?
- [ ] Is `UiState` a `data class` with sensible defaults?
- [ ] Does `ScreenUiModel` hold all user-visible strings (no hardcoded strings in composables)?
- [ ] Are effects one-shot actions (navigate, snackbar) and NOT persistent state?
- [ ] Is the root screen the only stateful composable? Are children stateless?
- [ ] Does every async operation return `Either<Failure, T>`?
- [ ] Are business rules in UseCases (not in ViewModel event handlers)?
- [ ] Does the repository interface live in `domain/`, implementation in `data/`?

---

## MVI Specific

- [ ] Every `Event` has a handler in the ViewModel `event()` function
- [ ] `updateState` is used for all state mutations (no direct `_state.value = ...`)
- [ ] Form validation calls `recalculateFormValidity()` after every field update
- [ ] `DialogState` is a sealed class — not multiple boolean flags
- [ ] Effects are collected in `LaunchedEffect(Unit)` at the screen root

---

## Data Layer

- [ ] Repository calls go through `safeApiCall`
- [ ] DTOs are mapped to domain models at the repository boundary
- [ ] `toDomain()` / `toDto()` are extension functions on DTOs, not methods on domain models
- [ ] No domain models in `data/` package

---

## Compose

- [ ] No `hiltViewModel()` called in child composables
- [ ] No `remember { mutableStateOf(...) }` for state that belongs in the ViewModel
- [ ] `key = { item.id }` on every `LazyColumn`/`LazyRow`
- [ ] `collectAsStateWithLifecycle()` used (not `collectAsState()`)
- [ ] Every composable has `Modifier = Modifier` param

---

## Testing

- [ ] Every new public ViewModel event has a test
- [ ] Every failure path has a test
- [ ] Fake repositories used (not mocks) for ViewModel tests
- [ ] MockK used only for infrastructure dependencies (Context, API service)

---

## Common Rejections

| Finding | Fix |
|---------|-----|
| Hardcoded string in a composable | Move to `ScreenUiModel`, load in ViewModel constructor |
| `if/else` on multiple boolean flags for dialogs | Replace with `DialogState` sealed class |
| Business rule in a ViewModel event handler | Extract to a UseCase |
| `collectAsState()` instead of `collectAsStateWithLifecycle()` | Replace |
| Missing `key` in `LazyColumn` | Add `key = { item.id }` |
| Catching exceptions in repository instead of `safeApiCall` | Use `safeApiCall` |
| Navigation in ViewModel | Emit an Effect, navigate in screen's `LaunchedEffect` |
