---
name: android-engineering-handbook
description: Engineering handbook for modern Android applications using Kotlin, Jetpack Compose, Modular Architecture, MVI, and Clean Architecture. Read this file first, then load only the documentation required for the current task.
---

# Android Engineering Skill

> Consistency over novelty. Simplicity over complexity. Reuse over duplication.

## Purpose

This handbook defines the engineering standards used to build and maintain Android applications.

It serves as the entry point for all engineering documentation.

---

## Operation rules

- Read this file first.
- Load only the documentation required for the task.
- Reuse existing project patterns.
- Prefer consistency over new abstractions.

---

# Workflow

## Before Writing Code

1. Understand the requested task.
2. Inspect the project structure.
3. Identify the existing architecture.
4. Look for similar implementations.
5. Reuse existing patterns.
6. Read only the required documentation.
7. Make the smallest safe change.

## After Writing Code

- Verify architecture consistency.
- Verify naming conventions.
- Verify formatting.
- Verify KDoc (when applicable).
- Verify previews (Compose components).
- Verify tests (business logic changes).

---

# Engineering Principles

## Always

- Follow the existing architecture.
- Respect module boundaries.
- Keep UI stateless whenever possible.
- Follow unidirectional data flow.
- Prefer composition over inheritance.
- Prefer immutable state.
- Reuse Design System components.
- Document reusable public APIs.
- Keep components small and focused.

## Never

- Put business logic inside Composables.
- Expose DTOs outside the Data layer.
- Return mutable StateFlow.
- Hardcode colors, typography or spacing.
- Duplicate reusable components.
- Break architectural boundaries.
- Modify unrelated files.

---

## Scope

This handbook defines engineering standards, architecture, and implementation guidelines.

It does not replace the existing project architecture.

Whenever project code and documentation disagree, the project code is the source of truth unless instructed otherwise.

---

## Architecture Assumptions

Unless the existing project architecture differs, the following architecture is the default baseline:

- Modular Gradle project.
- MVI presentation layer.
- Unidirectional Data Flow.
- Clean Architecture.
- Dependency Injection.
- Jetpack Compose UI.

---

## Documentation Loading

Only open documentation that is required for the current task.

Do NOT load entire sections if a single document answers the question.

Prefer targeted documentation over broad context.

---

## While Writing Code

- Follow the existing architecture.

- Reuse existing abstractions.

- Keep changes isolated.

- Avoid unnecessary complexity.

- Prefer readability over cleverness.

---

## Stop Before Coding

Before creating new code, verify that:

- A similar implementation does not already exist.
- The project does not already provide the component.
- The architecture supports the proposed solution.
- The change follows existing conventions.

Reuse before creating.

---

## Quick Navigation

| Goal | Documentation |
|------|---------------|
| Understand the architecture | `01-architecture/` |
| Create a feature | `10-templates/feature.md` |
| Build a screen | `10-templates/screen.md` |
| Create a ViewModel | `10-templates/viewmodel.md` |
| Create a Contract | `10-templates/contract.md` |
| Build reusable UI | `02-design-system/` |
| Access data | `05-data/` |
| Implement business rules | `06-domain/` |
| Write tests | `08-testing/` |
| Review quality | `09-quality/` |
| Add a Room DAO | `10-templates/dao.md` |
| Add a DTO | `10-templates/dto.md` |
| Add a Room Entity | `10-templates/entity.md` |
| Add a Retrofit service | `10-templates/retrofit-service.md` |
| Add a bottom sheet | `10-templates/bottom-sheet.md` |
| Load context efficiently | `11-ai/context-loading.md` |
| Browse examples | `examples/` |

---

# Documentation Index

| Topic | Location |
|-------|----------|
| Philosophy | `00-philosophy/` |
| Architecture | `01-architecture/` |
| Design System | `02-design-system/` |
| Compose | `03-compose/` |
| MVI | `04-mvi/` |
| Data | `05-data/` |
| Domain | `06-domain/` |
| Core | `07-core/` |
| Testing | `08-testing/` |
| Quality | `09-quality/` |
| Templates | `10-templates/` |
| AI Engineering | `11-ai/` |
| Examples | `examples/` |

---

## Architecture Priority

When multiple solutions are possible, follow this order:

1. Existing project implementation
2. Existing project conventions
3. This handbook
4. Android best practices

Never replace an existing architecture unless explicitly requested.

---

## Output Expectations

Changes should be:

- Minimal
- Safe
- Incremental
- Easy to review
- Consistent with the existing codebase
- Preserve public APIs unless the task explicitly requires changes.

Avoid unnecessary refactors.

---

## Default Behavior

- Think before editing.
- Read before creating.
- Reuse before replacing.
- Extend before rewriting.
- Keep the codebase predictable.
- Keep it simple, but significant.

---

## Non Goals

This handbook does not:

- Force a specific project structure.
- Replace existing project conventions.
- Override explicit user instructions.
- Introduce architectural changes without request.
- Recommend unnecessary abstractions.