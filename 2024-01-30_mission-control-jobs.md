# Mission Control — Jobs

**Author:** Rosa Gutiérrez, Programmer, SIP
**Published:** January 30, 2024
**Source:** <https://dev.37signals.com/mission-control-jobs/>

*Dashboard and Active Job extensions to operate and troubleshoot background jobs.*

---

As promised [back when we introduced Solid Queue](https://dev.37signals.com/introducing-solid-queue/), today we’ve open-sourced [Mission Control — Jobs](https://github.com/basecamp/mission_control-jobs), a dashboard and set of extensions to operate and observe background jobs, that we’ve been using for over a year, in the beginning with [Resque](https://github.com/resque/resque) only, and later with both Resque and Solid Queue.

This is how it looks:

![The main view of Mission Control — Jobs](https://dev.37signals.com/assets/images/mission-control-jobs/screenshot.png)

The main view of Mission Control — Jobs, with both Resque and Solid Queue

[Version 0.1.0](https://rubygems.org/gems/mission_control-jobs) is available now. You can read more on [GitHub](https://github.com/basecamp/mission_control-jobs#readme). We hope you find it useful! If you are interested in contributing, we have prepared [a list of improvements and new features](https://github.com/basecamp/mission_control-jobs/issues) we’d like to see in the future, some of them great for first-time contributors.

---

## A bit of history — why did we build this?

My colleague [Jorge](https://dev.37signals.com/author/jorge/) built Mission Control — Jobs in 2022. Until then, we had been using [resque-web](https://github.com/resque/resque-web) and a simple made-in-house dashboard to manage Resque jobs for all our apps. We still use this for our older apps.

![Resque web, fully functional and operative](https://dev.37signals.com/assets/images/mission-control-jobs/resque-web.png)

Resque web, fully functional and operative

Retrying failed jobs in `resque-web` is a bit cumbersome (for example, jobs aren’t cleared after they’re retried) and requires many clicks. To make things a bit more complicated, in some of our apps we also wrap some of our jobs in a wrapper job that shows up as that in `resque-web`, making it harder to observe certain jobs. Because of this, in 2012 we built our own dashboard that centralized Resque servers for all our apps to make these routine tasks (checking failed jobs, retrying or discarding them, and inspecting errors) easier. We still use it for our legacy apps.

![Our minimalistic yet practical dashboard for Resque jobs](https://dev.37signals.com/assets/images/mission-control-jobs/dash-resque.png)

Our minimalistic yet practical dashboard for Resque jobs

Both `resque-web` and our internal dashboard served us well for regular, routine tasks. However, we couldn’t rely on them for exceptional, incident-like instances when we had to operate over or inspect large sets of jobs and selectively discard or retry them in bulk. For these cases, we always ended up writing custom scripts that would later be added to a runbook. After several of those, we had a collection of hacky, hard-to-modify scripts targeting different Resque versions with very low-level and hard-to-understand operations, let alone modify in a different type of incident while things are on fire.

Despite how great Resque is, it doesn’t offer a high-level, well-documented API that makes operating with jobs simple or safe. Some operations in bulk can be quite dangerous depending on the volume of jobs we’re trying to retry or discard, and there’s no warning or clue that something might be a big mistake because you’re, perhaps, loading a million of jobs from Redis at once without being aware of it.

Because of this, and after being burned by some of these issues too many times, Jorge set two goals for a project to improve this situation:
- Extend Active Job with a query-like API that would be standard across adapters and that would allow fetching, filtering, retrying and discarding jobs.

```
ActiveJob.jobs.failed.where(type: "MyJob").retry_all
ActiveJob.jobs.failed.where(type: "MyJob").discard_all
ActiveJob.jobs.find("3121dsa-123flgj4-zpq123312").retry
```
- Provide an admin UI that would use this new API and offer the tools that would be most useful to us during incidents: pausing and resuming queues and examining, retrying and discarding jobs selectively, with filters by job class and queue name.

All these operations should be safe by default, using batches and delays when necessary.

He shipped this internally in September 2022, and we’ve been using it with great success for our two main apps, [HEY](https://www.hey.com/) and [Basecamp 4](https://basecamp.com/). It has made a massive difference during minor issues and bigger incidents, and also improved routine tasks.

In 2023, when we started building Solid Queue, it was clear we would use Mission Control to operate it. We needed to have it working before moving jobs over from Resque, in case we had to inspect, retry or discard Solid Queue jobs. Jorge had done an incredible job to make it generic and easy to add a new adapter, but Resque’s and Solid Queue’s underlying data structures couldn’t be more different, so there was still some work to do on removing or working around limitations imposed by Resque’s.

Most recently, we’ve added quite a few new features to it, all targeted to Solid Queue and enhanced Solid Queue with proper bulk discard and retry operations. We’ve also improved the setup for the most simple case, having a single Rails app to manage and a single adapter, to make it ready to be shared with the world. We hope you like it!
