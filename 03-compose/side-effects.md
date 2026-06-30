# Side Effects

## Overview

Side effects interact with the outside world from inside a composable. Compose provides a specific API for each scenario. Using the wrong one causes bugs — effects firing too early, too often, or never cleaning up.

---

## LaunchedEffect

Runs a suspend block when the composable enters composition. Cancels and restarts when the key changes.

```kotlin
// Run once — collect effects for the full screen lifetime
LaunchedEffect(Unit) {
    viewModel.effect.collect { effect ->
        when (effect) {
            is Contract.Effect.NavigateBack -> navController.navigateUp()
            is Contract.Effect.ShowSnackbar -> snackbarHost.showSnackbar(effect.message)
        }
    }
}

// Restart when id changes — reload data for a new item
LaunchedEffect(groupId) {
    viewModel.loadGroup(groupId)
}
```

Use `LaunchedEffect(Unit)` for effects that must run for the composable's entire lifetime.
Use `LaunchedEffect(key)` when the suspend block must restart on key change.

---

## DisposableEffect

Runs setup code and provides a cleanup block via `onDispose`. Cleanup fires when the composable leaves composition or the key changes.

```kotlin
val lifecycleOwner = LocalLifecycleOwner.current

DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event ->
        when (event) {
            Lifecycle.Event.ON_RESUME -> viewModel.onResume()
            Lifecycle.Event.ON_PAUSE  -> viewModel.onPause()
            else -> Unit
        }
    }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}
```

Use for registering and unregistering listeners, callbacks, or observers that require cleanup.

Never use `DisposableEffect` without an `onDispose` block — that is a bug.

---

## SideEffect

Runs after every successful recomposition. Not cancelable, not a coroutine.

```kotlin
SideEffect {
    analyticsTracker.setCurrentScreen(uiState.screenName)
}
```

Use only to sync Compose-managed state to a non-Compose system (analytics, legacy View system, external SDK) after each recomposition.

Never make network calls or async operations inside `SideEffect` — it runs synchronously on every recomposition.

---

## produceState

Converts a non-Compose state source (callback API, external observable) into Compose `State<T>`.

```kotlin
val connectionState by produceState(initialValue = ConnectionState.Unknown) {
    val callback = object : ConnectivityManager.NetworkCallback() {
        override fun onAvailable(network: Network) { value = ConnectionState.Available }
        override fun onLost(network: Network) { value = ConnectionState.Lost }
    }
    connectivityManager.registerDefaultNetworkCallback(callback)
    awaitDispose { connectivityManager.unregisterNetworkCallback(callback) }
}
```

Use when bridging a callback-based or subscription-based API into composable state.

---

## derivedStateOf

Computes a value from other `State` objects. Recomputes only when its inputs change — not on every recomposition.

```kotlin
val isButtonEnabled by remember {
    derivedStateOf {
        uiState.name.isNotBlank() && uiState.label.isNotBlank()
    }
}
```

Only use `derivedStateOf` when the derived computation is expensive or when frequent recompositions from raw State reads are measured as a problem. Do not wrap all computations in it.

Always wrap `derivedStateOf` in `remember`.

---

## snapshotFlow

Converts Compose `State` reads into a `Flow`. Enables using Flow operators (filter, debounce, distinctUntilChanged) on state changes.

```kotlin
LaunchedEffect(Unit) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .filter { it > 0 }
        .distinctUntilChanged()
        .collect { viewModel.onScrolled() }
}
```

Use inside `LaunchedEffect` — `snapshotFlow` is a Flow, not a composable.

---

## Decision Table

| Need | API |
|------|-----|
| Run suspend code on entry or key change | `LaunchedEffect` |
| Register listener with cleanup | `DisposableEffect` |
| Sync to non-Compose system after recomposition | `SideEffect` |
| Bridge callback API to State | `produceState` |
| Compute derived state from other State | `derivedStateOf` |
| React to State changes with Flow operators | `snapshotFlow` |

---

## Rules

Always:

- Use `LaunchedEffect(Unit)` for collecting ViewModel effects at screen root
- Provide `onDispose { }` in every `DisposableEffect`
- Wrap `derivedStateOf` in `remember`
- Use `snapshotFlow` inside `LaunchedEffect`

Never:

- Make network calls inside `SideEffect`
- Use `DisposableEffect` without `onDispose`
- Use `LaunchedEffect` for work that requires teardown — use `DisposableEffect`
- Use `derivedStateOf` for cheap computations
