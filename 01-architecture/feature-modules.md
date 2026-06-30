# Feature Modules

## Purpose

Feature modules isolate features into independent Gradle modules. Each feature can be compiled, tested, and evolved in isolation.

Large projects grow by adding feature modules — not by increasing complexity inside existing modules.

---

## Module Pattern

Each feature has two modules:

| Module | Responsibility |
|--------|---------------|
| `:feature:groups:api` | Public contract: navigation routes, entry point interfaces |
| `:feature:groups:impl` | Implementation: ViewModel, screens, DI, data wiring |

`:app` depends on `:feature:groups:impl`.
Other features depend only on `:feature:groups:api`.

---

## Directory Structure

```
feature/
└── groups/
    ├── api/
    │   └── src/main/kotlin/feature/groups/api/
    │       └── GroupsNavigation.kt
    └── impl/
        └── src/main/kotlin/feature/groups/
            ├── di/
            │   └── GroupsModule.kt
            ├── ui/
            │   └── GroupsScreen.kt
            └── viewmodel/
                ├── GroupsContract.kt
                └── GroupsViewModel.kt
```

---

## Navigation Contract (api module)

```kotlin
// feature/groups/api/GroupsNavigation.kt
object GroupsNavigation {
    const val ROUTE = "groups"
    const val ROUTE_DETAIL = "groups/{groupId}"

    fun detailRoute(groupId: String) = "groups/$groupId"
}
```

The api module has no Android, Hilt, or Compose dependencies. It is a pure Kotlin module.

---

## Registering a Feature (app module)

```kotlin
// app/navigation/AppNavHost.kt
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController = navController, startDestination = HomeNavigation.ROUTE) {
        groupsNavGraph(navController)
        homeNavGraph(navController)
    }
}

// feature/groups/impl/navigation/GroupsNavGraph.kt
fun NavGraphBuilder.groupsNavGraph(navController: NavHostController) {
    composable(GroupsNavigation.ROUTE) {
        GroupsScreen(
            onNavigateBack = { navController.popBackStack() },
            onNavigateToDetail = { id -> navController.navigate(GroupsNavigation.detailRoute(id)) },
        )
    }
    composable(
        route = GroupsNavigation.ROUTE_DETAIL,
        arguments = listOf(navArgument("groupId") { type = NavType.StringType }),
    ) { backStackEntry ->
        val groupId = backStackEntry.arguments?.getString("groupId") ?: return@composable
        GroupDetailScreen(groupId = groupId, onNavigateBack = { navController.popBackStack() })
    }
}
```

---

## Gradle Configuration

```kotlin
// feature/groups/api/build.gradle.kts
plugins {
    id("java-library")
    id("org.jetbrains.kotlin.jvm")
}

// feature/groups/impl/build.gradle.kts
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    implementation(project(":feature:groups:api"))
    implementation(project(":domain"))
    implementation(project(":design-system"))
    implementation(project(":core"))
}

// app/build.gradle.kts
dependencies {
    implementation(project(":feature:groups:impl"))
    implementation(project(":feature:home:impl"))
}
```

---

## Cross-Feature Navigation

Features never depend on each other's impl modules. Navigation between features goes through the app's NavHost.

```kotlin
// ✅ Feature A triggers navigation via an Effect
is HomeContract.Effect.OpenGroups -> navController.navigate(GroupsNavigation.ROUTE)

// ❌ Feature A imports Feature B impl directly
import com.example.feature.groups.impl.GroupsScreen  // never
```

---

## Shared Code Placement

| Type | Module |
|------|--------|
| Domain models | `:domain` |
| Repository interfaces | `:domain` |
| UI components | `:design-system` |
| Core utilities | `:core` |
| Navigation routes | `:feature:xxx:api` |

---

## Rules

Always:

- Keep `:api` modules thin — routes and entry point interfaces only
- `:impl` depends on `:api` of the same feature
- `:app` wires all navigation and registers all feature nav graphs
- Features communicate through the domain layer, not directly

Never:

- Depend on another feature's `:impl`
- Put navigation routes inside `:impl`
- Create circular dependencies between feature modules
- Share mutable state between feature modules directly
