# The 10x development environment

**Author:** Alberto Fernández-Capel, Programmer
**Published:** November 29, 2022
**Source:** <https://dev.37signals.com/the-10x-development-environment/>

*My hunch is that if anything can make you 10 times more productive, it’s the environment not the programmer.*

---

There’s a myth in programming — originating from the aptly named [Mythical Man Month](https://en.wikipedia.org/wiki/The_Mythical_Man-Month) book — that says that a top-notch programmer is 10 times more productive than the average programmer. I don’t know if that’s true, or even if it was how you’d measure it. But in all the years I’ve been programming I’ve noticed that my working environment has a huge impact on how productive I can be. My hunch is that if anything can make you 10 times more productive, it’s the environment not the programmer.

And of all the places I’ve worked at, 37signals is, hands down, the one where it’s been easier, and more enjoyable, to get things done.

---

## The Card Table

As an example, we recently launched a new feature in [Basecamp 4](https://basecamp.com) called the Card Table. It’s our own take on kanban, with a few enhancements that make it special. It includes all the things you’d expect from a kanban board, like creating columns and cards, assigning cards, and setting due dates. Plus there are a few novel features of our own invention (if curious, you can see the Card Table in action [here](https://www.youtube.com/watch?v=wUL6MpR8eMI)). And everything has real time updates.

As a feature, it’s quite a lot: there are entire products that are less ambitious than this single Card Table feature.

![Card Table](https://dev.37signals.com/assets/images/the-10x-development-environment/card-table.png)

---

## Two people and six weeks to version 1.0

Now, something remarkable about the Card Table is that it only took two people and a 6-week cycle to build the first version, and only one more cycle to prep for public release. During the first cycle it was only me on programming, and my colleague [Dorin](https://www.dorinvancea.com/) on design. By the end of the cycle we had a shippable v1 that everyone in the company could use.

![The timeline of Card Table development](https://dev.37signals.com/assets/images/the-10x-development-environment/card-table-timeline.png)

The timeline of Card Table development

After rolling out the Card Table internally, we spent the next six weeks using it in earnest, to see how it worked and felt in practice. During this time we mostly worked on other things, as well as being on holidays! This took place during the summer when [we only work 4 day weeks](https://basecamp.com/handbook/06-benefits-and-perks#summer-hours).

Finally, we spent one last cycle to polish things up and build high-fidelity mobile versions for Android and iOS, each of which were also built by a single programmer, over a six week cycle.

And that’s it! Very little time, and very few people for such a big project. But it’s not because we are 10x programmers and designers. There are no supernatural abilities at play. This kind of productivity wouldn’t be possible if it wasn’t for the way we work at 37signals.

---

## Teams of two

In the first place, a lot of the productivity comes precisely from how small the team is. Back in 2006, [37signals was already advocating for teams of three](https://basecamp.com/gettingreal/03.3-the-three-musketeers) as the perfect team size. But since then we’ve moved to having teams of just two.

A team of two people might seem tiny, but being tiny and nimble also has lots of advantages. For one, there’s little communication overhead. When there are only two people in the team it’s very easy to keep everyone in the loop and know what everyone else is up to.

And there’s very little management required. 37signals has always hired [managers of one](https://signalvnoise.com/posts/1430-hire-managers-of-one), people that are self-driven and capable of organizing themselves. This greatly simplifies everything because it ensures that decisions about the work are taken by the people closer to the actual work and with a better sense of it.

---

## Time constraints

The same way that having a team limited in size is a constraint but also a feature, being time constrained and [having only six weeks](https://basecamp.com/shapeup/2.2-chapter-08#six-week-cycles) to deliver a project also has lots of benefits. If we hadn’t had a 6-week constraint, we would have never got a complete v1 after only 6 weeks.

When you have very little time, you have to make hard decisions about what’s important and what’s accessory and focus only on what really matters. [Everything else can wait](https://basecamp.com/shapeup/1.2-chapter-03#fixed-time-variable-scope).

![Card Perma](https://dev.37signals.com/assets/images/the-10x-development-environment/card-perma.png)

This also makes the end product better because it acts as a counterbalance to the natural tendency that people who love building things have towards building new things, just for the sake of building them. It’s the perfect motivator to focus on simplicity and avoid feature creep.

---

## Uninterrupted time

We also have long stretches of uninterrupted time, with very few meetings in between. In my normal work week at 37signals I only have a one hour recurring meeting, a web programmers call that we have on Wednesdays. This is a chance to catch up with other programmers in the team. Half social, half business. Other than that, I have a 1-1 every other week and the occasional All-Hands company meeting. But that’s it! All the other communications happen on Basecamp and are [asynchronous by default](https://37signals.com/how-we-communicate). There’s rarely anything so urgent that cannot wait if you’re focused on programming.

![My schedule](https://dev.37signals.com/assets/images/the-10x-development-environment/schedule.png)

---

## A rich domain model

Another thing that helps a lot with productivity is the delightful Basecamp domain model. It’s all the more remarkable because the Basecamp code base was started in 2014. It’s now almost 9 years old and with 500 models and 400 controllers, it’s quite big. Normally you’d expect such an old and large code base to be a nightmare, the Dreadful Legacy Code Base. A legacy code base is supposed to be a hindrance. Yet in this case it was the opposite: the existing code base helped us enormously to get the feature done. If the Card Table had been a greenfield project it would have taken us much more time to finish it, not less.

If you’re interested in the details, we have written at length about how we structure our domain model in [this article](https://dev.37signals.com/vanilla-rails-is-plenty).

But here’s a brief recap:

The Basecamp domain model is based on the concept that originally inspired [ActiveRecord delegated types](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html). It has a base model, the `Recording`, that holds the record metadata (such as its id or timestamps), and multiple delegated types (`Recordable`s) that store the record content. In the Basecamp code base most content, from comments to todos, to chat lines, are `Recordable`s.

The operations that you can perform on a Recording are implemented as concerns, and usually included in the base `Recording` model, so every `Recording` type can use them if they need.

Here is everything included in `Recording`:

```
class Recording < ApplicationRecord
  include Tree, Status
  include Readable, Bookmarkable, Incineratable # Depends on Status
  include AdditionSubscribable, Attachables, Boostable, Cancelable,
          Recording::Categories, Chartable, ClientVisibility, Colored, Commentable,
          Copyable, Dropboxed, Exportable, Indexed, InternallyAccessible, HillChartable,
          Kanbanable, Lockable, Mentions, Movable, Notify, Pinnable, Preloadable,
          Printable, Publishable, Recordables, Replyable, Respondable, Subscribable,
          Threadable, Visibility, TrackStorage, TimelineOrdered
  include Invalidation, Remindable, Pausable, Participatable, Publishing, Categorizable,
          OrderedChildren, PositionedChildren, Recurrable, Repeatable
  include Eventable # Depends on Recordables
  include Assignable # Depends on Subscribable
  include Positioned # Depends on Status, PositionedChildren
  include Dockable # Depends on Status
  include Completable # Depends on Positioned
  include PinnedChildren # Depends on Pinnable
end
```

`Completable`, for example, allows you to mark a `Recording` as completed, and has operations to complete or incomplete a `Recording`, no matter whether it’s an existing `Todo` model or a new `Card` model in a new kanban feature.

Remarkably, by being a `Recording` our new kanban `Card` model can already use, without any modification, all of these concerns. It can be assigned, it can be completed, it can be sorted. It can include mentions, and those mentions will automatically notify the mentioned person.

---

## Hotwire

On the client side the story is slightly different. As I mentioned earlier the Basecamp code base is almost 9 years old. In those 9 years the front-end landscape has changed a lot and you can see that in the code. For example, some of the older parts in Basecamp are still implemented in jQuery. They work fine and there’s little practical reason to rewrite these areas (that’s another thing about productivity at 37signals: we only focus on things that matter). The old JS code is not bad code, but it’s not the kind of code you’d write in 2022, and because of that we didn’t reuse much of it for the Card Table.

But we still had another code abstraction that helped us enormously: [Hotwire](https://hotwired.dev/). The Card Table has many complex interactions, often including live and partial page updates. For example, for large Card Tables we need pagination, and lazy loading cards from the server when they come close to the viewport. But Hotwire made all this remarkably easy to build, all without needing any custom JS code.

Probably the only exception was the drag and drop interaction, to drag cards to another column or reorder columns. But in this case we only needed a touch of JS sprinkles in the form of a few Stimulus controllers. And even those controllers only amount to about 200 lines of code.

It’s amazing how far you can go and what you can build today with Hotwire and a splash of JS sprinkles.

---

## Conclusion

In our culture we seem to be obsessed with productivity tips. We want to know about this secret productivity hack, this little detail that you can easily change in your morning routine and suddenly be 2x more productive. Bonus points if you can stack it up with other improbable productivity gains and be 4x, 8x, 16x… more productive.

In the programming world this manifests as endless discussions about workflow improvements: which editor — vim or emacs — or which keyboard layout or key bindings allow you to type faster, or how to make your tests run faster, or about the nuances of a linter rule. These are all fine and useful — I like nuanced discussions more than anyone, and I’ll take any marginal improvement that I can get! But the big productivity gains only come from getting the fundamentals right.

To do good work you need to have time to focus on the work itself. You need to have a solid technical foundation. You need autonomy. You need to make hard decisions about what’s important and what’s accessory. That’s what makes the big difference and can increase your productivity manifold.
