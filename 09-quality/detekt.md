# Detekt

## Purpose

Detekt performs static analysis on Kotlin code. It catches code smells, complexity issues, and style violations before code review.

---

## Setup

```kotlin
// build.gradle.kts (root)
plugins {
    id("io.gitlab.arturbosch.detekt") version libs.versions.detekt.get()
}

detekt {
    config.setFrom(files("$rootDir/config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
    allRules = false
    parallel = true
}

dependencies {
    detektPlugins(libs.detekt.formatting)
}
```

```kotlin
// Run detekt
./gradlew detekt

// Run detekt with auto-correct for formatting
./gradlew detektMain --auto-correct
```

---

## Configuration

```yaml
# config/detekt/detekt.yml

build:
  maxIssues: 0

complexity:
  ComplexCondition:
    active: true
    threshold: 4
  CyclomaticComplexMethod:
    active: true
    threshold: 15
  LongMethod:
    active: true
    threshold: 60
  LongParameterList:
    active: true
    functionThreshold: 8
    constructorThreshold: 8
  TooManyFunctions:
    active: true
    thresholdInFiles: 20
    thresholdInClasses: 15

naming:
  FunctionNaming:
    active: true
    functionPattern: '[a-z][a-zA-Z0-9]*'
    excludes: ['**/test/**', '**/androidTest/**']

style:
  ForbiddenComment:
    active: true
    comments: ['FIXME:', 'STOPSHIP:', 'TODO:']
  MagicNumber:
    active: true
    ignoreNumbers: ['-1', '0', '1', '2']
    ignorePropertyDeclaration: true
    ignoreConstantDeclaration: true
  MaxLineLength:
    active: true
    maxLineLength: 140

formatting:
  active: true
  android: true
  autoCorrect: true
```

---

## Suppressing False Positives

Use `@Suppress` at the narrowest possible scope:

```kotlin
// Suppress a single function
@Suppress("LongParameterList")
fun createUiStrings(
    toolbarTitle: String,
    nameHint: String,
    nameError: String,
    labelHint: String,
    labelError: String,
    submitLabel: String,
    cancelLabel: String,
    networkError: String,
): ScreenUiModel = ScreenUiModel(...)

// Suppress a single expression
val result = someComplexExpression  // detekt:disable:MagicNumber
```

Never suppress entire files or modules. Never suppress `LongMethod` instead of extracting the method.

---

## Baseline

When introducing Detekt to an existing project, generate a baseline to ignore existing issues and only catch new ones:

```bash
./gradlew detektBaseline
```

This generates `config/detekt/baseline.xml`. Commit the baseline and set `ignoreFailures = false` going forward.

Remove baseline entries as underlying issues are resolved.

---

## CI Integration

```yaml
# .github/workflows/quality.yml
- name: Run Detekt
  run: ./gradlew detekt

- name: Upload Detekt report
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: build/reports/detekt/detekt.sarif
```

Detekt failures block the PR merge.

---

## Rules

Always:

- Use a shared `detekt.yml` at the project root — not per-module config
- Enable `buildUponDefaultConfig = true` to extend the default ruleset
- Include `detekt.formatting` plugin for Kotlin formatting checks
- Add detekt to CI with `maxIssues: 0`

Never:

- Suppress issues at file or module scope
- Disable rules to pass CI without fixing the underlying problem
- Commit with unresolved `FIXME` or `STOPSHIP` comments (blocked by `ForbiddenComment`)
