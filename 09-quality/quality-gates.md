# Quality Gates

## What a Quality Gate Is

A quality gate is a pass/fail check that blocks a merge if the code doesn't meet a defined standard.

The goal: catch regressions automatically so the team doesn't have to catch them manually in reviews.

---

## Minimum Gates (CI)

Every PR must pass:

| Gate | Command | Blocks merge on |
|------|---------|----------------|
| Compile | `./gradlew assembleDebug` | Compilation error |
| Unit tests | `./gradlew test` | Any test failure |
| Ktlint | `./gradlew ktlintCheck` | Formatting violation |
| Lint | `./gradlew lint` | New lint error (not baseline) |

---

## Recommended Gates

| Gate | Command | When to add |
|------|---------|-------------|
| Instrumented tests | `./gradlew connectedAndroidTest` | When UI test suite is established |
| Detekt | `./gradlew detekt` | When complexity rules are configured |
| Test coverage minimum | JaCoCo threshold | When coverage is tracked |

---

## GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ hashFiles('**/*.gradle*', '**/libs.versions.toml') }}

      - name: Build
        run: ./gradlew assembleDebug

      - name: Ktlint
        run: ./gradlew ktlintCheck

      - name: Lint
        run: ./gradlew lint

      - name: Unit Tests
        run: ./gradlew test

      - name: Upload Lint Report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: lint-report
          path: '**/build/reports/lint-results*.html'
```

---

## Branch Protection

Configure branch protection on `main`:

- Require status checks to pass before merging
- Require at least 1 approval
- Dismiss stale reviews on new pushes
- Do not allow force pushes to `main`

---

## Local Pre-commit Hook (Optional)

Run ktlint locally before every commit:

```bash
# .git/hooks/pre-commit
#!/bin/sh
./gradlew ktlintCheck --daemon
```

```bash
chmod +x .git/hooks/pre-commit
```

Or use `lefthook` / `husky` for team-wide hook management.

---

## Rules

Always:

- All gates run on every PR — not just on merge to main
- Fix failing gates immediately — don't merge with a suppressed check
- Keep CI fast: target under 5 minutes for the unit test + lint pass

Never:

- Merge with a failing gate using admin override, except for a documented emergency
- Add a gate that regularly produces false positives — it trains the team to ignore failures
