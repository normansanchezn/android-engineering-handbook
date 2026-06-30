# Retrofit Service Template

## Standard Service

```kotlin
// data/remote/service/GroupApiService.kt
interface GroupApiService {

    @GET("groups")
    suspend fun getGroups(): List<GroupDto>

    @GET("groups/{id}")
    suspend fun getGroupById(@Path("id") id: String): GroupDto

    @GET("groups")
    suspend fun searchGroups(@Query("q") query: String): List<GroupDto>

    @POST("groups")
    suspend fun createGroup(@Body request: CreateGroupRequest): GroupDto

    @PUT("groups/{id}")
    suspend fun updateGroup(
        @Path("id") id: String,
        @Body request: UpdateGroupRequest,
    ): GroupDto

    @DELETE("groups/{id}")
    suspend fun deleteGroup(@Path("id") id: String)
}
```

---

## DI Binding

```kotlin
// data/di/NetworkModule.kt
@Provides
@Singleton
fun provideGroupApiService(retrofit: Retrofit): GroupApiService =
    retrofit.create(GroupApiService::class.java)
```

---

## Response<T> for explicit HTTP handling

Use `Response<T>` when you need to inspect HTTP status codes inside the repository:

```kotlin
@GET("groups/{id}")
suspend fun getGroupById(@Path("id") id: String): Response<GroupDto>
```

```kotlin
// In repository
override suspend fun getGroupById(id: String): Either<Failure, GroupModel> =
    safeApiCall {
        val response = apiService.getGroupById(id)
        when {
            response.isSuccessful -> response.body()?.toDomain()
                ?: return@safeApiCall Either.Error(Failure.EmptyData("Group not found"))
            response.code() == 404 -> return@safeApiCall Either.Error(Failure.EmptyData("Group $id not found"))
            else -> return@safeApiCall Either.Error(Failure.ServerError(response.code(), response.message()))
        }
    }
```

For most cases, use raw return type — `safeApiCall` handles HTTP errors via `HttpException` automatically.

---

## Headers

```kotlin
@GET("groups")
suspend fun getGroups(@Header("Accept-Language") language: String): List<GroupDto>

// Static headers on all calls in a service
@Headers("Content-Type: application/json")
@POST("groups")
suspend fun createGroup(@Body request: CreateGroupRequest): GroupDto
```

Dynamic auth headers belong in an OkHttp `Interceptor`, not in service definitions.

---

## Multipart (File Upload)

```kotlin
@Multipart
@POST("groups/{id}/avatar")
suspend fun uploadAvatar(
    @Path("id") id: String,
    @Part avatar: MultipartBody.Part,
): GroupDto
```

```kotlin
// In repository
val file = File(filePath)
val requestBody = file.asRequestBody("image/*".toMediaType())
val part = MultipartBody.Part.createFormData("avatar", file.name, requestBody)
apiService.uploadAvatar(id, part)
```

---

## Pagination

```kotlin
@GET("groups")
suspend fun getGroups(
    @Query("page")     page: Int,
    @Query("per_page") perPage: Int = 20,
): PagedResponse<GroupDto>
```

---

## Rules

Always:

- Use `suspend` on every service function — no `Call<T>` wrappers
- Define one interface per domain area (GroupApiService, StudentApiService)
- Provide the service via DI as a `@Singleton`
- Use `@Path` for URL segments, `@Query` for query parameters, `@Body` for request bodies

Never:

- Add auth headers inside service methods — use an `Interceptor`
- Return `Call<T>` — use `suspend` functions
- Use `@Url` for dynamic base URLs — use a separate Retrofit instance per base URL
- Put business logic inside the service interface
