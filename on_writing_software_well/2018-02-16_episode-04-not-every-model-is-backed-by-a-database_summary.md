# On Writing Software Well #4: Not every model is backed by a database — Summary

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 16, 2018
**Series:** On Writing Software Well
**Episode:** 4 — Not every model is backed by a database
**Source:** <https://www.youtube.com/watch?v=MQw9zF9IehI>

---

> **Note:** The transcript for this episode is unavailable (captions are disabled on the video). The summary below is based on the video description and the series context.

## Overview

DHH explores domain models that don't need database backing — plain Ruby objects (POROs) that encapsulate business logic, coordinate between other objects, or represent transient concepts. He demonstrates how to combine the API elegance of Active Support concerns with the encapsulation of plain objects, pushing back against the Rails convention that every model must inherit from `ActiveRecord::Base`.

## Known Topics (from video description)

- The world outside database-backed models in an MVC context like Rails
- Combining the API beauty of concerns with the encapsulation of plain objects
- When and why to reach for POROs instead of Active Record models

Watch the full episode at: <https://www.youtube.com/watch?v=MQw9zF9IehI>
