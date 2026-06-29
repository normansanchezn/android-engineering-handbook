# Lint

## Android Lint

Run Android Lint as part of the CI pipeline. It catches API usage errors, resource issues, and Compose-specific problems.

```bash
./gradlew lint
```

Key checks relevant to this stack:

| Check | What it catches |
|-------|----------------|
| `UnusedResources` | String/drawable resources defined but never used |
| `HardcodedText` | String literals in XML layouts (less relevant with Compose) |
| `InvalidVectorPath` | Malformed vector drawable paths |
| `ComposableLambdaParameterNaming` | Lambda params that don't follow Compose conventions |
| `ComposableNaming` | Composable functions that don't start with uppercase |
| `MutableCollectionMutableState` | `mutableListOf` inside `mutableStateOf` — stability problem |

---

## Baseline File

When adopting lint on an existing codebase, generate a baseline to suppress existing warnings and only fail on new ones:

```bash
./gradlew lint -PwriteLintBaseline
```

Commit `lint-baseline.xml`. CI fails only on new issues, not the existing backlog.

---

## Ktlint

`ktlint` enforces Kotlin code formatting automatically. Integrate via the `jlleitschuh/ktlint-gradle` plugin.

```kotlin
// build.gradle.kts (root)
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.2"
}

ktlint {
    android.set(true)
    outputToConsole.set(true)
    filter {
        exclude("**/generated/**")
    }
}
```

Check:

```bash
./gradlew ktlintCheck
```

Auto-fix:

```bash
./gradlew ktlintFormat
```

---

## Detekt (Optional)

`detekt` performs static analysis: complexity, code smells, naming conventions.

Useful checks for this stack:

| Rule set | What it catches |
|----------|----------------|
| `complexity` | Functions with too many params, deeply nested code |
| `naming` | Class/function names that don't match conventions |
| `style` | Redundant code, unused imports |

Configure in `detekt.yml` and suppress project-wide via baselines for existing code.

---

## CI Integration

Run lint, ktlint, and tests as a single check step:

```yaml
# .github/workflows/ci.yml
- name: Lint & Tests
  run: |
    ./gradlew ktlintCheck
    ./gradlew lint
    ./gradlew test
```

Fail the pipeline on any new lint warning — do not use `lintOptions { abortOnError false }` in production.

---

## Rules

Always:

- Fix lint warnings before merging — do not suppress without a comment explaining why
- Run `ktlintFormat` before committing
- Commit the lint baseline when first enabling lint on a legacy codebase

Never:

- Suppress a lint warning with `@SuppressLint` or `// noinspection` without a comment
- Set `abortOnError false` in `lintOptions` for production builds
- Skip lint in CI to save time
