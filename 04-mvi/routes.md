# Routes

## Purpose

A Router is a Kotlin `object` that holds the navigation route string for a screen.

It decouples navigation registration from screen implementation, and makes routes refactorable without touching the NavHost string directly.

---

## Definition

```kotlin
object GroupRouter {
    const val ROUTE = "Groups"
}

object SignInRouter {
    const val ROUTE = "Sign In"
}

object StudentDetailRouter {
    const val ARG_ID = "studentId"
    const val ROUTE = "StudentDetail/{$ARG_ID}"

    fun route(id: String) = "StudentDetail/$id"
}
```

File location: `presentation/feature/xxx/XxxRouter.kt`

---

## NavHost Registration

```kotlin
NavHost(navController = navController, startDestination = HomeRouter.ROUTE) {

    composable(route = SignInRouter.ROUTE) {
        SignInScreen(
            onNavigateBack = { navController.popBackStack() },
            onSignInComplete = { navController.navigate(HomeRouter.ROUTE) }
        )
    }

    composable(route = GroupRouter.ROUTE) {
        GroupScreen(
            onNavigateBack = { navController.popBackStack() }
        )
    }

    composable(
        route = StudentDetailRouter.ROUTE,
        arguments = listOf(navArgument(StudentDetailRouter.ARG_ID) { type = NavType.StringType })
    ) { backStackEntry ->
        val studentId = backStackEntry.arguments?.getString(StudentDetailRouter.ARG_ID).orEmpty()
        StudentDetailScreen(
            studentId = studentId,
            onNavigateBack = { navController.popBackStack() }
        )
    }
}
```

Navigation from a ViewModel effect:

```kotlin
// In NavHost:
is HomeContract.Effect.NavigateToDetail -> {
    navController.navigate(StudentDetailRouter.route(effect.studentId))
}
```

---

## Rules

Always:

- One Router object per feature screen
- Route strings are human-readable (useful for debugging the backstack)
- Use `const val ROUTE` for routes without arguments
- Use `fun route(arg)` for routes with arguments — keeps string construction in one place

Never:

- Navigate using string literals in the NavHost — always use the Router constant
- Put navigation logic in the ViewModel — it emits an Effect, the NavHost reacts
- Put navigation logic inside Composables — pass navigation callbacks as lambdas
