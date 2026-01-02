# Globals, callbacks and other sacrileges

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** July 31, 2023
**Series:** Code I like
**Source:** <https://dev.37signals.com/globals-callbacks-and-other-sacrileges/>

*Maximalist positions are a thing in our industry. Take a technique, outline its drawbacks, extrapolate you can’t use it under any circumstance, and ban it forever. We are lucky that Rails embraces exactly the opposite mindset as one of its pillars.*

---

[Sacrificing purity for convenience is one of the Rails pillars](https://rubyonrails.org/doctrine#no-one-paradigm). This principle informs several Rails features that some recommend avoiding but that we happily use in our apps. In this post, I’ll discuss how we use three of those in [Basecamp](https://basecamp.com).

I’ll show some examples involving projects. In Basecamp, a `Bucket` represents a top-level container of data, and there are several types of buckets, one of which is a `Project`. Internally, we implement this with [delegated type](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) attribute, where a `Bucket` has an associated `Bucketable`. A `Bucket` can contain many `Events`, and an `Event` is a holder for information on the related request, such as the IP or the transaction id (`Event::Request`) and some additional information (`Event::Detail`).

![](https://dev.37signals.com/assets/images/globals-callbacks-and-other-sacrileges/buckets-projects-and-events-in-basecamp.png)

Model for buckets, projects and events

This is a class diagram showing the following models:
- `Bucket` aggregates a `Bucketable` as a [delegated type](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) and many `Events`.
- `Project` is a `Bucketable` and belongs to a `Person` creator.
- `Event` aggregates a `Event::Request`, a `Event::Detail` and belongs to its `Person` creator.

---

## Callbacks

A project exists associated with the bucket that contains it. This is the relevant code that creates projects in Basecamp’s `ProjectsController`:

```
class ProjectsController < ApplicationController
  def create
     @template ? create_from_template : create_without_template
  end

  private
     def create_without_template
      @project = Current.account.projects.create! create_project_params

      # ...
     end
end
```

As you can see, the controller just creates a `Project` record. There is no reference to the associated bucket or anything related to tracking events or the project’s creator.

Let’s start with the `Bucket`. How is created? A [callback](https://guides.rubyonrails.org/active_record_callbacks.html) in the `Bucketable` concern takes care of that:

```
class Project < ApplicationRecord
  include Bucketable
end

module Bucketable
  extend ActiveSupport::Concern

  included do
     after_create { create_bucket! account: account unless bucket.present? }
  end
end
```

This shows a common scenario for callbacks: hook additional bits of logic into the lifecycle of objects. I will show a more juicy example later, but this is good enough to discuss the pros and cons.

A common critique of callbacks is that they bring indirection, making code difficult to follow. And, indeed, you don’t want to orchestrate complex flows using callbacks. But, here, the operation is quite simple – create a bucket along its project if it does not exist – and this is not a primary `Project` responsibility either, so the indirection is a good fit: you’re plugging in a secondary function in a declarative way. Callbacks work great in such scenarios.

This approach lets us create any *buckletable* without caring about its companion bucket. Assuming you don’t want to place such responsibility on the caller side, the alternative here would be to encapsulate this logic in some factory. Let me show you the contrast.

In Basecamp, elements such as todos, messages, or calendar events are recordings. A `Recording` hosts a [delegated type](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) relationship with the specific underlying entity (`Todo`, `Message`, etc.). Some day we’ll explain this pattern in detail, but – for the purpose of this article – it happens that creating recordings is a more involved operation. In this case, we use a factory because the complexity justifies the benefits of a class encapsulating the details away. We expose that factory through `Bucket`, which is a container for recordings:

```
class Bucket < ApplicationRecord
  def record(...)
     Recorder.new(self).record(...)
  end
end

class Bucket::Recorder
  def initialize(bucket)
     @bucket = bucket
  end

  def record(recordable, children: nil, parent: nil, position: nil, scheduled_posting_at: nil, **options)
     # ...
  end
end
```

We create all the recordings in Basecamp through this factory. For example, this is the controller that creates messages in Basecamp:

```
class MessagesController < ApplicationController
  def create
    @recording = @bucket.record new_message,
      parent: @parent_recording,
      category: find_category,
      subscribers: find_subscribers,
      status: status_param,
      visible_to_clients: visible_to_clients_param,
      scheduled_posting_at: scheduled_posting_at_param
    # ...
  end
end
```

While the factory is justified, the tradeoff is that we now have to use it whenever we want to create a recording. It’s a new dependency and element you need to know of. It makes things a bit more complex. Using a callback for creating buckets, we can create projects seamlessly as regular records. Different choices with different tradeoffs for different scenarios.

---

## CurrentAttributes

When Rails introduced [a thread-safe attributes singleton](https://api.rubyonrails.org/classes/ActiveSupport/CurrentAttributes.html) – another extraction from [Basecamp](https://basecamp.com) – there was no shortage of detractors. Sharing global variables between controllers and models? Seriously? Seriously.

`CurrentAttributes` offered an alternative to the `#current_user` method most apps used to access the currently authenticated user. You could now do `Current.user`, and use this pattern to access similar request-level attributes uniformly. But let’s focus on the controversial aspect of having models accessing such global attributes.

Our full `Current` class in Basecamp is:

```
class Current < ActiveSupport::CurrentAttributes
  attribute :account, :person
  attribute :http_method, :request_id, :user_agent, :ip_address, :referrer

  delegate :user, :integration, to: :person, allow_nil: true
  delegate :signal_identity, to: :user, allow_nil: true

  resets { Time.zone = nil }

  def person=(person)
     super
     self.account = person.try(:account)
     Time.zone = person.try(:time_zone)
  end
end
```

At the controller level, our authentication system sets `Current.person` as the currently authenticated `Person`. You can access that person as `Current.person` from the execution context of controllers handling authenticated requests.

This is how we associate a project with the authenticated person that creates it in Basecamp.

```
class Project < ApplicationRecord
  belongs_to :creator, class_name: "Person", default: -> { Current.person }
end
```

The alternative here would be to pass the creator in the controller invocation. For example:

```
@project = Current.account.projects.create! create_project_params.merge(creator: Current.user)
```

Here, we prefer the callback form. It captures quite succinctly that, when creating a project, a *creator* `Person` is mandatory and that, if not provided, it will be the currently authenticated person. Notice that the creator of a project doesn’t contain the project, structurally speaking. It’s more like an audit trait unrelated to what’s involved in creating a project, so it’s good to discharge the controller from knowing about it. The declarative callback approach with `Current.user` captures this whole idea in a way that an explicit controller invocation doesn’t.

We use this pattern in many places for tracking who creates records in Basecamp and HEY.

---

## Callbacks + CurrentAttributes

Tracking changes and request-level information for certain domain entities is handy at many levels. For example, to debug issues reported by customers. I’ll show how we track events for buckets next; we use a similar system for recordings in Basecamp and a very similar approach in HEY.

First, at the controller level, we collect several essential request information in `Current`.

```
class ApplicationController < ActionController::Base
  include SetCurrentRequestDetails
end

module SetCurrentRequestDetails
  extend ActiveSupport::Concern

  included do
     before_action do
      Current.http_method = request.method
      Current.request_id = request.uuid
      Current.user_agent = request.user_agent
      Current.ip_address = request.ip
      Current.referrer = request.referrer
     end
  end
end
```

The [concern](https://dev.37signals.com/good-concerns/) `Bucket::Eventable` contains the relevant code for handling events. It defines the relationship with `Event`, and uses an `after_create` callback to create an event when a new `Bucket` is created. Notice how the `creator:` param in `#track_event` defaults to `Current.person`.

```
class Bucket < ApplicationRecord
  include Eventable
end

module Bucket::Eventable
  extend ActiveSupport::Concern

  included do
     has_many :events, dependent: :destroy

     after_create :track_created
  end

  def track_event(action, creator: Current.person, **particulars)
     Event.create! bucket: self, creator: creator, action: action, detail: Event::Detail.new(particulars)
  end

  private
    def track_created
      track_event :created
    end
end
```

Creating an `Event` will auto-build an `Event::Request` that will, finally, grab the request details from `Current`. Here is the relevant code:

```
class Event < ApplicationRecord
  include Requested
end

module Event::Requested
  extend ActiveSupport::Concern

  included do
     has_one :request, dependent: :delete, required: true
     before_validation :build_request, on: :create
  end
end

class Event::Request < ApplicationRecord
  belongs_to :event
  before_create :set_from_current

  private
     def set_from_current
      self.guid ||= Current.request_id
      self.user_agent ||= Current.user_agent
      self.ip_address ||= Current.ip_address
      self.email_message_id ||= Current.email_message_id
     end
end
```

On the tactical side, the combination of callbacks with `CurrentAttributes` achieves something powerful: you can create a project normally, and the operation gets seamlessly audited.

```
@project = Current.account.projects.create! create_project_params
```

Think about the alternative code where you must carry the request information from the controller layer to `Event::Request`, the last internal piece of the auditing system. You would need, again, a place to contain the creation logic, like a service or factory, and you would need to prepare all the parts to push down the request information. The resulting code could look something like:

```
class ProjectsController < ApplicationController
  def create
     ProjectRecorder.new(account).record(create_project_params, Current.person, request)
  end
end

class ProjectRecorder
  def initialize(account)
     @account = account
  end

  def record(params, creator, request)
     Project.transaction do
      @account.projects.create!(params, request).tap do |project|
       project.bucket.events.create! creator: creator, action: :create, request: Event::Request.create_from(request)
      end
     end
  end
end
```

I think this approach is worse, and for reasons that go beyond the mere aesthetics of it. An auditing concern is orthogonal to creating projects. Even if you need one, a factory that creates projects should care about about how to compose the structural parts of those. Auditing doesn’t sound like a factory responsibility.

By mixing an orthogonal concern into the main path, you are coupling parts that don’t belong together. This is a scenario where indirection actually helps. And callbacks provide precisely an answer to this need.

Back in the day, a paradigm called [Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) (AoP) gained some traction in academic circles. Rails’ callbacks offer pragmatic support for AoP ideas. If you want to see another example, check [this video by DHH about how Basecamp uses callbacks to detect message mentions](https://www.youtube.com/watch?v=m1jOWu7woKM). Same idea: extracting mentions is orthogonal to posting messages; callbacks offer direct support for expressing this without polluting the primary responsibilities of `Message` as a domain entity.

---

## Suppress

Rails introduced [support for suppressing ActiveRecord save operations in arbitrary blocks](https://api.rubyonrails.org/classes/ActiveRecord/Suppressor.html). As you would expect, the idea of creating some local context that hijacked persistence meant to be used with callbacks got  *some*  pushback.

Here’s an example where we use this mechanism. In Basecamp, you can copy any element to a new destination. Internally, this complex system involves several pieces, including a `Recording::Copier` class in charge of copying trees of recordings. When copying a recording, we don’t want the regular event tracking system to trigger, so we suppress it:

```
class Recording::Copier
  # ...
  private
     def copy_recording
      Event.suppress do
       @destination_recording = destination_bucket.record(source_recording.recordable, parent: destination_parent, **copyable_attributes)
      end
     end
end
```

The key to using `.suppress` is exceptionality. You normally want the default behavior to kick in – tracking events in this case. Exceptionally, you want to suppress it. This mechanism allows you to do it without altering the original system to accommodate the conditional logic.

This aligns greatly with callbacks-powered systems that implement orthogonal concerns. As discussed, indirection is a feature for those. `.suppress` represents a succinct mechanism to introduce conditionality while keeping that indirection in place.

---

## Conclusion

There is a strong subjectivity component in these discussions. The same code can look fantastic to some and terrible to others. But here’s something objective: we use the discussed techniques in our applications, which [we actively maintain](https://updates.37signals.com) while millions of people use them.

Maximalist positions are a thing in our industry. Take a technique, outline its drawbacks, extrapolate that you can’t use it under any circumstance, and ban it forever. We are lucky that Rails embraces [exactly the opposite mindset](https://rubyonrails.org/doctrine#provide-sharp-knives) as one if its pillars.

And I think we are lucky because software development is so complex that it benefits from nuance. When balancing convenience and purity, there is a rigidity threshold that causes more harm than good. The advice of the kind *never-ever do this, always do that* should put you on alert. Software development is a game of tradeoffs, and any choice you make comes with them. You should at least understand the tradeoffs you are accepting when you reject techniques on behalf of paradigm purity.

There is no question that callbacks, `CurrentAttributes` and `.suppress` are [sharp knives](https://rubyonrails.org/doctrine#provide-sharp-knives). You can put yourself in a deep hole if you misuse or abuse them. But, in our experience, there are scenarios where they are [the best alternative at hand](https://dev.37signals.com/compared-to-what/), and they don’t cause any maintenance burden. So please, consider this post as a counterweight whenever you hear that using one of these Rails features will make hell break loose.
- [Other posts in the “Code I like” series](/series/code-i-like/)
