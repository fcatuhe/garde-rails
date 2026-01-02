# The radiating programmer

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** November 19, 2023
**Source:** <https://dev.37signals.com/the-radiating-programmer/>

*The right ceremony can save you from the wrong one.*

---

![](https://dev.37signals.com/assets/images/the-radiating-programmer/cover.jpg)

You are an individual contributor at heart. You like writing code and solving technical problems. You dislike meetings and ceremony. Here’s what you can do to maximize what you like and minimize what you don’t: *radiate information*.

The daily standup meetings that Scrum popularized have bad press for a reason. The daily periodicity is way too much. And using a recurring meeting to give people a chance to catch up is atrociously disruptive and inefficient. But the questions themselves make sense: What did you do? What are you going to do? Are there any blockers? You should aim to communicate that information routinely, just not in a meeting.

When building software, everyone involved must adjust what they do as they learn about the problem. And that learning happens — in no small part — through the people doing the job. If they only offer their output, someone else needs to *extract* those lessons to make adjustments. That’s precisely when the bad kind of [ceremony](https://world.hey.com/jorge/ceremony-8f47e582) — the intrusive, disruptive, and lousy one — leaks into the process.

It’s indeed by comparison that *radiating information* shines: instead of having *someone* *pulling information* from you, you *push* the information out there for *everyone*. It might look subtle, but there is a significant difference: the control remains on your side, not anyone else’s.

One can argue that whenever you communicate, you radiate information. But I am talking about something more intentional here. In my daily job, I find three recurring scenarios:
- Communicate what you have been up to periodically.
- Communicate progress on projects.
- When making decisions, to give others a chance to intervene without being blocked.

Next, I will show examples from our own [Basecamp](https://basecamp.com). The underlying idea is not tied to any tool, but if you are familiar with [37signals’ communication philosophy](https://37signals.com/how-we-communicate/), you won’t find it surprising that Basecamp comes with first-class support for it.

---

## What have you worked on?

This is the backbone. We use a [check-in](https://basecamp.com/features/automatic-check-ins) so that everyone can answer “What have you worked on?” at least twice per week. [I wrote about this question some time ago](https://world.hey.com/jorge/what-did-you-work-on-today-b153fba3). You answer these asynchronously, whenever you want, with the style and level of detail you prefer, and everyone in the company can read everyone else’s answers.

Here’s the last answer I wrote:

![](https://dev.37signals.com/assets/images/the-radiating-programmer/what-have-you-worked-on.png)

Answer to “What did you work on?”

This is an screenshot from Basecamp 4 showing the following contents. Notice that the links have been removed. There are three sections highlighted in the text that are later referenced in the article.

Last two:

**Turbo 8**

On Thursday, when I was doing the last testing round of Turbo 8 with HEY before shipping it, I found an issue I hadn’t noticed before: when paginating new periods and creating events in those, the page wasn’t refreshing fine. There were two issues there.

One was easy to spot: we were deleting the pagination frames. Using data-turbo-permanent with those created other issues in the Day screen, so this wasn’t easy to solve. That was the nth reason to recover the original idea of flagging such frames with an attribute, so after talking to Alberto, we recovered that here (this was actually part of the API that we originally presented in Rails World, but at some point we thought we could simplify things further).

**Section 1:** But the second problem took me most of my Thursday to troubleshoot: idiomorph was failing to match the paginated turbo-frames properly. It took me a long debugging session within idiomorph to understand what was going on there, as the error didn’t make sense at all. There were two issues there: one, idiomorph creates a map of nodes and children ids first; this wasn’t playing well with a pagination trick we were doing with JS, where we were moving a turbo frame outside of its container. And second, we had some duplicated “ids” inside our markup that was confusing idiomorph’s heuristics, which are also based on that map.

I fixed the problem by reworking the pagination approach. I am happy with the change as the new system is simpler than the previous one. With that, I could finally wrap up the pull request and ship 🥳.

Also, discussion about an issue that Jay found: we shouldn’t use morphing in regular page visits. Until we fix for good, I enabled morphing only for the two main views, to prevent the issue from happening with the mobile app.

**Mobile API**
- **Section 2:** Addressed feedback in the PR and shipped support for repeating events. Thanks to a question by Jeffrey I found a tricky issue with database precision and URL serialization I fixed here.
- Chat about including modified records in responses. I should complete the short list of requests we need to support this Monday.

**Other**
- Weighted in on:
- Control to jump to specific days.
- **Section 3:** Will you still be able to move calendars between accounts after creation? We’re not supporting that for launch 🪓.
- Some chatting about bugs and organizing those. Next week I’ll dedicate a good amount of time to clear some serious bugs we have.
- Some thinking about next projects for the web team. Alberto has already started with support for multiple home accounts, and control to jump to specific days will be next.
- PR reviews:
- For Matt who built a nice foundation for supporting the rich iCal invitations workflow and integrate it with HEY.
- For Alberto: one, two
- Wrote some hill chart updates for the projects I worked on this week:
- Mobile API
- Open source: Upstream turbo-morph
- Performance

I have highlighted three sections to illustrate how others could benefit from the information I shared:
- An unexpected problem that affected what I could accomplish this week.
- Something I learned that other programmers might find useful.
- Some [scope hammering](https://basecamp.com/shapeup/3.5-chapter-14#scope-hammering) that could be interesting to folks making product calls.

---

## Communicating progress on projects

Tools can help, but it is very hard to get a high-level picture of how a project is progressing unless another human — preferably the one doing the job — helps. And no, I don’t think AI will change that anytime soon.

Our preferred way to show progress in Basecamp is through [Hill Charts](https://basecamp.com/features/hill-charts) updates. They combine a high-level completion gauge with a textual description. Here’s the last one I wrote:

![](https://dev.37signals.com/assets/images/the-radiating-programmer/hill-chart.png)

Answer to “Hill chart update”

An screenshot from a Basecamp 4 Hill Chart update with the following contents, and with 3 sections highlighted:

**Section 1:** This week Alberto cleared all the remaining issues with the Card Table, and Jorge all the remaining ones in HEY. Both translated into tweaks to the upstreamed library. HEY is now using Turbo 8 in production, and the Calendar, finally, is using the upstreamed version with idiomorph.

**Section 2:** Today we recovered a setting we had nixed: the one to flag turbo-frames that you want to reload during page refreshes. At the end, removing the option created a ton of frictions in different scenarios.

**Section 3:** To launch Turbo 8 we still need to document the library, and also to solve this problem that Jay detected: we should not be using morphing during regular page navigations. We also need to complete some changes to the adapters that Jay and Alberto already cooked (that aren’t strictly beneficial to morphing, but without which morphing doesn’t make much sense in mobile apps).

So not much to do to think of a release here. We should probably release a beta first.

**Section 4:** And the good news is that this is totally decoupled from the calendar launch now.

Again, different people in the company can get different things from the update:
- Progress since the last update.
- A problem that resulted in recovering some API we had removed.
- Something we discovered that implies additional work.
- How does this affect, or doesn’t, the Calendar launch?

---

## Non-blocking decisions

I have written about how [you should avoid blocking yourself when working remotely](https://world.hey.com/jorge/don-t-block-yourself-a-remote-worker-super-power-7322c679). When you need input from others to make a decision, you can often make the decision yourself while informing about it. Others can intervene if they want, or you can move forward if they don’t.

Here’s an example from last week. I had a question about the mobile API I wanted to check with the team. I used my best judgement to make a call and looped them in. I was happy to change the approach based on the answer, but I didn’t put myself in a self-blocking situation waiting for it.

![](https://dev.37signals.com/assets/images/the-radiating-programmer/non-blocking-decisions.png)

Collaborating without blocking yourself

A comment from Jorge in Basecamp 4:

> Hey folks,
> For the API responses, I’m seeing that, in general in HEY and BC4, we rely on response codes (e.g: 201 for created, 204 for updated/deleted). We don’t render the response body.
> For the Calendar API, I’m thinking that it might be interesting to return the created/updated resource in the JSON body? For example: if you create an event, you need to grab the id for the future. I’m going with that if you don’t tell me otherwise.

A reply from Milan:

> Hey Jorge 👋
> Generally, I don’t think we’ll need the newly created/updated resources in the responses right now because we’ll just hit the incremental sync endpoint to get the fresh data right after.
> On the other hand, there will probably be situations where we’ll want to reference the newly created object — as you mentioned, at least know what ID it received from the backend. So I don’t think there’s any harm in sending the resources back in responses even if we don’t use them right now.

---

## Conclusion

Information radiation can look like a bureaucratic practice on the surface, but it is actually a protection against that. Think of the alternatives for the examples I referenced:
- Catchup meeting to see what’s left for launching Turbo 8.
- Chat mention to someone in the mobile team to synchronously discuss the API response format.
- 1-1 with a manager to communicate how the stuff we discovered with Turbo affects the Calendar launch.
- Quick call with a designer to decide whether to implement the feature we decided to nix.

Exaggerated? That’s exactly how work looks in a lot of places. If you need a meeting, make sure it’s not one you could have avoided by radiating information. Think of *this meeting could have been an email* but with a process where *emails* flow proactively, by design.

As a programmer, you can add much value by summarizing your work. You can share lessons learned, struggles, unexpected turns, motivations, and, more broadly, how things move forward. No tool or automatic report can do that. Many people can benefit; you can help others and receive help. You favor making decisions based on how reality looks, which is an essential trait for any non-predictable endeavor such as software development. And ultimately, it brings clarity to what you do, which counterweights the intrinsic opacity of spending your days writing code.

Radiating information is not free; it takes time. You need to find a balance that makes sense. I estimate I spend one or two hours per week writing what I did or project updates. I usually leave it for the last part of my day. It doesn’t interrupt my main focused work, and it doesn’t happen every day. It’s also a muscle to train. The more you do it, the easier it gets.

And whenever it feels like a chore, I remind myself that it is a chore that lets me spend most of my time doing what I like.
- Translations: [Chinese](https://xfyuan.github.io/2023/12/the-radiating-programmer)
