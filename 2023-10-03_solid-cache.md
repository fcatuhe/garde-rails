# Solid Cache

**Author:** Donal McBreen, Programmer, SIP
**Published:** October 3, 2023
**Source:** <https://dev.37signals.com/solid-cache/>

---

We’ve just open-sourced [Solid Cache](https://github.com/rails/solid_cache), a new ActiveRecord::Cache::Store that we use in [Basecamp](https://basecamp.com) and [HEY](https://www.hey.com).

Solid Cache uses a SQL database as its cache store. We get a much larger cache at a fraction of the storage costs of memory caches like Redis or Memcached. For us, that’s a cache size of months rather than days.

While memory access is many times faster than disk, it only accounts for a fraction of cache operation time — there’s also network time, serialization and compression.

In any case, usually we won’t need to go to disk, as databases contain [built-in memory caches](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html).

On Basecamp, compared to our old Redis cache, reads are now about 40% slower. But the cache is 6 times larger, and running on storage that’s 80% cheaper.

You can configure Solid Cache to run in your application’s database, in its own database, or sharded across a cluster of databases.

Since we added Solid Cache to Basecamp, the 95 percentile request duration fell from 375ms to 225ms. There’s no magic here, just the effect of a bigger cache!

![Graph of 95% percentile of Basecamp's request duration](https://dev.37signals.com/assets/images/solid-cache/basecamp-95-percentile.png)

Why the big speed up? Basecamp heavily uses [fragment caching](https://guides.rubyonrails.org/caching_with_rails.html#fragment-caching) so was primed to benefit from the bigger cache.

With lightly used Rails caches you may not see the same performance boost, but you could benefit from simpler and cheaper infrastructure by moving the cache into the primary database.

[Version 0.1.0](https://rubygems.org/gems/solid_cache) is available now. You can read more on [GitHub](https://github.com/rails/solid_cache#readme).
