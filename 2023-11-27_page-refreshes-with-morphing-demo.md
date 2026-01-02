# Demo of page refreshes with morphing

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** November 27, 2023
**Source:** <https://dev.37signals.com/page-refreshes-with-morphing-demo/>

*How page refreshes work, and how they compare to stream actions.*

---

We published a demo showing how Page Refreshes with morphing work in Turbo 8:

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/page-refreshes-with-morphing-demo/page-refreshes-with-morphing.mp4)

Comparing code [helps a lot in software discussions](https://dev.37signals.com/compared-to-what/), so I thought it would be valuable to show how the new feature compares to Turbo stream actions for performing partial updates and broadcasts. Notice that page refreshes don’t deprecate stream actions — which remain Turbo’s most responsive mechanism, but they should reduce the need to use those. And this is a good thing because stream actions are costly.

There is a [companion GitHub repository for the demo](https://github.com/basecamp/turbo-8-morphing-demo) you can check:
- [The demo app code using vanilla Rails](https://github.com/basecamp/turbo-8-morphing-demo/pull/3)
- [Modification using stream actions to enhance and broadcast updating tasks](https://github.com/basecamp/turbo-8-morphing-demo/pull/6)
- [Modification using page refreshes to enhance and broadcast all the interactions](https://github.com/basecamp/turbo-8-morphing-demo/pull/4)

Last week, we released [the first beta](https://github.com/hotwired/turbo/releases/tag/v8.0.0-beta1) of Turbo 8 featuring this new system. We want to make this robust, so please give it a try and report issues away.

You can learn more about the new feature in the [announcement post](https://dev.37signals.com/a-happier-happy-path-in-turbo-with-morphing/). We will document the new system in the official docs soon.
