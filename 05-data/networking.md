# Networking

## OkHttp Client

The OkHttp client is configured once in the DI module and shared across all Retrofit instances.

```kotlin
// data/di/NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(
        authInterceptor: AuthInterceptor,
        loggingInterceptor: HttpLoggingInterceptor,
    ): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(authInterceptor)
        .addInterceptor(loggingInterceptor)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.BASE_URL)
        .client(client)
        .addConverterFactory(
            Json { ignoreUnknownKeys = true }.asConverterFactory("application/json".toMediaType())
        )
        .build()

    @Provides
    @Singleton
    fun provideLoggingInterceptor(): HttpLoggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
    }
}
```

Logging is enabled only in debug builds. Never log request bodies in production — they may contain sensitive data.

---

## Auth Interceptor

Attaches the auth token to every request:

```kotlin
// data/network/interceptor/AuthInterceptor.kt
class AuthInterceptor @Inject constructor(
    private val sessionStore: SessionStore,
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = sessionStore.getToken()
        val request = if (token != null) {
            chain.request().newBuilder()
                .addHeader("Authorization", "Bearer $token")
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}
```

`SessionStore` is a lightweight synchronous store (SharedPreferences or DataStore with blocking read). Interceptors run on OkHttp's IO thread — no coroutines here.

---

## Token Refresh

Handle 401 responses with an `Authenticator`:

```kotlin
// data/network/authenticator/TokenAuthenticator.kt
class TokenAuthenticator @Inject constructor(
    private val authRepository: AuthRepository,
) : Authenticator {

    override fun authenticate(route: Route?, response: Response): Request? {
        if (responseCount(response) >= 2) return null  // stop after 2 retries

        val newToken = runBlocking { authRepository.refreshToken() }
            .fold(onSuccess = { it.token }, onError = { return null })

        return response.request.newBuilder()
            .header("Authorization", "Bearer $newToken")
            .build()
    }

    private fun responseCount(response: Response): Int {
        var count = 1
        var prior = response.priorResponse
        while (prior != null) { count++; prior = prior.priorResponse }
        return count
    }
}
```

Register the authenticator in the OkHttpClient:

```kotlin
OkHttpClient.Builder()
    .authenticator(tokenAuthenticator)
    ...
```

---

## Multiple Base URLs

Use separate Retrofit instances for different base URLs:

```kotlin
@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class AuthApi

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class ContentApi

@Provides @Singleton @AuthApi
fun provideAuthRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
    .baseUrl(BuildConfig.AUTH_BASE_URL)
    .client(client)
    .addConverterFactory(jsonConverterFactory)
    .build()

@Provides @Singleton @ContentApi
fun provideContentRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
    .baseUrl(BuildConfig.CONTENT_BASE_URL)
    .client(client)
    .addConverterFactory(jsonConverterFactory)
    .build()
```

---

## Environment Configuration

Base URLs and API keys belong in `BuildConfig`, sourced from `local.properties` or CI environment variables — never hardcoded:

```kotlin
// build.gradle.kts
android {
    buildTypes {
        debug {
            buildConfigField("String", "BASE_URL", "\"${properties["DEBUG_BASE_URL"]}\"")
        }
        release {
            buildConfigField("String", "BASE_URL", "\"${properties["RELEASE_BASE_URL"]}\"")
        }
    }
}
```

---

## Certificate Pinning

For high-security endpoints, add certificate pinning to the OkHttpClient:

```kotlin
val certificatePinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build()

OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    ...
```

Pin the certificate's public key hash, not the certificate itself — allows certificate rotation without an app update.

---

## Rules

Always:

- Configure OkHttp once as a `@Singleton` in a DI module
- Gate logging interceptor behind `BuildConfig.DEBUG`
- Attach auth tokens via interceptor, not inside service methods
- Use `Authenticator` for token refresh — not manual 401 checks in repositories
- Source base URLs from `BuildConfig` — never hardcode

Never:

- Create a new `OkHttpClient` per service
- Log request bodies in production
- Perform blocking operations in interceptors beyond synchronous token reads
- Add auth headers manually in Retrofit service definitions
