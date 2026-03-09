# On Writing Software Well #5: Testing without test damage or excessive isolation — Summary

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 20, 2018
**Series:** On Writing Software Well
**Episode:** 5 — Testing without test damage or excessive isolation
**Source:** <https://www.youtube.com/watch?v=Tc5z64XIwIY>

---

## Overview

DHH demonstrates how Basecamp tests a concrete feature — document locking — across both model and controller layers, without mocks, stubs, or excessive isolation. The episode is a practical rebuttal to the "TDD is Dead" misconception: Basecamp has tens of thousands of lines of test code, but optimizes for confidence and readability over raw speed.

---

## Testing Philosophy

### What "TDD is Dead" Actually Meant
DHH clarifies that his criticism was never about testing itself, but about the **trade-offs imposed by strict unit testing orthodoxy**:

- Isolating every class behind mocks and stubs
- Optimizing for millisecond-fast test runs at the expense of testing real behavior
- Letting test requirements distort production code ("test damage")

### The Alternative: Test the Real Thing
Basecamp's approach:

- **Hit the real database** — no in-memory fakes
- **Render real views** — including asset compilation in controller tests
- **Let callbacks run** — don't stub out concerns or side effects
- **Accept "fast enough"** — 0.5s per model test, 0.7s per controller test is fine
- **Maximize confidence** — the purpose of tests is shipping high-quality features, not having fast tests

---

## The Feature Under Test: Documents + Locking

### Document Model
A thin class — mostly data attributes (title, content via Trix rich text), a default title ("Untitled"), and three predicate methods (`auto_positioning?`, `subscribable?`, `exportable?`) that recording's generic concerns interrogate to determine capabilities.

### Lockable Concern (on Recording)
A generic concern applicable to any content type. Maintains a `lock` association (owned by a user) with DSL methods to lock, unlock, and check lock status. Applied to documents through the recording base class.

### The Subscribable Pattern
Documents declare themselves `subscribable?`, and the generic subscribable concern on recording checks this predicate. This **inverts the dependency** — the generic concern doesn't maintain a list of subscribable types; each type answers for itself.

---

## Model Testing

### Document Test
- **Setup**: only sets `Current.person` (globals from Episode 3)
- **Fixtures, not factories**: pulls pre-configured records (`buckets(:anniversary_project)`) instead of building objects inline
- **Real database interactions**: `bucket.record(new_document)` writes to the database, triggers all callbacks
- **Simple assertions**: `assert` and `assert_equal` only — DHH's preferred stock assertion vocabulary

### Version Test
Tests that updating a document preserves previous versions:
- Uses `travel 5.seconds` to avoid `created_at` ordering ambiguity (a pragmatic workaround — in production, edits are always seconds apart)
- Asserts both the latest version and the previous version are accessible

### Lock Test
Lives under `test/models/recording/lock_test` — mirroring the concern's structure. Tests locking, lock ownership, and lock-by-same-user scenarios. Despite recording having dozens of concerns mixed in, the test focuses exclusively on locking and ignores everything else. All other callbacks run freely.

### Speed
Four model tests: **~2 seconds total**. Not blazing fast, but fast enough — especially with Spring preloading the Rails environment.

---

## Controller Testing

### Documents Controller Test
- Inherits from `ActionDispatch::IntegrationTest` — not a narrow controller unit test
- **Tests the `new` action first**: asserts the form renders correctly (including full view rendering and asset compilation)
- **Tests `create`**: posts to the controller, follows the redirect, asserts the resulting page contains the expected h1
- **Breadcrumb assertion**: a custom domain-specific assertion for navigation correctness

### Locks Controller Test
- Tests the generic locks controller (works for any lockable content type, not just documents)
- Scenarios: locked by another user (shows lock screen), locked by same user (shows "you're editing"), not locked (proceeds normally), force-unlock
- Uses `sign_in` helper to set up the current user for integration tests

### Speed
Ten controller tests: **~7 seconds total**. Slower than model tests because they render full views and compile assets. DHH considers this a worthy trade-off.

---

## Fixtures Over Factories

Basecamp uses **vanilla Rails fixtures** exclusively — no FactoryBot or similar:

- **4 document fixtures** — enough for all document tests
- **Many recording fixtures** — since every content type has a corresponding recording
- Pre-configured "characters in a world" that tests can pull from without setup boilerplate
- Test-specific objects (like a new document for version testing) are created inline when they represent something particular to that test

The advantage: tests read cleanly because they don't start with 10 lines of factory setup. The base world is assumed.

---

## Multiple Assertions Per Test

DHH explicitly rejects the "one assertion per test" rule:

> You should have all the assertions that you need to feel comfortable about testing one aspect of what you're doing.

- Model tests average 2–4 assertions
- Controller tests average 2–3 assertions
- The unit of testing is **one aspect or action**, not one assertion

---

## Key Themes

- **Confidence over speed** — real database hits and real view rendering catch real bugs; mocks catch mock bugs
- **No test damage** — production code should never be distorted to satisfy testing requirements
- **Concerns don't complicate testing** — even though recording has dozens of concerns, testing one concern is straightforward; just let the others run
- **Fixtures as shared world** — a stable base of test data that eliminates setup noise
- **Integration-style controller tests** — stop just short of browser automation; test HTTP requests/responses and HTML output directly
- **Parallelized tests coming in Rails 6** — acknowledged as useful for large suites, but not needed for individual feature tests
- **Two assertions are enough** — `assert` and `assert_equal` cover virtually everything at the model level
