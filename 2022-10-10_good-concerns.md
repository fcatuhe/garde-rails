# Good concerns

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** October 10, 2022
**Series:** Code I like
**Source:** <https://dev.37signals.com/good-concerns/>

*We love concerns and have been using them for years in large codebases. Here we share some of the design principles we use.*

---

![](https://dev.37signals.com/assets/images/good-concerns/cover.jpg)

[Rails concerns](https://api.rubyonrails.org/classes/ActiveSupport/Concern.html) have received much criticism over the years. Are they the solution to all the problems or something to avoid at all costs? I think a problem with concerns is that you can use them however you want, so no surprise you can shoot yourself in the foot when doing it. After all, concerns are just [Ruby mixins](https://ruby-doc.com/docs/ProgrammingRuby/html/tut_modules.html) with some syntax sugar to remove common boilerplate code.

[37signals](https://37signals.com) has years of experience using them in large Rails codebases, so I would like to share some of the design principles we use in this post.

---

## Where to put concerns

Ruby mixins are often presented as an alternative to multiple inheritance: a code-reuse mechanism across classes. We use some concerns this way, but the most common scenario where we use them is to organize code within a single model. We use different conventions for each case:
- For common model concerns: we place them in `app/models/concerns`.
- For model-specific concerns: we place them in a folder matching the model name: `app/models/<model_name>`.

For example, this is an example of a model-specific concern from Basecamp:

```
# app/models/recording.rb
class Recording < ApplicationRecord
  include Completable
end

# app/models/recording/completable.rb
module Recording::Completable
  extend ActiveSupport::Concern
end
```

This convention removes the need to repeat the namespace when including the concern.

For controllers, the situation is inverted. We place most concerns in the `controllers/concerns` folder, with some concerns that only apply to a certain subsystem placed in a subfolder named after that: `controller/concerns/<subsystem>`. I would like to explore how we do controllers in another post.

---

## Improve readability

A common criticism of concerns is that [they worsen readability](https://www.cloudbees.com/blog/when-to-be-concerned-about-concerns). I think the opposite is true. When used right, they improve readability in two ways:

First, *they help to manage complexity*. The essence of dealing with complex systems is to [divide them into smaller pieces over and over so we can focus on one thing at a time](https://world.hey.com/jorge/code-i-like-ii-fractal-journeys-b7688f93). A concern is another tool in your toolbox to achieve precisely that.

The key here is that each concern should be a cohesive unit that captures a trait of the host model. In other words, they should only contain things that belong together. You should not treat concerns as arbitrary containers of behavior and structure to split a large model into smaller parts. They need to feature a genuine *has trait* or *acts as* semantics to work, just like class inheritance needs the *is a* relationship. They will cause more harm than good otherwise.

Check this example from the HEY screener I [talked about in the past](https://world.hey.com/jorge/code-i-like-i-domain-driven-boldness-71456476). Users in HEY act as examiners of clearance petitions from other contacts that want to send them an email:

```
class User < ApplicationRecord
  include Examiner
end

module User::Examiner
  extend ActiveSupport::Concern

  included do
    has_many :clearances, foreign_key: "examiner_id", class_name: "Clearance", dependent: :destroy
  end

  def approve(contacts)
    …
  end

  def has_approved?(contact)
    …
  end

  def has_denied?(contact)
    …
  end

  …
end
```

The concern matches the domain role of an examiner of clearance petitions, and it only contains code related to that role. This enhances maintainability: the fewer concepts you need to manage at any moment, the easier things are to understand.

And second, *concerns offer an additional abstraction to reflect domain concepts*.

Below are the concerns included by the `Topic` model in HEY. Just like the examiner example, notice how most names capture domain concepts that are easy to grasp. They offer an additional opportunity to resemble the domain, which is a net positive regarding readability.

```
class Topic < ApplicationRecord
  include Accessible, Breakoutable, Deletable, Entries, Incineratable, Indexed, Involvable, Journal, Mergeable, Named, Nettable, Notifiable, Postable, Publishable, Preapproved, Collectionable, Recycled, Redeliverable, Replyable, Restorable, Sortable, Spam, Spanning
```

…

---

## Enhance, but not replace, rich object models

A common misconception with Rails concerns is that they represent an alternative to traditional object-oriented techniques, such as class inheritance or composition. Take [this](https://medium.com/@carlescliment/about-rails-concerns-a6b2f1776d7d):

> Business logic is better modeled as abstractions (classes), rather than concerns. Value objects, services, repositories, aggregates or whatever artifact that fits better.

Or [this](https://www.cloudbees.com/blog/when-to-be-concerned-about-concerns):

> *Favor composition*
> I’m not saying you HAVE to put everything in one file. Please, by all means, extract some logic into a custom class and call it.

I think this is a false dichotomy. Using concerns doesn’t limit or replace the need to design systems properly. In particular, you shouldn’t use concerns to enable fat and flat Active Record models nicely organized instead of proper systems of objects with a good distribution of responsibilities. I know that’s a real risk with concerns because I created such messes during my first experiences with them.

37signals is big on good old object-oriented design, inheritance and composition, design and implementation patterns, and we have POROs all over our *models* folder. Concerns actually play great with this approach. Let me illustrate this with a simple example.

In HEY, paid customers keep their email address reserved forever, even if they cancel their subscription. Because of that, when the system terminates an account, it chooses between fully deleting all the data (incineration) or just keeping a minimal set, such as outbound forwarding (purging). Let me show you some relevant parts of the code:

```
class Account < ApplicationRecord
  include Closable
end

module Account::Closable
  def terminate
    purge_or_incinerate if terminable?
  end

  private
    def purge_or_incinerate
      eligible_for_purge? ? purge : incinerate
    end

    def purge
      Account::Closing::Purging.new(self).run
    end

    def incinerate
      Account::Closing::Incineration.new(self).run
    end
end
```

Incineration and purging are involved operations that share some common code. So guess how we solve that? With additional classes encapsulating the operations and with good old inheritance to reuse common bits:

![](https://dev.37signals.com/assets/images/good-concerns/account-closing.png)

I love this approach of using concerns to offer a nice domain-oriented API on models that hides a complex subsystem from the caller’s point of view. If we want to terminate an account, we can just say:

```
account.terminate
```

Versus something more verbose and less fluid like:

```
AccountTerminationService.new(account).run
```

And notice that we don’t have a fat `Account` model responsible for dealing with all the logic of incinerating or purging accounts. There is a subsystem of three classes in charge of that, and the `Account` model offers just the door to use it.

Concerns enable these more concise, and nicer-looking APIs while keeping model code organized and without sacrificing what you can do regarding system design.

---

## Conclusion

Concerns are a tool. I am not sure if they qualify as [sharp](https://rubyonrails.org/doctrine#provide-sharp-knives) or if they are just too open, but they can cause trouble when misused. However, with some simple guidelines, I think they are a fantastic resource if you are a Rails programmer.

Concerns combined with good object-oriented design are a sweet combo. Of course, concerns won’t remove the need for having to know how to design software. Still, they are a pragmatic mechanism to improve your code organization, making it more intelligible and maintainable.

You often hear that vanilla Rails will only get you so far and that you need additional constructs, harnesses, and conventions on top of it. If it serves, [Basecamp](https://basecamp.com) and [HEY](https://www.hey.com) are vanilla Rails apps using traditional object orientation and patterns, and they heavily use concerns.
- [Other posts in the “Code I like” series](/series/code-i-like/)
- Photo by [Vardan Papikyan](https://unsplash.com/@varpap?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/puzzle?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
- Translations: [Chinese](https://xfyuan.github.io/2023/06/good-concerns), [Japanese(zenn)](https://zenn.dev/tkebt/articles/9e481331719d6d), [Japanese(techracho)](https://techracho.bpsinc.jp/hachi8833/2023_04_04/127023), [Korean](https://velog.io/@heka1024/Code-I-like-I-Good-concerns)
