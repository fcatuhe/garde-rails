# Pending tests

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** March 1, 2023
**Source:** <https://dev.37signals.com/pending-tests/>

*I rarely write my tests first or use them to help design my code.*

---

I recently started working on a new thing at 37signals. We have a blank slate in front of us, and nothing is set in stone, which means we are moving fast. I find myself creating meaty pull requests every day. I want to open them early to inform discussions, and they all include this:

![Checkbox showing 'pending tests' from a Pull Request](https://dev.37signals.com/assets/images/pending-tests/pending-tests.png)

This is just a reflection of how, most of the time, I write tests at the end. An exception is when a test offers me the shortest feedback loop, which, in my experience, happens more frequently in infrastructure work than in product work. But even then, I don’t look for tests to help me design systems. I don’t practice TDD (Test-driven development).

I am well-versed in TDD and what it has to offer. I fully embraced the paradigm when it exploded back in the day and I stopped using it at some point, first with a sense of guilt, then [with a sense of relief](https://dhh.dk/2014/tdd-is-dead-long-live-testing.html).

There is something I like about TDD: it encourages observing a system from the outside, as a black box that hides complexity and offers an intelligible interface. As I’ve written in the past, I think [this is a key design principle you need to apply at every level of abstraction](https://dev.37signals.com/fractal-journeys), from outside your app boundaries until the last internal method it contains. I don’t think you need TDD to do this at all, but thinking of interfaces from their consumer’s point of view is positive.

On the negative side, TDD encourages a testing style I am very wary of: building very small and fast tests by mocking slow dependencies out. In my experience, this testing approach has a bad cost/benefit: it’s both expensive and ineffective when it comes to [reaching a given level of confidence](https://stackoverflow.com/questions/153234/how-deep-are-your-unit-tests/153565#153565). Years ago, I wrote [some thoughts about my testing preferences](https://www.jorgemanrubia.com/2018/05/19/on-rails-testing/). I can sum them up as *test the real thing* as much as possible.

Another danger of TDD is the companion belief that the emphasis on building blocks that can be tested independently produces well-designed systems. I can buy that a well-designed system must be – to some extent — testable, but I don’t think the opposite is true or that every single part must be testable in isolation. You can build a terrible design out of perfectly testable units with a thousand tests that executes in less than a second. Then, there is the concern of making every module testable without dependencies, which comes with a tax to pay in terms of making those injectable and indirection. I believe you can build great or terrible designs with or without TDD; I just don’t see the solid cause-effect relationship many defend here.

But more than anything, what I like the least about TDD is the lack of pragmatism you often see associated with it. It’s rarely presented as a tool to use under certain circumstances but as a design technique that should drive how you build software. This letter of introduction, combined with the ceremony TDD brings, is a magnet for dogmatism and for all the negative things that come with it.

TDD is a topic that produces strong reactions from both practitioners and detractors. I never do TDD, but I know it works well for many. As long as it doesn’t smell like dogma or is used as a weapon to attack others’ professionalism, I have no problems with it. I just see it as a tool I don’t use.
- Translations: [Chinese](https://xfyuan.github.io/2023/03/pending-tests), [Japanese(techracho)](https://techracho.bpsinc.jp/hachi8833/2023_05_29/130053)
