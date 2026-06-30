# Context Loading

## Purpose

This handbook is designed for selective loading. Load only the documentation required for the current task — loading the full handbook wastes context and adds noise.

---

## Loading Strategy

The entry point is `SKILL.md`. It provides enough orientation to determine which documents to load next.

```
SKILL.md (always)
  └── Load only the section(s) relevant to the task
```

Never load an entire section when a single document answers the question.

---

## Task → Documentation Map

| Task | Load |
|------|------|
| Create a new feature | `10-templates/feature.md` → then follow its references |
| Build a new screen | `10-templates/screen.md`, `04-mvi/contracts.md` |
| Create a ViewModel | `10-templates/viewmodel.md`, `04-mvi/mvi.md` |
| Add a repository | `10-templates/repository.md`, `05-data/repositories.md` |
| Add a use case | `10-templates/usecase.md`, `06-domain/usecases.md` |
| Add a Room DAO | `10-templates/dao.md`, `10-templates/entity.md` |
| Add a Retrofit service | `10-templates/retrofit-service.md`, `05-data/retrofit.md` |
| Add serialization | `05-data/serialization.md` |
| Configure networking | `05-data/networking.md` |
| Debug recomposition | `03-compose/recomposition.md` |
| Implement offline-first | `01-architecture/offline-first.md`, `05-data/datasource.md` |
| Add animations | `02-design-system/motion.md` |
| Implement side effects | `03-compose/side-effects.md` |
| Write unit tests | `08-testing/unit-testing.md`, `08-testing/fake-repositories.md` |
| Write integration tests | `08-testing/integration-testing.md` |
| Test coroutines / Flows | `08-testing/coroutine-testing.md` |
| Add a design system component | `02-design-system/components.md`, `02-design-system/atomic-design.md` |
| Review code | `09-quality/code-review.md`, `09-quality/checklist.md` |
| Add a dependency | `09-quality/dependency-review.md` |
| Add a feature module | `01-architecture/feature-modules.md` |

---

## Depth Control

### Shallow load (overview + rules only)

For orientation or code review: read the target document's **Rules** section first. Rules sections are at the end of every document and summarize all constraints concisely.

### Deep load (full implementation)

For writing new code: read the full document including code examples.

### Reference only

For a specific pattern or template: search for the relevant code block in the target document without reading surrounding prose.

---

## What Not to Load

| Document | Skip when |
|----------|----------|
| `00-philosophy/` | The task is implementation, not architecture design |
| `01-architecture/architecture.md` | You already know the project structure |
| `09-quality/sonar.md` | Task is feature development, not CI configuration |
| Examples in `examples/` | A template already covers the pattern |

---

## Project Code Takes Precedence

Before loading handbook documentation, inspect the existing project code.

An existing implementation in the project is the most reliable reference:

1. Find a similar feature already implemented
2. Follow its patterns exactly
3. Load handbook documentation only when the project does not have a prior example

The handbook defines standards. The project defines the implementation.
