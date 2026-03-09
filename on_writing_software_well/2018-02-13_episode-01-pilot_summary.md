# On Writing Software (well?) #1: Pilot Episode — Summary

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 13, 2018
**Series:** On Writing Software Well
**Episode:** 1 — Pilot Episode
**Source:** <https://www.youtube.com/watch?v=H5i1gdwe1Ls>

---

## Overview

DHH introduces his new video series "On Writing Software Well" — a casual, conversational look at real code from Basecamp 3, focused not on specific libraries or frameworks, but on the everyday judgment calls programmers face when trying to make code clearer and more maintainable. The pilot covers two refactoring examples from the Basecamp 3 codebase.

---

## Part 1: Replacing Code Comments with Explaining Constants

### The Problem

In the `Access` model (the join between people and "buckets" — Basecamp's abstraction for projects, teams, and circles), there's a `remove_inaccessible_records` method with a code comment explaining a 30-second grace period. When someone's access is revoked, a background job destroys their associated records (readings, bookmarkings, etc.), but the system waits 30 seconds before firing to allow an "undo" in case the removal was accidental.

DHH's position: **code comments are usually a smell**. If you need prose to explain what the code does, the code itself isn't clear enough.

### The Refactoring Steps

1. **Identify what the comment actually explains** — not the conditional logic, but the "magic number" `30.seconds`. The comment is compensating for a value whose purpose isn't self-evident.

2. **Extract an explaining variable** — replace the inline `30.seconds` with a named local variable: `grace_period_for_removing_inaccessible_records = 30.seconds`. This already makes the comment redundant.

3. **Promote to a constant** — since the value never changes, it shouldn't be recomputed on every method call. Extract it as `GRACE_PERIOD_FOR_REMOVING_INACCESSIBLE_RECORDS = 30.seconds`.

4. **Make it private** — this constant is an implementation detail, not part of the public API. Move it below the `private` keyword.

5. **Place it near its usage** — position the constant declaration immediately next to the method that references it, so they're read together.

### The Tradeoff

This creates a conflict between two aesthetic principles:

- **Table of contents ordering**: private methods should appear in the order they're called (via callbacks: `after_destroy` → `after_destroy_commit`).
- **Proximity**: constants should be declared close to where they're used.

DHH chooses proximity over ordering in this case, noting that with only two private methods, the table-of-contents violation is minor. This is exactly the kind of nuanced judgment that automated style checkers can't capture — the goal is code that's pleasant to read and easy to return to.

---

## Part 2: Extracting `create_or_find_by` into Rails

### The Problem

The `Administratorship` model (a join between accounts and people representing admin privileges) has a `grant` method on its has-many association. Granting an administratorship should be idempotent — if someone already has one, don't create a duplicate.

The existing implementation:
- Attempts to `create` the record
- Catches the `ActiveRecord::RecordNotUnique` exception (from the database's unique index)
- Falls back to finding the existing record
- Requires a code comment to explain the exception-as-control-flow pattern

### Why Not `find_or_create_by`?

Rails already has `find_or_create_by`, but it works in the wrong order for high-traffic applications. It:
1. Opens a transaction
2. Runs a `SELECT` (find)
3. Runs an `INSERT` if not found (create)

This two-step process is vulnerable to race conditions and stale reads on busy apps like Basecamp.

### The Solution: Flip the Order

The better approach is **create first, find second**:
1. Attempt the `INSERT`
2. If the unique index rejects it, `SELECT` the existing record

This is a single atomic operation backed by the database constraint — no race condition possible.

DHH proposes extracting this as `create_or_find_by` in Rails itself, which would:
- Eliminate the boilerplate try/catch pattern in Basecamp
- Reduce `grant` to a simple alias: `alias grant create_or_find_by`
- Provide a safer default for all Rails applications

He even suggests that `create_or_find_by` should perhaps become the recommended approach, with `find_or_create_by` potentially deprecated.

---

## Key Themes

- **Code comments as a smell**: if you need a comment, the code probably isn't expressive enough. Extract explaining constants or methods instead.
- **Aesthetic judgment over mechanical rules**: good code style involves tradeoffs between competing principles (ordering, proximity, naming). Automated linters can't make these calls.
- **Extracting from applications into frameworks**: reading through a production codebase reveals patterns that belong in Rails itself. The `create_or_find_by` method was later [added to Rails](https://github.com/rails/rails/pull/31989).
- **Domain modeling**: encapsulate concepts as first-class objects (`Access`, `Administratorship`) rather than booleans or flags. Give domain concepts a place to live.
- **Refactoring as ongoing practice**: DHH periodically reads through the entire Basecamp codebase to revisit past decisions with fresh eyes — a practice he calls his annual "refactoring" branch.
