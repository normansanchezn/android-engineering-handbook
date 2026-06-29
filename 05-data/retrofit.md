# Retrofit

## Setup

```kotlin
// di/NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                        else HttpLoggingInterceptor.Level.NONE
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    @Singleton
    fun provideGroupApiService(retrofit: Retrofit): GroupApiService =
        retrofit.create(GroupApiService::class.java)
}
```

---

## Auth Interceptor

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider,
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getToken()
        val request = chain.request().newBuilder()
            .apply { if (token != null) addHeader("Authorization", "Bearer $token") }
            .build()
        return chain.proceed(request)
    }
}
```

---

## API Service Definition

```kotlin
interface GroupApiService {
    @GET("groups")
    suspend fun getGroups(): List<GroupDto>

    @GET("groups/{id}")
    suspend fun getGroupById(@Path("id") id: String): GroupDto

    @POST("groups")
    suspend fun createGroup(@Body request: CreateGroupRequest): GroupDto

    @PUT("groups/{id}")
    suspend fun updateGroup(@Path("id") id: String, @Body request: CreateGroupRequest): GroupDto

    @DELETE("groups/{id}")
    suspend fun deleteGroup(@Path("id") id: String)
}
```

---

## Error Handling via safeApiCall

All network calls go through `safeApiCall` — it converts exceptions to `Either.Error(Failure.xxx)`.

See [07-core/result.md](../07-core/result.md) for the full implementation.

```kotlin
override suspend fun getGroups(): Either<Failure, List<GroupModel>> =
    safeApiCall { apiService.getGroups().map { it.toDomain() } }
```

`safeApiCall` maps:
- `IOException` → `Failure.NetworkError`
- `HttpException(401)` → `Failure.Unauthorized`
- `HttpException(5xx)` → `Failure.ServerError`
- Other `Throwable` → `Failure.UnknownError`

---

## Rules

Always:

- Define one `@Provides` per API service in the DI module
- Use `suspend` functions in API services — Retrofit handles coroutine dispatch
- Wrap every call in `safeApiCall`

Never:

- Catch exceptions in the repository — `safeApiCall` handles it
- Put `BuildConfig.BASE_URL` inline in a composable or ViewModel
- Share a single `ApiService` interface across multiple domains — split by feature
