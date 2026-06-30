# Dependency Review

## Version Catalog

All dependencies are declared in `gradle/libs.versions.toml`. No dependency is added directly to a module's `build.gradle.kts` without a corresponding catalog entry.

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.1.0"
compose-bom = "2025.05.00"
hilt = "2.52"
retrofit = "2.11.0"
room = "2.7.1"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
room = ["room-runtime", "room-ktx"]

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

---

## Evaluating a New Dependency

Before adding a dependency, answer these questions:

| Question | Acceptable |
|----------|-----------|
| Is it actively maintained? | Last release within 12 months |
| Does it have tests? | Yes |
| What is its license? | Apache 2.0, MIT, BSD — never GPL |
| How large is it (method count, binary size)? | Justify if > 5k methods |
| Does the existing project already solve this? | No — check first |
| Is there a lighter alternative? | Prefer fewer, larger, well-maintained libs over many small ones |

---

## Checking Transitive Dependencies

Before adding a library, inspect what it pulls in:

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath | grep "my-new-lib"
```

Watch for:
- Duplicate transitive dependencies with conflicting versions
- Unexpected Google Play Services dependencies in a non-GMS build
- Libraries that pull in `okhttp` or `gson` — check version compatibility with existing network stack

---

## BOMs

Prefer BOMs for library families that release together:

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)               // version managed by BOM
    implementation(libs.compose.material3)
}
```

BOM ensures version consistency across a family. Do not override BOM-managed versions without a documented reason.

---

## Security

Check for known vulnerabilities before adding a dependency:

```bash
./gradlew dependencyCheckAnalyze
```

Uses the OWASP Dependency Check plugin. Integrate in CI for release builds:

```kotlin
// build.gradle.kts (root)
plugins {
    id("org.owasp.dependencycheck") version "..."
}

dependencyCheck {
    failBuildOnCVSS = 7.0f  // fail on High and Critical CVEs
}
```

---

## Removing Dependencies

When removing a library:
1. Remove from `libs.versions.toml`
2. Remove from all module `build.gradle.kts` files
3. Run `./gradlew build` to confirm no usages remain
4. Check if any transitive dependencies it was providing are needed by other libs

---

## Rules

Always:

- Add new dependencies to `libs.versions.toml` before referencing them in modules
- Verify license compatibility before merging
- Use BOMs for library families (Compose, Firebase, etc.)
- Check transitive conflicts with `./gradlew dependencies`

Never:

- Add a dependency to solve a problem the existing stack already handles
- Use GPL-licensed libraries
- Add a dependency without checking if an existing utility in `core` already solves it
- Pin to `-SNAPSHOT` versions in production code
