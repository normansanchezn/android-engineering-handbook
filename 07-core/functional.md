# Functional Helpers

## Purpose

Extensions on `Either` enable composing operations without nested `when` blocks or manual unwrapping.

These helpers live in `core/functional/` alongside `Either.kt`.

---

## map (transform success value)

```kotlin
inline fun <L, R, T> Either<L, R>.map(transform: (R) -> T): Either<L, T> = when (this) {
    is Either.Error   -> this
    is Either.Success -> Either.Success(transform(value))
}
```

Usage:

```kotlin
override suspend fun getGroups(): Either<Failure, List<GroupModel>> =
    safeApiCall { api.getGroups() }
        .map { dtos -> dtos.map { it.toDomain() } }
```

---

## flatMap (chain Either-returning operations)

```kotlin
inline fun <L, R, T> Either<L, R>.flatMap(transform: (R) -> Either<L, T>): Either<L, T> = when (this) {
    is Either.Error   -> this
    is Either.Success -> transform(value)
}
```

Usage — chain two operations where the second depends on the first:

```kotlin
suspend operator fun invoke(): Either<Failure, CourseModel> =
    getSessionUseCase()
        .flatMap { session -> getCourseUseCase(session.courseId) }
```

Without `flatMap` this would require a nested `when` or `fold`.

---

## mapError (transform error value)

```kotlin
inline fun <L, R, T> Either<L, R>.mapError(transform: (L) -> T): Either<T, R> = when (this) {
    is Either.Error   -> Either.Error(transform(value))
    is Either.Success -> this
}
```

Usage — convert domain `Failure` to a UI-specific error type at the boundary:

```kotlin
result.mapError { failure -> mapFailureToMessage(failure, uiModel) }
```

---

## onSuccess / onError (side effects without transforming)

```kotlin
inline fun <L, R> Either<L, R>.onSuccess(action: (R) -> Unit): Either<L, R> {
    if (this is Either.Success) action(value)
    return this
}

inline fun <L, R> Either<L, R>.onError(action: (L) -> Unit): Either<L, R> {
    if (this is Either.Error) action(value)
    return this
}
```

Usage:

```kotlin
safeApiCall { api.deleteGroup(id) }
    .onSuccess { analytics.trackGroupDeleted() }
    .onError  { failure -> logger.e("Delete failed: $failure") }
```

---

## Parallel Use Cases

Run multiple independent use cases in parallel using `coroutineScope` with `async`:

```kotlin
class LoadDashboardUseCase @Inject constructor(
    private val getGroupsUseCase: GetGroupsUseCase,
    private val getStudentsUseCase: GetStudentsUseCase,
    private val getCourseUseCase: GetCourseUseCase,
) {
    suspend operator fun invoke(): Either<Failure, DashboardModel> = coroutineScope {
        val groupsDeferred   = async { getGroupsUseCase() }
        val studentsDeferred = async { getStudentsUseCase() }
        val courseDeferred   = async { getCourseUseCase() }

        val groups   = groupsDeferred.await()
        val students = studentsDeferred.await()
        val course   = courseDeferred.await()

        when {
            groups   is Either.Error -> groups
            students is Either.Error -> students
            course   is Either.Error -> course
            else -> Either.Success(
                DashboardModel(
                    groups   = (groups as Either.Success).value,
                    students = (students as Either.Success).value,
                    course   = (course as Either.Success).value,
                )
            )
        }
    }
}
```

`coroutineScope` cancels all children if any `async` block throws. All three calls run concurrently.

---

## zip (combine two Either results)

```kotlin
inline fun <L, A, B, C> Either<L, A>.zip(
    other: Either<L, B>,
    transform: (A, B) -> C,
): Either<L, C> = flatMap { a -> other.map { b -> transform(a, b) } }
```

Usage:

```kotlin
getGroupUseCase(groupId).zip(getStudentsUseCase(groupId)) { group, students ->
    GroupDetailModel(group = group, students = students)
}
```

---

## Rules

Always:

- Use `map` instead of `fold` when only transforming the success value
- Use `flatMap` to chain sequential Either-returning operations
- Use `async` + `coroutineScope` for parallel use case execution

Never:

- Unwrap `Either` with `(this as Either.Success).value` — use `fold` or `map`
- Chain with nested `fold` blocks — use `flatMap`
- Run parallel operations with sequential `suspend` calls
