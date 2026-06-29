# SonarQube / SonarCloud

## When to Use

SonarQube/SonarCloud is an optional quality layer on top of lint and ktlint.

Add it when:
- The team is large enough that individual reviews miss things
- You need to track coverage trends over time
- A client or compliance requirement mandates it

For small/solo projects, lint + ktlint + JaCoCo coverage report is sufficient.

---

## SonarCloud Setup (CI)

```yaml
# .github/workflows/sonar.yml
name: SonarCloud

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  sonar:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Sonar needs full git history

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run Tests with Coverage
        run: ./gradlew test jacocoTestReport

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## sonar-project.properties

```properties
sonar.projectKey=myorg_myapp
sonar.organization=myorg

sonar.sources=app/src/main,presentation/src/main,domain/src/main,data/src/main
sonar.tests=app/src/test,presentation/src/test,domain/src/test,data/src/test
sonar.java.coveragePlugin=jacoco
sonar.coverage.jacoco.xmlReportPaths=**/build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml

sonar.exclusions=**/generated/**,**/*Test*.kt,**/*Fake*.kt
sonar.coverage.exclusions=**/di/**,**/*Module.kt,**/*Router.kt
```

---

## JaCoCo Coverage Report

```kotlin
// build.gradle.kts (app or root)
tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest")

    reports {
        xml.required.set(true)
        html.required.set(true)
    }

    val fileFilter = listOf(
        "**/R.class", "**/R$*.class", "**/BuildConfig.*",
        "**/Manifest*.*", "**/*Test*.*", "**/di/**",
    )

    val debugTree = fileTree(layout.buildDirectory.dir("intermediates/javac/debug")) {
        exclude(fileFilter)
    }
    val kotlinDebugTree = fileTree(layout.buildDirectory.dir("tmp/kotlin-classes/debug")) {
        exclude(fileFilter)
    }

    classDirectories.setFrom(files(debugTree, kotlinDebugTree))
    sourceDirectories.setFrom(files("src/main/java", "src/main/kotlin"))
    executionData.setFrom(fileTree(layout.buildDirectory) { include("**/*.exec", "**/*.ec") })
}
```

---

## Quality Gate Rules (SonarQube Defaults)

| Metric | Minimum |
|--------|--------|
| Coverage on new code | 80% |
| Duplicated lines on new code | < 3% |
| Maintainability rating | A |
| Reliability rating | A |
| Security rating | A |

Customize in SonarCloud → Quality Gates for your project.

---

## What SonarQube Adds Over Lint

| Tool | Catches |
|------|---------|
| Android Lint | Android-specific issues, resource misuse |
| Ktlint | Formatting |
| Detekt | Kotlin code smells, complexity |
| SonarQube | Coverage trends, duplication, cross-file smell tracking, historical dashboards |

SonarQube is additive — it does not replace the other tools.
