# Serialization

## Strategy

Use Kotlin Serialization (`kotlinx.serialization`) as the default serialization library.

It is Kotlin-first, null-safe, and integrates with Retrofit via `retrofit2-kotlinx-serialization-converter`.

---

## Setup

```kotlin
// build.gradle.kts (data module)
plugins {
    kotlin("plugin.serialization")
}

dependencies {
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.retrofit.kotlinx.serialization)
}
```

```kotlin
// Retrofit client
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(
        Json { ignoreUnknownKeys = true }.asConverterFactory("application/json".toMediaType())
    )
    .client(okHttpClient)
    .build()
```

`ignoreUnknownKeys = true` is required — APIs add fields over time and unknown fields must not crash the app.

---

## DTO Annotation

```kotlin
@Serializable
data class GroupDto(
    @SerialName("id")          val id: String,
    @SerialName("name")        val name: String,
    @SerialName("label")       val label: String,
    @SerialName("teacher_id")  val teacherId: String,
    @SerialName("created_at")  val createdAt: String,
    @SerialName("is_active")   val isActive: Boolean = true,
)
```

Always use `@SerialName` explicitly on every field — do not rely on property name matching. API field names change independently of Kotlin naming conventions.

---

## Nullable Fields

Fields that may be absent in API responses must be nullable with a default of `null`:

```kotlin
@Serializable
data class UserDto(
    @SerialName("id")           val id: String,
    @SerialName("name")         val name: String,
    @SerialName("avatar_url")   val avatarUrl: String? = null,
    @SerialName("bio")          val bio: String? = null,
)
```

Never make a field non-null when the API can omit it — this causes a `SerializationException` at runtime.

---

## Nested Objects

```kotlin
@Serializable
data class CourseDto(
    @SerialName("id")       val id: String,
    @SerialName("name")     val name: String,
    @SerialName("teacher")  val teacher: TeacherDto,
    @SerialName("groups")   val groups: List<GroupDto> = emptyList(),
)

@Serializable
data class TeacherDto(
    @SerialName("id")   val id: String,
    @SerialName("name") val name: String,
)
```

---

## Date and Time

Receive dates as `String` in DTOs and parse to `kotlinx.datetime` or `java.time` types at the mapping boundary:

```kotlin
@Serializable
data class SessionDto(
    @SerialName("id")         val id: String,
    @SerialName("start_time") val startTime: String,   // "2024-09-01T08:00:00Z"
    @SerialName("end_time")   val endTime: String,
)

// In toDomain()
fun SessionDto.toDomain() = SessionModel(
    id = id,
    startTime = Instant.parse(startTime),
    endTime = Instant.parse(endTime),
)
```

Never use `@Contextual` with date serializers on DTOs — it adds complexity for no benefit when string parsing at the boundary is simpler.

---

## Enums

```kotlin
@Serializable
enum class GroupStatusDto {
    @SerialName("active")   ACTIVE,
    @SerialName("inactive") INACTIVE,
    @SerialName("archived") ARCHIVED,
}
```

Use `@SerialName` on every enum value. Map to a domain enum in `toDomain()`.

---

## Sealed Classes (Union Types)

```kotlin
@Serializable
sealed class NotificationDto {
    @Serializable @SerialName("message")
    data class MessageDto(val content: String) : NotificationDto()

    @Serializable @SerialName("alert")
    data class AlertDto(val title: String, val body: String) : NotificationDto()
}
```

Requires `classDiscriminator` configuration if the API uses a different type field name:

```kotlin
Json {
    ignoreUnknownKeys = true
    classDiscriminator = "type"
}
```

---

## Json Configuration

```kotlin
val json = Json {
    ignoreUnknownKeys = true    // required — APIs evolve
    isLenient = false           // strict parsing
    encodeDefaults = false      // don't send null fields in requests
    prettyPrint = false         // production
}
```

Share a single `Json` instance via DI. Do not create multiple `Json` instances.

---

## Rules

Always:

- Annotate every DTO with `@Serializable`
- Use `@SerialName` on every field and enum value
- Configure `ignoreUnknownKeys = true`
- Receive dates as `String`, parse in `toDomain()`
- Provide default values for optional fields (`null` or empty list)

Never:

- Use Gson or Moshi in new code — use Kotlin Serialization
- Rely on property name matching without `@SerialName`
- Parse dates inside a DTO — that belongs in the mapping function
- Expose serialization annotations outside the data layer
