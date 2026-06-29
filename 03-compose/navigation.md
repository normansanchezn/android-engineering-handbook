# Navigation

## Setup

Navigation uses `NavHost` with a single `NavController` at the top level.

```kotlin
// presentation/navigation/AppNavHost.kt
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController(),
) {
    NavHost(
        navController = navController,
        startDestination = HomeRouter.ROUTE,
    ) {
        homeGraph(navController)
        groupGraph(navController)
        settingsGraph(navController)
    }
}
```

Applied at root in `MainActivity`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                AppNavHost()
            }
        }
    }
}
```

---

## Router

Each feature defines its own `Router` object. No string literals scattered through the codebase.

```kotlin
// presentation/feature/groups/GroupRouter.kt
object GroupRouter {
    const val ROUTE = "groups"
}

// With arguments
object GroupDetailRouter {
    const val ARG_GROUP_ID = "groupId"
    const val ROUTE = "group/{$ARG_GROUP_ID}"

    fun buildRoute(groupId: String) = "group/$groupId"
}
```

---

## NavGraph Extension

Each feature registers itself as a NavGraph extension function.

```kotlin
// presentation/feature/groups/GroupNavGraph.kt
fun NavGraphBuilder.groupGraph(navController: NavController) {
    composable(route = GroupRouter.ROUTE) {
        GroupScreen(navController = navController)
    }
    composable(
        route = GroupDetailRouter.ROUTE,
        arguments = listOf(
            navArgument(GroupDetailRouter.ARG_GROUP_ID) { type = NavType.StringType }
        ),
    ) { backStackEntry ->
        val groupId = backStackEntry.arguments?.getString(GroupDetailRouter.ARG_GROUP_ID).orEmpty()
        GroupDetailScreen(groupId = groupId, navController = navController)
    }
}
```

---

## Navigating

Pass `navController` into screen composables or pass navigation lambdas (preferred for testability).

**Option A: pass lambdas (preferred)**

```kotlin
composable(GroupRouter.ROUTE) {
    GroupScreen(
        onNavigateToDetail = { groupId ->
            navController.navigate(GroupDetailRouter.buildRoute(groupId))
        },
        onNavigateBack = { navController.navigateUp() },
    )
}
```

**Option B: pass navController directly (simpler for small apps)**

```kotlin
composable(GroupRouter.ROUTE) {
    GroupScreen(navController = navController)
}
```

Navigation is triggered by an `Effect`:

```kotlin
// In the screen's LaunchedEffect
is GroupContract.Effect.NavigateToDetail -> {
    navController.navigate(GroupDetailRouter.buildRoute(effect.groupId))
}
is GroupContract.Effect.NavigateBack -> navController.navigateUp()
```

---

## Arguments

Pass simple scalars only: `String`, `Int`, `Long`, `Boolean`.

```kotlin
// Passing an ID
navController.navigate(GroupDetailRouter.buildRoute(groupId = group.id))

// Reading it in the screen
@Composable
fun GroupDetailScreen(
    groupId: String,     // received from NavBackStackEntry by NavGraph
    navController: NavController,
    viewModel: GroupDetailViewModel = hiltViewModel(),
)
```

Pass the argument to the ViewModel via `SavedStateHandle`:

```kotlin
@HiltViewModel
class GroupDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val getGroupUseCase: GetGroupUseCase,
) : ViewModel(), GroupDetailContract {

    private val groupId: String = checkNotNull(savedStateHandle[GroupDetailRouter.ARG_GROUP_ID])
    ...
}
```

---

## Back Stack

| Goal | How |
|------|-----|
| Go back one screen | `navController.navigateUp()` |
| Go back and clear the current screen | `navController.popBackStack()` |
| Go back to a specific destination | `navController.popBackStack(route, inclusive = false)` |
| Navigate and clear up to root | `navigate(...) { popUpTo(startDestination) { inclusive = true } }` |

---

## Rules

Always:

- Define routes in `object XxxRouter { const val ROUTE = "..." }`
- Build route strings with a function (`buildRoute(id)`) — never concatenate at the call site
- Trigger navigation from an `Effect`, not directly from a UI event lambda
- Read route arguments from `SavedStateHandle` in the ViewModel

Never:

- Use string literals for routes outside the Router object
- Navigate from inside a ViewModel — emit an Effect and navigate in the screen
- Pass complex objects as nav arguments — pass an ID and load in the destination ViewModel
