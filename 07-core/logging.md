# Logging

## Approach

Use a thin logging wrapper around `Timber` (or the logging library of your choice).

The wrapper lets you:
- Strip logs from release builds in one place
- Add metadata (tags, user context) consistently
- Swap the underlying library without touching callers

---

## Setup

```kotlin
// core/logging/AppLogger.kt
object AppLogger {

    fun init(isDebug: Boolean) {
        if (isDebug) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashReportingTree())
        }
    }

    fun d(message: String, vararg args: Any?) = Timber.d(message, *args)
    fun i(message: String, vararg args: Any?) = Timber.i(message, *args)
    fun w(message: String, vararg args: Any?) = Timber.w(message, *args)
    fun e(throwable: Throwable? = null, message: String, vararg args: Any?) =
        Timber.e(throwable, message, *args)
}

private class CrashReportingTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.WARN) return
        // Forward WARN+ to crash reporter (e.g. Firebase Crashlytics)
        // FirebaseCrashlytics.getInstance().log("$tag: $message")
        // t?.let { FirebaseCrashlytics.getInstance().recordException(it) }
    }
}
```

Initialize in `Application`:

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        AppLogger.init(isDebug = BuildConfig.DEBUG)
    }
}
```

---

## What to Log

| Level | When | Example |
|-------|------|---------|
| `d` | Debug flow, state transitions | `"OnLoginClick received"` |
| `i` | Notable events (user action, screen shown) | `"User logged in: ${userId}"` |
| `w` | Unexpected but recoverable | `"Group not found in cache, fetching..."` |
| `e` | Errors and exceptions | `"Login failed: ${failure}"` |

---

## Logging in a ViewModel

```kotlin
override fun event(event: LoginContract.Event) {
    AppLogger.d("LoginViewModel event: $event")
    when (event) {
        is LoginContract.Event.OnLoginClick -> login()
        ...
    }
}

private fun login() {
    viewModelScope.launch {
        loginUseCase(email, password).fold(
            onSuccess = { AppLogger.i("Login success") },
            onError = { failure -> AppLogger.e(message = "Login failed: $failure") },
        )
    }
}
```

---

## What NOT to Log

- Passwords, tokens, PII (personally identifiable information)
- Full HTTP responses (use OkHttp's `HttpLoggingInterceptor` for debug only)
- Verbose loop data in release builds

---

## Rules

Always:

- Init `AppLogger` in `Application.onCreate()`
- Use `AppLogger` instead of `Log` or `println` directly
- Log errors with `AppLogger.e(throwable, message)` — include the exception

Never:

- Use `println()` or `Log.d()` directly in production code
- Log sensitive data (passwords, auth tokens, personal details)
- Leave `HttpLoggingInterceptor.Level.BODY` enabled in release builds
