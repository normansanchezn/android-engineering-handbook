# AI-Assisted Code Review

## Purpose

Use AI to catch architecture violations, missing error handling, and contract deviations before human review.

AI code review is most effective when given a specific checklist — not an open-ended "review this PR."

---

## Review Prompt Template

```
Review the following diff against the Android Engineering Handbook standards.

Check only:
1. Architecture violations (wrong layer for business logic, DTOs crossing layer boundaries)
2. Missing Either<Failure, T> wrapping on repository or use case methods
3. MVI contract violations (stateful child composables, effects not in LaunchedEffect, missing updateState)
4. Hardcoded strings in composables or ViewModels
5. Missing test coverage for new ViewModel events

For each finding:
- File and line number
- What rule is violated
- What the fix is

Skip formatting, naming style, and comments unless they change correctness.

[paste diff here]
```

---

## Checklist-Driven Review

Map each checklist item from `09-quality/code-review.md` to an explicit question:

```
For the diff below, answer yes/no for each:

Architecture:
[ ] Repository interface in domain/, implementation in data/?
[ ] All repository and use case methods return Either<Failure, T>?
[ ] No business logic in ViewModel event handlers?
[ ] No DTOs in domain or presentation layers?

MVI:
[ ] Only the root Screen composable calls hiltViewModel()?
[ ] Effects collected in LaunchedEffect(Unit)?
[ ] All state mutations go through updateState {}?

Compose:
[ ] collectAsStateWithLifecycle() used (not collectAsState())?
[ ] LazyList items have key = { item.id }?
[ ] Every composable accepts modifier: Modifier = Modifier?

Testing:
[ ] New events have tests?
[ ] FakeRepositories used (not mocks)?
```

---

## Refactoring Review

Before AI-assisted refactoring, establish a contract:

```
I'm going to refactor [FileName]. 

Before refactoring:
- Summarize what the current code does
- List its public API (what it exposes to callers)
- Identify any tests that cover it

The refactoring goal: [specific goal]

Constraint: the public API must not change. Callers must not need modification.
```

This ensures the AI does not silently change interfaces that other modules depend on.

---

## AI-Assisted Documentation

For generating KDoc on public APIs:

```
Generate KDoc for the following public interface.

Rules:
- Document the contract, not the implementation
- Do not describe what the code does (the reader can see that)
- Describe WHY: preconditions, postconditions, error scenarios
- No @throws — exceptions are not used; errors are returned as Either.Error
- Keep descriptions under 2 lines

[paste interface here]
```

---

## Limits of AI Review

AI review is reliable for:
- Pattern matching against documented rules (Either, MVI, layer boundaries)
- Syntax-level checks (missing annotations, wrong return types)
- Completeness checks (all events handled, all Failure types mapped)

AI review is unreliable for:
- Business logic correctness (AI does not know the product requirements)
- Performance characteristics (profiling required)
- Thread safety in complex concurrent code
- Security implications of data handling

Human review is required for all items in the second list.

---

## Review Workflow

1. Run Detekt and fix all findings first
2. Run AI review with the checklist prompt — fix findings
3. Human review focuses on business logic, security, and edge cases
4. AI review catches mechanical issues; human review catches judgment issues
