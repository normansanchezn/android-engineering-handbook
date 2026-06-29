# KDoc

## Default: No Comments

Well-named functions, parameters, and classes are self-documenting. Do not add KDoc by default.

---

## When to Write KDoc

Write KDoc only when the **why** is non-obvious from the name and signature alone.

| Worth documenting | Not worth documenting |
|------------------|-----------------------|
| A subtle invariant that would surprise a reader | `fun getGroups(): List<GroupModel>` |
| A workaround for a specific API or OS bug | `data class GroupModel(val id: String, val name: String)` |
| A constraint that comes from outside the code (legal, spec) | Standard CRUD usecase |
| A non-obvious parameter default | Simple constructor injection |

---

## Format

One short line. No multi-paragraph blocks. No `@param` / `@return` repetition of what the signature already shows.

```kotlin
// ✅ One line, explains the non-obvious
/**
 * Requires the group name to be non-blank; the API rejects blank names with a 400.
 */
class SaveGroupUseCase @Inject constructor(...) { ... }

// ❌ Restates the obvious
/**
 * Saves a group.
 * @param group The group to save.
 * @return Either success or failure.
 */
class SaveGroupUseCase @Inject constructor(...) { ... }
```

---

## Inline Comments

Use `//` inline comments sparingly — only for the same cases as KDoc.

```kotlin
// ✅ Explains a non-obvious behavior
_effect = MutableSharedFlow(extraBufferCapacity = 1) // prevents suspension if collector is slow

// ❌ Narrates the obvious
// Update the state
updateState { copy(isLoading = true) }
```

---

## What Never to Comment

- Task references ("added for the login flow", "fixes #123") — belongs in the commit/PR
- Caller references ("called by XxxScreen") — code changes, comments don't
- Disabled code blocks — delete them; git history has them
- TODOs that have no owner or deadline — they rot; use your issue tracker

---

## Rules

Always:

- Prefer renaming to commenting — if you need a comment to explain a name, fix the name first
- Keep comments short — one line maximum for inline, one short paragraph for KDoc

Never:

- Write a KDoc for every public function by default
- Repeat what the function signature says
- Leave commented-out code in a PR
