# Introducing Solid Queue

**Author:** Rosa Gutiérrez, Programmer, SIP
**Published:** December 18, 2023
**Source:** <https://dev.37signals.com/introducing-solid-queue/>

*A new DB-based queuing backend for Active Job that we open-sourced today.*

---

We’ve just open-sourced [Solid Queue](https://github.com/basecamp/solid_queue), a new backend for [Active Job](https://edgeguides.rubyonrails.org/active_job_basics.html) that we use in [HEY](https://www.hey.com) to run about 1/3 of our roughly 18 million jobs per day. We’ll be moving more jobs in the coming days until we run HEY exclusively using Solid Queue. Besides regular job enqueuing and processing, Solid Queue supports delayed jobs, concurrency controls, pausing queues, numeric priorities per job, and priorities by queue order. Unique jobs and recurring, cron-like tasks are coming very soon.

At 37signals, we’ve been running [Resque](https://github.com/resque/resque) and [Redis](https://redis.io/) for many years. They’ve served us really well, and I am personally a big fan of Redis and Resque, but we’ve also faced a number of challenges using and operating it at scale and had some needs that Resque alone didn’t support. Some of these could be solved with known plugins such as [resque-scheduler](https://github.com/resque/resque-scheduler) or [resque-pool](https://github.com/resque/resque-pool), and others required building our own libraries and supporting tools. As of now, we still need seven different gems just to run jobs in Basecamp and HEY with Resque. This is a section from our `Gemfile` in Basecamp:

```
# Jobs
gem "resque", "~> 2.0.0"
gem "resque_supervised_fork", bc: "resque_supervised_fork"
gem "resque-pool", bc: "resque-pool"
gem "resque-scheduler", github: "resque/resque-scheduler"
gem "resque-pause", bc: "resque-pause"
gem "sequential_jobs", bc: "sequential_jobs"
gem "scheduled_job", bc: "scheduled_job"
```

Looking at this with fresh eyes and with the experience we gained from [Solid Cache](https://dev.37signals.com/solid-cache/), we thought that we could return to using a database to run our jobs instead of relying on Redis, much like when [delayed_job](https://github.com/tobi/delayed_job) was more widely used. Our goal was to enable developers to install Rails, set up a database, and have background job processing right out of the box without having to manage seven different gems and other systems. We reviewed the whole ecosystem of backends for Active Job, and in particular, [GoodJob](https://github.com/bensheldon/good_job) stood out, but unfortunately, it is built only for PostgreSQL. We don’t use PostgreSQL at 37signals, and for a backend that could aspire to become the new Rails default, we needed to support MySQL, PostgreSQL, and SQLite at a minimum.

Performance is typically the primary reason for relying on Redis to run background jobs. However, as GoodJob demonstrates, it’s possible to develop a performant queuing system using a database, provided the system is designed appropriately. In our case, one feature that [PostgreSQL has had for quite some time](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) and [that was finally introduced in MySQL 8](https://dev.mysql.com/blog-archive/mysql-8-0-1-using-skip-locked-and-nowait-to-handle-hot-rows/) has been crucial to our implementation:

```
SELECT ... FOR UPDATE SKIP LOCKED
```

This allows Solid Queue’s workers to fetch and lock jobs without locking other workers. Furthermore, we designed the system so that jobs that are being processed, waiting to be scheduled in the future, failed and needing manual intervention, or blocked because of concurrency limits are isolated from jobs ready to be executed. This design allows us to keep the table that workers poll as small as possible. Finally, our strategy to select jobs makes use of a covering index that we can also use for sorting, enabling us to support both ordered named queues and priorities with very fast, non-blocking queries. Currently, with around 5.6 million jobs executed daily, we perform around 1,300 polling queries per second, with an average query time of 110 µs and examination of 0.02 rows per query.

After a few weeks of using Solid Queue for our most critical jobs in HEY, the main benefit we’ve already reaped has been simplicity and ease of operation. Having everything stored in a relational DB, interfaced by Active Record, has made debugging job-related issues significantly easier compared to troubleshooting issues with Resque. Additionally, we’ve built a dashboard called *Mission Control* to observe and operate jobs that we’re using for both Resque and Solid Queue. We plan to open-source Mission Control early next year to complement Solid Queue.

[Version 0.1.1](https://rubygems.org/gems/solid_queue) is available now. You can read more on [GitHub](https://github.com/basecamp/solid_queue#readme).
