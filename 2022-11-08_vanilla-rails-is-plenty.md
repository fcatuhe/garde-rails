# Vanilla Rails is plenty

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** November 8, 2022
**Series:** Code I like
**Source:** <https://dev.37signals.com/vanilla-rails-is-plenty/>

*A common critique of Rails is that it encourages a poor separation of concerns. That when things get serious, you need an alternative that brings the missing pieces. We disagree.*

---

![](https://dev.37signals.com/assets/images/vanilla-rails-is-plenty/rails.jpg)

I have often heard this: vanilla Rails can only get you so far. At some point, apps become unmaintainable, and you need a different approach that brings the missing pieces, because Rails encourages a poor separation of concerns at the architectural level.

The seminal [Domain Driven Design (DDD) book](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/) discusses these four conceptual layers: presentation, application, domain, and infrastructure. The application layer implements business tasks by coordinating work with the domain layer. But Rails only offers *controllers* and *models*: models include persistence via Active Record, and Rails encourages direct access to them from controllers. Critics argue that the application, domain, and infrastructure layers inevitably merge into a single mess of fat models. Indeed, alternatives always include additional buckets, such as services or use case interactors, at the application layer, or repositories, at the infrastructure layer.

I find this discussion fascinating because here at [37signals](https://37signals.com) we are big fans of both vanilla Rails and Domain Driven Design. We don’t run into the alleged maintenance problems when evolving our apps, so here I would like to discuss how we organize our application code.

---

## We don’t distinguish application and domain layers

We don’t separate application-level and domain-level artifacts. Instead, we have a set of domain models (both Active Records and POROs) exposing public interfaces to be invoked from the system boundaries, typically controllers or jobs. We don’t separate that API from the domain model, architecturally speaking.

We care a lot about how we design these models and the API they expose, we just find little value in an additional layer to orchestrate the access to them.

In other words, we don’t default to create services, actions, commands, or interactors to implement controller actions.

---

## Controllers access domain models directly

We are fine with plain CRUD accesses from controllers for simple scenarios. For example, this is how we create boosts for messages and comments in Basecamp:

```
class BoostsController < ApplicationController
  def create
    @boost = @boostable.boosts.create!(content: params[:boost][:content])
  end
```

But more often, we perform these accesses through methods exposed by domain models. For example, this is the controller to select the desired box for a given contact in HEY:

```
class Boxes::DesignationsController < ApplicationController
  include BoxScoped

  before_action :set_contact, only: :create

  def create
    @contact.designate_to(@box)

    respond_to do |format|
      format.html { refresh_or_redirect_back_or_to @contact, notice: "Changes saved. This might take a few minutes to complete." }
      format.json { head :created }
    end
  end
```

Most of our controllers use this approach of accessing models directly: a model exposes a method, and the controller invokes it.

---

## Rich domain models

As opposed to [anemic domain models](https://martinfowler.com/bliki/AnemicDomainModel.html), our approach encourages building *rich* domain models. We think of domain models as our application API and, as a guiding design principle, we want that one to feel as natural as possible.

Because we like to access business logic via domain models, some core domain entities end up offering much functionality. How do we avoid the issues related to the dreaded *fat model* problem? With two tactics:
- Using concerns to organize model’s code.
- Delegating functionality to additional systems of objects (AKA using plain object-oriented programming).

I’ll clarify with an example. A core domain entity in Basecamp is a `Recording`. Most elements a user manages in Basecamp are recordings — the original use case that motivated [Rails’ delegated types](https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html).

You can do many things with recordings, including copying them to other places or incinerating them. *Incineration* is the [term we use](https://world.hey.com/jorge/code-i-like-i-domain-driven-boldness-71456476) for “deleting data for good”. For the caller — e.g., controller or job — we want to offer a natural API:

```
recording.incinerate
recording.copy_to(destination_bucket)
```

But, on the inside, incinerating data and copying are pretty different responsibilities. So we use concerns to capture each:

```
class Recording < ApplicationRecord
  include Incineratable, Copyable
end

module Recording::Incineratable
  def incinerate
    # ...
  end
end

module Recording::Copyable
  extend ActiveSupport::Concern

  included do
    has_many :copies, foreign_key: :source_recording_id
  end

  def copy_to(bucket, parent: nil)
    # ...
  end
end
```

I wrote an article about [how we use concerns here](https://world.hey.com/jorge/code-i-like-iii-good-concerns-5a1b391c), if you are interested.

Now, incineration and copying are involved operations. `Recording` wouldn’t be a good spot to implement those. Instead, it delegates the work itself to additional systems of objects.

For incinerating, `Recording::Incineratable` creates and executes a `Recording::Incineration`, which encapsulates the logic of incinerating a recording:

```
module Recording::Incineratable
  def incinerate
    Incineration.new(self).run
  end
end
```

For copying, `Recording::Copyable` creates a new `Copy` record.

```
module Recording::Copyable
  extend ActiveSupport::Concern

  included do
    has_many :copies, foreign_key: :source_recording_id
  end

  def copy_to(bucket, parent: nil)
    copies.create! destination_bucket: bucket, destination_parent: parent
  end
end
```

Here, things are more complex: `Copy` is a child of `Filing`. `Filing` is a common parent class for both the *copy* and *move* operations. When a filing is created, it enqueues a job that will eventually invoke its `#process` method. That method invokes `file_recording`, a [template method](https://en.wikipedia.org/wiki/Template_method_pattern) to be implemented by child classes. When implementing that method, `Copy` creates a `Recording::Copier` instance to perform the copy.

```
module Recording::Copyable
  extend ActiveSupport::Concern

  included do
    has_many :copies, foreign_key: :source_recording_id
  end

  def copy_to(bucket, parent: nil)
    copies.create! destination_bucket: bucket, destination_parent: parent
  end
end

class Copy < Filing
  private
    def file_recording
      Current.set(person: creator) do
        Recording::Copier.new(
          source_recording: source_recording,
          destination_bucket: destination_bucket,
          destination_parent: destination_parent,
          filing: self
        ).copy
      end
    end
end

class Filing < ApplicationRecord
  after_create_commit :process_later, unless: :completed?

  def process_later
    FilingJob.perform_later self
  end

  def process
    # ...
    file_recording
    # ...
  end
end
```

This example is not as simple as the incineration one, but the principle is the same: rich internal object models hidden behind high-level APIs on domain models. This doesn’t mean we always create additional classes to implement concerns, far from it, but we do when complexity justifies it.

Along with concerns, this makes the approach of classes with a large API surface work. If you are thinking about the Single Responsibility Principle (SRP), as Michael Feathers says in [Working effectively with Legacy code](https://www.oreilly.com/library/view/working-effectively-with/0131177052/) you must differentiate between SRP violations at the interface level or the implementation level:

> The SRP violation we care more about is violation at the implementation level. Plainly put, we care whether the class really does all of that stuff or whether it just delegates to a couple of other classes. If it delegates, we don’t have a large monolithic class; we just have a class that is a facade, a front end for a bunch of little classes and that can be easier to manage.

In our example, there are no fat models in charge of doing too many things. `Recording::Incineration` or `Recording::Copier` are cohesive classes that do one thing. `Recording::Copyable` adds a high-level `#copy_to` method to `Recording`’s public API and keeps the related code and data definitions separated from other `Recording` responsibilities. Also, notice how this is just good old object orientation with Ruby: inheritance, object composition, and a simple design pattern.

On a final note, one could argue that these three are equivalent:

```
recording.incinerate
Recording::Incineration.new(recording).run
Recording::IncinerationService.execute(recording)
```

We don’t believe they are: we strongly prefer the first form. On one side, it does a better job of hiding complexity, as it doesn’t shift the burden of composition to the caller of the code. On the other, it feels more natural, like plain English. It feels more Ruby.

---

## What about services?

One of the building blocks of DDD is *services*, which are meant to *“capture important domain operations that can’t find a natural home in a domain entity or value object”*.

We don’t use services as first-class architectural artifacts in the DDD sense (stateless, named after a verb), but we have many classes that exist to encapsulate operations. We don’t call those *services* and they don’t receive special treatment. We usually prefer to present them as domain models that expose the needed functionality instead of using a mere procedural syntax to invoke the operation.

For example, this is the code for signing up a new user in Basecamp via invitation tokens:

```
class Projects::InvitationTokens::SignupsController < Projects::InvitationTokens::BaseController
  def create
    @signup = Project::InvitationToken::Signup.new(signup_params)

    if @signup.valid?
      claim_invitation @signup.create_identity!
    else
      redirect_to invitation_token_join_url(@invitation_token), alert: @signup.errors.first.message
    end
  end
end

class Project::InvitationToken::Signup
  include ActiveModel::Model
  include ActiveModel::Validations::Callbacks

  attr_accessor :name, :email_address, :password, :time_zone_name, :account

  validate :validate_email_address, :validate_identity, :validate_account_within_user_limits

  def create_identity!
    # ...
  end
end
```

So instead of having a `SigningUpService` in charge of the “signing up” domain operation, we have a `Signup` class that lets you validate and create an identity in the app. One can argue this is just a bit of syntax sugar away from being a service or even a form object. But, as I see it, it’s just plain object orientation with Ruby to give a domain concept a proper representation in code.

Also, we don’t make a big deal of distinguishing whether a domain model is persisted or not (Active record or PORO). From the business logic consumer’s point of view, that’s irrelevant, so we don’t capture the distinction between domain entities and value objects in code. They are both domain models to us. You can find many POROs in our `app/models` folders.

---

## The dangers of isolating the application layer

My main problem with the idea of an isolated application layer is that people often take it way too far.

The original DDD book warns about the problem of abusing services:

> Now, the more common mistake is to give up too easily on fitting the behavior into an appropriate object, gradually slipping towards procedural programming.

And you can find the same advice in [Implementing Domain Driven Design](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577/ref=sr_1_1?keywords=implementing+domain+driven+design&qid=1667229833&qu=eyJxc2MiOiIxLjM4IiwicXNhIjoiMC45OCIsInFzcCI6IjAuOTcifQ%3D%3D&sr=8-1&ufe=app_do%3Aamzn1.fos.006c50ae-5d4c-4777-9bc0-4513d670b6bc):

> Don’t lean too heavily toward modeling a domain concept as a Service. Do so only if the circumstances fit. If we aren’t careful, we might start to treat Services as our modeling “silver bullet.” Using Services overzealously will usually result in the negative consequences of creating an Anemic Domain Model, where all the domain logic resides in Services rather than mostly spread across Entities and Value Objects.

Both books discuss the challenges of isolating the application layer, starting with the nuances of differentiating domain and application services. Furthermore, they acknowledge that most layered DDD architectures are *relaxed*, with the presentation layer sometimes accessing the domain layer directly. The original DDD book states that what enables DDD is the *crucial separation of the domain layer*, noting that some projects *don’t make a sharp distinction between the user interface and the application layers*.

However, in the Rails world, you often see dogmatic takes that advocate against controllers directly talking to models with tremendous conviction. Instead, there should be an intermediary object to mediate between both — e.g. an application service from DDD or an interactor from [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html). I believe such nuance-free recommendations favor the appearance of either:
- Tons of boilerplate code because many of these application-level elements simply delegate the operation to some domain entity. Remember that the application layer should not contain business rules. It just coordinates and delegates work to domain objects in the layer below.
- An [anemic domain model](https://martinfowler.com/bliki/AnemicDomainModel.html), where the application-level elements are the ones implementing the business rules, and domain models become empty shells carrying data around.

These approaches are often presented as a [tradeoff-free answer](https://world.hey.com/jorge/no-silver-buckets-84d249d5) to a very complex problem: how to design software properly. They often imply that good architecture happens as a consequence of using a discrete set of archetypes, which is not only very naive but incredibly misleading for the inexperienced audience. I hope that the alternative I presented here resonates with people looking for more pragmatic alternatives.

---

## Conclusion

In our experience, this approach with vanilla Rails results in maintainable large Rails applications. As a recent example, we just launched [Basecamp 4](https://basecamp.com/new) built on top of Basecamp 3, the codebase of which is almost 9 years old, includes 400 controllers and 500 models, and serves millions of users every day. I don’t know if our approach would work at Shopify scale, but I am sure it would for most businesses using Rails out there.

Our approach reflects one of the Rails’ doctrine pillars: [*No one paradigm*](https://rubyonrails.org/doctrine#no-one-paradigm). I love architectural patterns, but a recurring problem in our industry is that people get very dogmatic when translating those into code. I think the reason is that simple strict recipes are very appealing when tackling a problem as complex as software development. 37signals’ code is the best I’ve seen in my career, and I say that as a spectator since I haven’t written most of it. In particular, it is the best incarnation of DDD principles I’ve seen, even if it doesn’t use most of its building blocks.

So if you abandoned the vanilla Rails path and now you are wondering if you really need those additional boilerplate classes whenever you need to handle some screen interaction, be sure that there is an alternative path that won’t compromise the maintainability of your application. It won’t prevent you from having to know how to write software — no alternative will — but it might bring you back to happy territory again.

---

*Thanks to [Jeffrey Hardy](http://quotedprintable.com) for his valuable feedback as I wrote this article. He’s one of the main contributors to this approach of architecting vanilla Rails applications I’ve come to learn and love.*
- [Other posts in the “Code I like” series](/series/code-i-like/)
- Photo by [Johannes Plenio](https://unsplash.com/es/@jplenio?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/rails?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
- Translations: [Chinese](https://xfyuan.github.io/2022/11/vanilla-rails-is-plenty/), [Japanese](https://techracho.bpsinc.jp/hachi8833/2023_01_12/124378)
