# Basecamp code runs 18% faster with YJIT

**Author:** Jacopo Beschi, Programmer, SIP
**Published:** December 1, 2023
**Source:** <https://dev.37signals.com/yjit-is-fast/>

*You should give it a try if you haven’t done it yet.*

---

Basecamp runs ~18% faster with YJIT. In this post I’ll share our setup, and the performance improvements we achieved.

---

## Our setup

Basecamp is currently running Ruby 3.3.0-preview3 and Rails Edge (master branch).

We configure YJIT in our servers via `RUBYOPT=--yjit-disable --yjit-exec-mem-size=192` and then enable YJIT at runtime via `RubyVM::YJIT.enable`. This allows us to achieve a faster boot time, compared to enabling YJIT at boot. We also enabled [yjit stats](https://github.com/ruby/ruby/blob/ruby_3_2/doc/yjit/yjit.md#other-statistics) to a few servers.

We track our metrics in [Prometheus](https://dev.37signals.com/prometheus-metrics-at-37signals/), using Yabeda to instrument, and Grafana to render them.

---

## Performance

We found that our app works at best with a bigger memory size (192 MiB) than the default (128 MiB). We set it via `--yjit-exec-mem-size=192` inside the `RUBYOPT` ENV variable. The value to set varies for each application. To find the best value for your app, I suggest you to measure the `RubyVM::YJIT.runtime_stats[:code_region_size]` and ensure it’s smaller than the `--yjit-exec-mem-size`.

### Response time

With YJIT we saw improvements across the board in the range of 16-22%:
- Median: 22% faster
- AVG: 16% faster
- p90: 16% faster

![p90 response time comparison](https://dev.37signals.com/assets/images/yjit-is-fast/bc4.png)

p90 response time comparison

### Upgrade to Ruby 3.3.0

If you’re considering testing YJIT, I strongly suggest you upgrade to Ruby 3.3.0 as soon as possible. That’s where we achieved our biggest improvements. Indeed after the last Ruby upgrade we increased our YJIT ratio dramatically: From ~43% to ~98%. This means that 98% of our code is executed by YJIT instead of the (slower) Ruby interpreter, that’s a massive change. Below the graph of our current `ratio_in_yjit`:

![p90 response time comparison](https://dev.37signals.com/assets/images/yjit-is-fast/ratio_in_yjit.png)

p90 response time comparison

---

## Conclusion

Thanks to YJIT we achieved massive performance improvements across the board, with zero code changes at a cost of a little memory overhead. If you haven’t done it yet you should try it now: It’s just a matter of time and it will be [enabled by default in Rails](https://github.com/rails/rails/pull/49947).
