# On Writing Software Well #3: Using globals when the price is right — Summary

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 15, 2018
**Series:** On Writing Software Well
**Episode:** 3 — Using globals when the price is right
**Source:** <https://www.youtube.com/watch?v=D7zUOtlpUPw>

---

## Overview

DHH defends the use of globals in web applications through Rails' `Current` class — a thin wrapper for request-scoped global state extracted from Basecamp 3 into Rails 5.1. He walks through the full authentication flow to show how `Current.person` and `Current.request_id` get set, then demonstrates their power deep inside the event-tracking system where passing these values explicitly would be impractical and noisy.

---

## The Argument: Globals as Sharp Knives

The conventional wisdom says globals are harmful. DHH disagrees — at least for web applications, where every request naturally establishes context (who's logged in, what account, what IP) that needs to be accessible throughout the entire request lifecycle.

The Rails philosophy of **sharp knives** applies: globals can absolutely be misused, but when used deliberately for a small number of request-scoped values, they provide precision and clarity that explicit parameter passing cannot.

---

## Basecamp's Current Class

The `Current` class in Basecamp is small by design — just a handful of attributes:

- **`Current.account`** — derived from the URL (account ID in the path), set via middleware
- **`Current.person`** — set during authentication (cookie, OAuth, etc.), with automatic side effects: sets the account if not already set, sets the time zone from the person's preferences
- **Request details** — `request_id`, `user_agent`, `ip_address` — set via a `before_action` in a concern on `ApplicationController`

The class also provides a `reset` hook to clean up after each request completes.

---

## How Current Gets Populated

### Application Controller
Basecamp's `ApplicationController` has **zero inline code** — it's entirely composed of concerns. Two are relevant here:

1. **`SetCurrentRequestDetails`** — a `before_action` that sets `Current.request_id`, `Current.user_agent`, and `Current.ip_address` from the incoming request
2. **`Authenticate`** — orchestrates the full authentication flow

### Authentication Flow
1. Try **OAuth** authentication
2. Try **cookie** authentication — looks up a signed identity cookie, finds the verified user
3. If neither works, redirect to login or return `401 Unauthorized`

When cookie auth succeeds, the system calls `authenticated(user, by: :cookie)` which logs the auth method and, crucially, **sets `Current.person`** — making the authenticated user available globally for the remainder of the request.

---

## The Payoff: Event Tracking Without Parameter Threading

### The Problem Globals Solve
Every recording (message, to-do, document, etc.) in Basecamp has an `after_create` callback that tracks a creation event. That event needs:

- **The creator** — `Current.person`
- **Request metadata** — `Current.request_id`, `Current.user_agent`, `Current.ip_address`

Without globals, you'd need to thread these values through: controller → `bucket.record` → recording creation → `track_event` → event creation → request details. That's 4–5 layers of parameter passing for auxiliary data that has nothing to do with the primary flow.

### How It Works with Current
The `Eventable` concern on `Recording` fires `track_created` after creation. Inside `track_event`, it plucks `Current.person` as the creator. The `Requested` concern on `Event` builds a request record by plucking `Current.request_id`, `Current.user_agent`, and `Current.ip_address` — all before validation, deep in the model layer.

The result: no controller, no model, no job needs to explicitly pass around authentication or request context. It's just there.

### Why This Matters for Support
The request details tied to every event power an admin interface that lets support look up: who made a change, from what IP, with what browser. This is invaluable for investigations — but it's purely auxiliary. It should never clutter the main code path of creating a message or recording.

---

## Guidelines: How Much Is Too Much?

DHH offers a practical heuristic: if your `Current` class grows to 10–15–20 attributes, you've probably gone too far and should reconsider your factoring.

Basecamp's `Current` has just a few attributes — account, person, and request details. That's the sweet spot: a small number of truly global, request-scoped values that many parts of the system need but that would be painful to pass explicitly.

---

## Key Themes

- **Globals aren't inherently evil** — in request-scoped web applications, a small number of globals is cleaner than threading context through every layer
- **Sharp knives** — powerful tools that can hurt if misused but enable elegant solutions when applied deliberately
- **Concerns as a recurring pattern** — just like mentions in Episode 2, event tracking uses concerns + callbacks + associations to encapsulate auxiliary complexity
- **Auxiliary details stay auxiliary** — request metadata for forensic investigation should never pollute the main code path of content creation
- **`Current` as a framework feature** — extracted from years of Basecamp usage into Rails itself, providing a standard pattern that other apps can adopt
- **Practical limits** — keep the number of global attributes small; if it grows, rethink your architecture
