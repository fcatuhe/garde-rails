# Fractal journeys

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** September 18, 2022
**Series:** Code I like
**Source:** <https://dev.37signals.com/fractal-journeys/>

*Good code is a fractal: you observe the same qualities repeated at different levels of abstraction.*

---

![](https://dev.37signals.com/assets/images/fractal-journeys/cover.jpg)

Fractal is about similar patterns recurring at progressively smaller scales. To me, good code is a fractal: you observe the same qualities repeated at different levels of abstraction.

This is not surprising. Good code is the one that is easy to understand, and the best mechanism we have to deal with complexity is building abstractions. These interchange complexity for an intelligible interface to us humans. But we still need to deal with that complexity they push down; to do that, we follow the same process all over: we build new abstractions that hide details and offer a higher-level mechanism to deal with those.

I am using *abstraction* to refer to everything: from a large subsystem to the last private method in some internal class. But how do you build those abstractions? Well, that’s the million-dollar question and the subject of countless books. In this post, I would like to focus on four qualities I consider essential when it comes to making code understandable:
- Domain-Driven: speak the domain of the problem.
- Encapsulation: expose crystal clear interfaces and hide details.
- Cohesiveness: do one single thing from the point of view of their caller.
- Symmetry: operate at the same level of abstraction.

Because this post is getting too, pardon me, abstract, I’ll clarify with some real-world code from [Basecamp](https://basecamp.com). In several places, the product offers an [activity timeline](https://3.basecamp-help.com/article/92-the-latest-activity). This timeline refreshes dynamically: it will update in real-time if someone does something while you are looking at it.

At the domain level, when you perform actions in Basecamp, such as completing todos, creating documents, or posting comments, the system creates *events*, and those events are *relayed* to several destinations, such as the activity timeline or webhooks. Let’s have a look at the code:

First, we have the `Event` model, which includes a `Relaying` concern (I am just showing the relevant parts):

```
class Event < ApplicationRecord
  include Relaying
end
```

This concern adds an association `relays` and a hook to relay events asynchronously when they are created:

```
module Event::Relaying
  extend ActiveSupport::Concern

  included do
    after_create_commit :relay_later, if: :relaying?
    has_many :relays
  end

  def relay_later
    Event::RelayJob.perform_later(self)
  end

  def relay_now
    …
  end
end

class Event::RelayJob < ApplicationJob
  def perform(event)
    event.relay_now
  end
end
```

So `Event#relay_now` is the method we are interested in. Notice that it speaks the domain language; it does one thing from the point of view of the job that invokes it; and that everything involved in relaying an event is hidden at this point. Let’s dig into that method:

```
module Event::Relaying
    def relay_now
        relay_to_or_revoke_from_timeline

        relay_to_webhooks_later
        relay_to_customer_tracking_later

        if recording
          relay_to_readers
          relay_to_appearants
          relay_to_recipients
          relay_to_schedule
        end
      end
end
```

This method orchestrates the invocations to a set of lower-level methods. They are all about relaying, so cohesiveness remains; they have clear names for the relay destinations based on the domain; details are still hidden; and they are symmetric: you don’t have to jump across levels of abstraction to understand what this method does.

That method `#relay_to_or_revoke_from_timeline` looks like the one we are looking for:

```
module Event::Relaying
  private
    def relay_to_or_revoke_from_timeline
      if bucket.timelined?
        ::Timeline::Relayer.new(self).relay
        ::Timeline::Revoker.new(self).revoke
      end
    end
end
```

Again, good domain-based names: it checks if a bucket is *timelined* and creates an object `Timeline::Relayer` to *relay* events to a timeline; notice the symmetry: there is a counterpart class to *revoke* events; the method is cohesive, it focuses on relays and timelines, and implementation details remain hidden. Let’s look into that class:

```
class Timeline::Relayer
  def initialize(event)
    @event = event
  end

  def relay
    if relaying?
      record
      broadcast
    end
  end

  private
    attr_reader :event
    delegate :bucket, to: :event

    def record
      bucket.record Relay.new(event: event), parent: timeline_recording, visible_to_clients: visible_to_clients?
    end

    def broadcast
      TimelineChannel.broadcast_event(event, to: recipients)
    end
end
```

This time the abstraction is a plain Ruby class, not a method, but we can observe the same traits. It exposes a public method `#relay` that hides its implementation details. Looking inside, we see it does two operations: record the relay in the database and broadcast it via Action Cable (this code was written years before [Hotwire](https://hotwired.dev)). Notice the symmetry: even when both operations are a single-line invocation, they are extracted out as higher-level methods.

Finally, we reach the low-level details. The method `#record` persists the relay in the database — relays are recordables for a recording, the seminal use case that originated [Rails’ delegated types](https://github.com/rails/rails/pull/39341). And `#broadcast` is the method where the event is broadcasted to recipients, the one we were interested in when we started.

In this example, we could easily understand the relaying logic from the moment an event is created until it is pushed through the action cable channel. We could do that because there is only one thing to pay attention to on each jump: one responsibility and one level of abstraction, with names reflecting the problem we are reasoning about. Of course, what constitutes good code is subjective and involves many more concepts, but the ability to make these journeys with ease on non-trivial systems is the number one quality in the code I like.
- [Other posts in the “Code I like” series](/series/code-i-like/)
- Photo by [Martin Rancourt](https://unsplash.com/@vonziper?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/fractal?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
- Translations: [Chinese](https://xfyuan.github.io/2023/03/fractal-journeys), [Japanese(zenn)](https://zenn.dev/tkebt/articles/4a42662163db92), [Japanese(techracho)](https://techracho.bpsinc.jp/hachi8833/2023_03_17/126697), [Korean](https://velog.io/@heka1024/Code-I-like-II-Fractal-journeys)
