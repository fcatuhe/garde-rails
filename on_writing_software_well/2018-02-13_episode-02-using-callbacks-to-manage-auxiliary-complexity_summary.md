# On Writing Software Well #2: Using callbacks to manage auxiliary complexity — Summary

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 13, 2018
**Series:** On Writing Software Well
**Episode:** 2 — Using callbacks to manage auxiliary complexity
**Source:** <https://www.youtube.com/watch?v=m1jOWu7woKM>

---

## Overview

DHH traces the full lifecycle of the @mentions feature in Basecamp 3 — from controller action to notification delivery — to demonstrate how callbacks and concerns can elegantly manage auxiliary complexity. The episode makes the case that side effects, when properly organized, keep the main code path clean and readable.

---

## The Core Argument: Callbacks as Complexity Management

Callbacks allow incidental complexity to live "on the side" rather than polluting the primary flow. The messages controller's `create` action contains zero mention of mentions — it simply records a new message with a subject and content. All the mention-detection logic lives elsewhere, activated through callbacks.

DHH embraces side effects as a positive pattern, pushing back against the functional programming orthodoxy that treats them as inherently harmful.

---

## Tracing the Mentions Feature End-to-End

### 1. Messages Controller → Recording
The controller calls `bucket.record` to create a new message. No mention of mentions, eavesdropping, or notifications anywhere in the controller code.

### 2. Mentionable Concern (on Recording)
Mixed into the `Recording` base class, this concern eavesdrops on every save. It uses two callbacks in sequence:

- **`after_save` → `remember_to_eavesdrop`**: Checks dirty attributes (content changed? draft became active?) while still inside the transaction where change-tracking is available. Stores the decision in an `@eavesdropping` instance variable.
- **`after_commit` → `eavesdrop_for_mentions`**: After the transaction commits, checks whether eavesdropping was flagged and whether it's been suppressed. If both pass, enqueues a background job.

**Key optimization**: The system launched by eavesdropping on every save. They later added the `remember_to_eavesdrop` guard to avoid scanning content that hadn't actually changed (e.g., when a comment touches the parent recording).

### 3. Eavesdropping Job → Eavesdropper Class
The job is a thin shell that instantiates an `Eavesdropper` with the recording and the mentioner (passed explicitly since the job runs outside the original request). The eavesdropper:

- **Scans content for mentionees** — delegating to a scanner that handles both rich text (extracting from HTML) and plain text (regex scan for @call signs)
- **Creates mention records** — using `find_and_initialize_by` to ensure one mention per mentionee per recording

### 4. Mention Model → Delivery
Each mention has an `after_commit` hook (on create and update) that triggers delivery via two channels:
- **Action Notifier** — in-app and native push notifications (a yet-to-be-extracted framework)
- **Action Mailer** — standard email notifications

### 5. Domain Language
DHH highlights the joy of inventing domain-specific terms: **mentioner**, **mentionee**, **eavesdropper**, **eavesdropping**. The terminology makes the code self-documenting and delightful to read.

---

## Suppression: Opting Out of Callbacks

The **project copier** (which imports content from older Basecamp versions) creates recordings with mentions but must not trigger notifications. The solution: a `suppressible` module on the eavesdropper that wraps a block, setting a flag that the callback checks before doing work.

```ruby
Mention::Eavesdropper.suppressed do
  # copying recordings here — no eavesdropping happens
end
```

This is presented as a critical complement to callbacks: you want them to fire by default on the happy path, but you need a clean way to opt out in special cases.

---

## Why Not Service Objects?

DHH explicitly considers alternatives — jamming the logic into the controller, a service object, or a transaction script — and rejects all of them. Mentions is auxiliary complexity: it doesn't deserve center stage in the controller, but it needs to exist and be inspectable.

Concerns provide:
- **Cohesion**: the entire mentions aspect lives in one file
- **Clean main path**: the controller and recording class remain focused
- **Discoverable indirection**: you can follow the callback chain when you need to understand it

---

## Key Themes

- **Side effects are valuable** — when properly organized, they keep the main flow simple
- **Callbacks + jobs** — use callbacks to decide whether work is needed, then defer the actual work to background jobs to avoid blocking the request
- **Concerns as cohesive units** — even though they mix methods into a host class, concerns let you reason about an entire aspect (mentions, events, locking) as one thing
- **Eavesdropping as optimization** — launch with the simple (wasteful) version, then add guards when you discover unnecessary work
- **Suppression** — every callback-heavy system needs a clean opt-out mechanism for exceptional flows
- **Globals (Current)** — briefly mentioned as a way to avoid threading the current person through every layer; promised for a dedicated episode
