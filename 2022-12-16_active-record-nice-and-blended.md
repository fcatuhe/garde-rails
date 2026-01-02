# Active Record, nice and blended

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** December 16, 2022
**Series:** Code I like
**Source:** <https://dev.37signals.com/active-record-nice-and-blended/>

*Active Record restates the traditional question of how to separate persistence from domain logic: what if you don’t have to?*

---

![](https://dev.37signals.com/assets/images/active-record-nice-and-blended/blended.jpg)

Persisting objects in relational databases is an intricate problem. Two decades ago, it looked like the ultimate orthogonal problem to solve: abstract persistence out so that programmers don’t have to care about it. Many years later, we can affirm that… it’s not that simple. Persistence is indeed a crosscutting concern, but one with many tentacles.

Because it can’t be fully abstracted, many patterns aim to isolate database access on its own layer to keep the domain model persistence-free. For example [repositories](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/), [Data Mappers](https://en.wikipedia.org/wiki/Data_mapper_pattern) or [DAOs (Data Access Objects)](https://en.wikipedia.org/wiki/Data_access_object). Rails, however, went with a different approach — [Active Record](https://guides.rubyonrails.org/active_record_basics.html#the-active-record-pattern) — a pattern introduced by Martin Fowler in [Patterns of EAA](https://martinfowler.com/books/eaa.html):

> An object that wraps a row in a database table or view, encapsulates the database access, and adds domain logic on that data.

The distinctive trait of the Active Record pattern is combining domain logic and persistence in the same class, and that’s how we use it here at [37signals](https://37signals.com). At first glance, this might not look like a good idea. Shouldn’t we keep separated things, well, separated? Even Fowler mentions that the pattern is a good choice for *domain logic that isn’t too complex*. It’s our experience, however, that Rails’ Active Record keeps code elegant and maintainable in large and complex codebases. In this article, I will try to articulate why.

---

## An impedance-less match

[Object–relational impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch) is a fancy way of saying that Object-Oriented languages and relational databases are distinct worlds, and this results in friction when translating concepts between them.

I believe Active Record — the Rails framework, not the pattern — works so well in practice because it reduces this impedance mismatch to a minimum. There are two main reasons:
- It looks and feels like Ruby, even when you need to go lower level to fine-tune things.
- It comes with fantastic and innovative answers for recurring needs when dealing with objects and relational persistence.

### A perfect Ruby companion

Let me show you an example from [HEY](https://www.hey.com). It shows the internals from the `Contact#designate_to(box)` method I referenced in the [Vanilla Rails article](https://dev.37signals.com/vanilla-rails-is-plenty/). This method handles the logic when you select a box as the destination for emails from a given contact. I’m highlighting the lines involving Active Record:

```
module Contact::Designatable
  extend ActiveSupport::Concern

  included do
    has_many :designations, class_name: "Box::Designation", dependent: :destroy
  end

  def designate_to(box)
    if box.imbox?
      # Skip designating to Imbox since it’s the default.
      undesignate_from(box.identity.boxes)
    else
      update_or_create_designation_to(box)
    end
  end

  def undesignate_from(box)
    designations.destroy_by box: box
  end

  def designation_within(boxes)
    designations.find_by box: boxes
  end

  def designated?(by:)
    designation_within(by.boxes).present?
  end

  private
    def update_or_create_designation_to(box)
      if designation = designation_within(box.identity.boxes)
        designation.update!(box: box)
      else
        designations.create!(box: box)
      end
    end
end
```

The persistence bits look natural and easy to follow. The code is eloquent and concise, and it reads like Ruby. It doesn’t feel like a mix of concerns that don’t belong together; you don’t see a cognitive jump between “business logic” and “persistence duties”. To me, this trait is a game-changer.

### Answers for persistence needs

Active Record offers many options to persist object-oriented models into tables. When presenting the original pattern, Fowler argues that:

> If your business logic is complex, you’ll soon want to use your object’s direct relationships, collections, inheritance, and so forth. These don’t map easily onto Active Record, and adding them piecemeal gets very messy.

Rails’ Active Record offers answers for those and more. To enum a few: [associations](https://guides.rubyonrails.org/association_basics.html), [single table inheritance](https://api.rubyonrails.org/classes/ActiveRecord/Inheritance.html), [serialized attributes](https://api.rubyonrails.org/v7.0.4/classes/ActiveRecord/AttributeMethods/Serialization/ClassMethods.html) or [delegated types](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html).

Some people recommend avoiding Rails associations at all costs. I have a hard time understanding this one: I think associations are one of the best features of Active Record, and we use them extensively in our apps. When studying object-oriented programming, *associations* between objects are a fundamental construct, just like inheritance. Same with relationships between tables in the relational world. What’s not to like about direct support for translating those into code and getting the framework to do all the heavy lifting for you?

Let me show you an example of associations. In [HEY](https://www.hey.com), an email thread internally looks like a `Topic` model with many entries (`Entry`). In some scenarios, the system needs to access aggregated data at the topic level based on the contained entries, such as the addressed contacts in the thread or the blocked trackers. We implement the bulk of this with associations:

```
class Topic
  include Entries

  #...
end

module Topic::Entries
  extend ActiveSupport::Concern

  included do
    has_many :entries, dependent: :destroy
    has_many :entry_attachments, through: :entries, source: :attachments
    has_many :receipts, through: :entries
    has_many :addressed_contacts, -> { distinct }, through: :entries
    has_many :entry_creators, -> { distinct }, through: :entries, source: :creator
    has_many :blocked_trackers, through: :entries, class_name: "Entry::BlockedTracker"
    has_many :clips, through: :entries
  end

  #...
end
```

We use other Active Record features profusely to get direct persistence support for rich Ruby object models. For example, [single table inheritance](https://api.rubyonrails.org/classes/ActiveRecord/Inheritance.html) to model the different kinds of boxes in HEY:

```
class Box < ApplicationRecord
end

class Box::Imbox < Box
end

class Box::Trailbox < Box
end

class Box::Feedbox < Box
end
```

Or [serialized attributes](https://api.rubyonrails.org/v7.0.4/classes/ActiveRecord/AttributeMethods/Serialization/ClassMethods.html) to store the recurrence schedule details for Basecamp checkins:

```
class Question < ApplicationRecord
  serialize :schedule, RecurrenceSchedule
end
```

And we use [delegated types](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) to model the different kinds of contacts in HEY.

```
class Contact < ApplicationRecord
  include Contactables
end

module Contact::Contactables
  extend ActiveSupport::Concern

  included do
    delegated_type :contactable, types: Contactable::TYPES, inverse_of: :contact, dependent: :destroy
  end
end

module Contactable
  extend ActiveSupport::Concern

  TYPES = %w[ User Extenzion Alias Person Service Tombstone ]

  included do
    has_one :contact, as: :contactable, inverse_of: :contactable, touch: true
    belongs_to :account, default: -> { contact&.account }
  end
end

class User < ApplicationRecord
  include Contactable
end

class Person < ApplicationRecord
  include Contactable
end

class Service < ApplicationRecord
  include Contactable
end
```

The caveat here is that Active Record requires you to control the database schema to leverage what it offers fully. Assuming this is the case, the ability to seamlessly persist rich and complex object models is key to making the pattern work in large and complex codebases.

---

## Encapsulation-friendly

Because it blends so well with Ruby, Active Record plays great with using regular Ruby goodies to hide details away. This allows you to write code that looks natural and encapsulates persistence concerns without paying the ceremony tax of a separate data-access layer.

For example, you can check the previous `Contact::Designatable` code. The Active Record code is wrapped in plain private methods. You can also notice how everything — domain logic and persistence — is hidden behind a method `#designate_to`, which is part of that natural interface we want to see from the system boundaries, as explained in [this article](https://dev.37signals.com/vanilla-rails-is-plenty/). So persistence is mixed in but well organized and encapsulated.

In more complex scenarios, nothing prevents you from creating objects to hide complexity away. For example, in [Basecamp](https://basecamp.com), to render the activity timeline for a given user, it uses a class `Timeline::Aggregator`, which is a PORO in charge of serving the relevant events. This class encapsulates the querying logic:

```
class Reports::Users::ProgressController < ApplicationController
  def show
    @events = Timeline::Aggregator.new(Current.person, filter: current_page_by_creator_filter).events
  end
end

class Timeline::Aggregator
  def initialize(person, filter: nil)
    @person = person
    @filter = filter
  end

  def events
    Event.where(id: event_ids).preload(:recording).reverse_chronologically
  end

  private
    def event_ids
      event_ids_via_optimized_query(1.week.ago) || event_ids_via_optimized_query(3.months.ago) || event_ids_via_regular_query
    end

    # Fetching the most recent recordings optimizes the query enormously for large sets of recordings
    def event_ids_via_optimized_query(created_since)
      limit = extract_limit
      event_ids = filtered_ordered_recordings.where("recordings.created_at >= ?", created_since).pluck("relays.event_id")
      event_ids if event_ids.length >= limit
    end

    def event_ids_via_regular_query
      filtered_ordered_recordings.pluck("relays.event_id")
    end

    # ...
end
```

For querying, we use [scopes](https://guides.rubyonrails.org/active_record_querying.html#scopes) extensively. Combining those with associations and other scopes lets you express complex queries with natural-looking code. For example, for rendering a [collection in HEY](https://www.hey.com/features/collections/), it needs to fetch all the active topics in the collection accessible to the acting contact — in HEY you can have different *acting* users depending on the selected filter. This is how the related code looks:

```
class Topic < ApplicationController
  include Accessible
end

module Topic::Accessible
  extend ActiveSupport::Concern

  included do
    has_many :accesses, dependent: :destroy
    scope :accessible_to, ->(contact) { not_deleted.joins(:accesses).where accesses: { contact: contact } }
  end

  # ...
end

class CollectionsController < ApplicationController
  def show
    @topics = @collection.topics.active.accessible_to(Acting.contact)
    # ...
  end
end
```

While this is a bit of an edge case, you can see how we also used scopes for one of the HEY performance optimizations Donal described in [this article](https://dev.37signals.com/faster-paging-in-hey/):

```
module Posting::Involving
  extend ActiveSupport::Concern

  DEFAULT_INVOLVEMENTS_JOIN = "INNER JOIN `involvements` USE INDEX(index_involvements_on_contact_id_and_topic_id) ON `involvements`.`topic_id` = `postings`.`postable_id`"
  OPTIMIZED_FOR_USER_FILTERING_INVOLVEMENTS_JOIN = "STRAIGHT_JOIN `involvements` USE INDEX(index_involvements_on_account_id_and_topic_id_and_contact_id) ON `involvements`.`topic_id` = `postings`.`postable_id`"

  included do
    scope :involving, ->(contacts, involvements_join: DEFAULT_INVOLVEMENTS_JOIN) do
      where(postable_type: "Topic")
        .joins(involvements_join)
        .where(involvements: { contact_id: Array(contacts).map(&:id) })
        .distinct
    end

    scope :involving_optimized_for_user_filtering, ->(contacts) do
      # STRAIGHT_JOIN ensures that MySQL reads topics before involvements
      involving(contacts, involvements_join: OPTIMIZED_FOR_USER_FILTERING_INVOLVEMENTS_JOIN)
        .use_index(:index_postings_on_user_id_and_postable_and_active_at)
        .joins(:user)
        .where("`users`.`account_id` = `involvements`.`account_id`")
        .select(:id, :active_at)
    end
  end
end
```

---

## A restatement of the persistence and domain logic separation problem

In theory, a rigid separation of persistence and domain logic sounds like a good idea. However, in practice, it comes with two significant challenges.

First, whatever approach you use will imply adding and orchestrating additional data-access abstractions in multiples of the number of persisted models in your app. That translates into additional ceremony and complexity.

Second, building [rich domain models](https://dev.37signals.com/vanilla-rails-is-plenty/) becomes harder. If domain models are free of persistence concerns, how can they implement business logic that needs to access the database? In [DDD](https://en.wikipedia.org/wiki/Domain-driven_design), for example, the answer is adding additional domain-level elements such as repositories and aggregates. Now you have three elements to coordinate, with plain domain entities knowing nothing about persistence. How do they access each other? How do they collaborate? This is a tempting scenario to come up with a service to orchestrate everything. It is not surprising that, ironically, many implementations that aim to embrace the best design practices end up suffering the [anemic domain model](https://martinfowler.com/bliki/AnemicDomainModel.html) problem, where most entities are mere data holders without behavior.

The desire to separate persistence for domain logic exists for a reason. Merging both can easily result in code that doesn’t belong together or, in other words, that is difficult to maintain. This becomes obvious if you use raw SQL directly, but it is also the case if you use most ORM libraries because they only focus on the persistence side of the equation. Active Record, however, is designed on the premise that persistence and domain logic belongs together, and it comes with almost two decades of iteration around this idea.

This insight in [an article by Kent Beck](https://tidyfirst.substack.com/p/cohesion) blew my mind quite recently:

> An element is cohesive to the degree that its subelements are coupled. A perfectly cohesive elements requires all subelements to change at the same time.

In a database-powered application, domain logic is indissolubly linked to persistence. Is isolating persistence a goal or mitigation against not having the right ORM (Object Relational Mapping) technology in place? My point of view is that *if* the ORM perfectly blends with the host language, and *if* it comes with good answers for persisting your object models, and *if* it offers good encapsulation mechanisms, then the original question gets restated. Instead of *“how can I isolate persistence from domain logic”*, it becomes *“why would I”*?

---

## Conclusion

I’ve known for a long time that using Active Record as I explained here works great in practice. I have never missed a standalone layer for persistence in the Rails apps I’ve worked on.

Like with [vanilla Rails](https://dev.37signals.com/vanilla-rails-is-plenty/), some people argue that Active Record works for quick prototypes but that, at some point, you need to embrace an alternative that isolates persistence from domain logic. This is not our experience. We use Active Record as described here in [Basecamp](https://rubyonrails.org/doctrine#beautiful-code) and [HEY](https://www.hey.com), which are two quite large Rails applications used by millions. Active Record is at the heart of everything these applications do, and [we keep evolving them](https://updates.37signals.com).

There is a caveat, which is common in these articles: Active Record is a tool, and of course, you can use it to write messy and unmaintainable code. In particular, like [it happens with concerns](https://world.hey.com/jorge/code-i-like-iii-good-concerns-5a1b391c), it doesn’t replace the need to design your system properly and it won’t help you to do that if you don’t know how. If you are using Active Record and having code maintenance problems, it might not be the tool, but that you are not getting the other thousand of things that make code maintainable right.

I believe Active Record is so good that it eliminates the traditional reasons for a strict separation of persistence and domain logic. I consider that a feature to embrace proudly, not a choice to justify.
- [Other posts in the “Code I like” series](/series/code-i-like/)
- Photo by [Michael Dziedzic](https://unsplash.com/@lazycreekimages?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/blended?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
- Translations: [Japanese(zenn)](https://zenn.dev/tkebt/articles/b56656f758acac), [Japanese(techracho)](https://techracho.bpsinc.jp/hachi8833/2023_04_04/127023)
