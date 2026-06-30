# Prompting for Android Tasks

## Principle

Effective prompts give the AI enough context to follow existing patterns — not enough to invent new ones.

The goal is to constrain the solution space to what the project already uses, not to describe what the ideal solution looks like in abstract.

---

## Context to Always Include

1. **What already exists** — a similar file path, a pattern already in the project
2. **What module** — where the new code belongs (`presentation`, `domain`, `data`)
3. **What the feature is called** — the noun used consistently in the codebase
4. **What the task requires** — the specific change, not the background

---

## Prompt Templates by Task

### New feature

```
Create a new feature called [FeatureName] following the pattern in [existing-feature-path].

Module structure: presentation/feature/[feature-name]/

The feature needs:
- [List of operations: display X, submit Y, navigate to Z]

Domain model fields: [field list]
Repository methods: [method signatures]
```

### New screen

```
Create a screen for [FeatureName] following the pattern in [ExistingScreen.kt].

Contract is at [path/to/Contract.kt].
The screen needs to:
- Display [list of content]
- Handle these events: [event list]
- Navigate to [destination] on [trigger]
```

### Bug fix

```
Bug: [exact description of wrong behavior]
File: [path:line]
Expected: [what should happen]
Actual: [what happens]

Do not change any files outside [specific file or module].
```

### Code review

```
Review [file or diff] against the MVI contract rules in 09-quality/code-review.md.

Flag:
- Architecture violations
- Missing Either wrapping
- Business logic in ViewModel or Screen
- Hardcoded strings
- Missing tests for new events

Do not flag style preferences.
```

### Adding a test

```
Add unit tests for [ViewModelName] covering:
- [Event name] success path
- [Event name] failure path

Follow the pattern in [ExistingViewModelTest.kt].
Use FakeXxxRepository, not mocks.
Use MainDispatcherRule.
```

---

## Constraining the Output

Include explicit constraints to prevent scope creep:

```
Do not:
- Modify files outside [module]
- Add abstractions not present in similar features
- Create new base classes
- Change existing interfaces
```

---

## Referencing the Handbook

When the handbook section is relevant, name it explicitly:

```
Follow the offline-first pattern from 01-architecture/offline-first.md.
Use safeApiCall as documented in 07-core/result.md.
Apply the DTO pattern from 10-templates/dto.md.
```

This is more effective than describing the pattern inline — the AI loads the document and applies it directly.

---

## Signs of a Weak Prompt

| Signal | Problem |
|--------|---------|
| "Create a feature" with no existing example | AI invents patterns instead of reusing |
| "Add error handling" without specifying the type | AI adds try/catch instead of Either |
| "Make it work" | No definition of correctness |
| No module specified | AI guesses where code belongs |
| No constraint on scope | AI modifies unrelated files |

---

## Signs of a Strong Prompt

- Names an existing file as the reference pattern
- Specifies the exact module
- Lists the exact operations needed
- Constrains what should NOT be changed
- References handbook sections for non-obvious conventions
