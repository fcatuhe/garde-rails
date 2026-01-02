# Domain driven boldness

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** June 13, 2022
**Series:** Code I like
**Source:** <https://dev.37signals.com/domain-driven-boldness/>

*How to create a good domain model is the subject of many books, but here’s a lesson I learned at 37signals: don’t be aseptic, double down on boldness.*

---

![](https://dev.37signals.com/assets/images/domain-driven-boldness/cover.png)

One of the first things I did when I started working at 37signals almost three years ago was cloning the git repo for [Basecamp](https://basecamp.com). I poked around and ended up at this method:

```
module Person::Tombstonable
  …
  def decease
    case
    when deceasable?
      erect_tombstone
      remove_administratorships
      remove_accesses_later
      self
    when deceased?
      nil
    else
      raise ArgumentError, "an account owner cannot be removed. You must transfer ownership first"
    end
  end
end
```

A person in Basecamp has a [delegate type attribute](https://github.com/rails/rails/pull/39341) that represents its specific kind (e.g., User or Client). When you remove a person from a given account, Basecamp replaces it with a placeholder so that its associated data remains untouched and functional.

I was well-versed in [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html) and the importance of code reflecting domain concepts, but I had never seen that idea put into practice so intentionally before. I would have expected something like *replacing a person with a placeholder when removing it*, but *erecting a tombstone* when *deceasing a person* was so much better.

It was more eloquent, clear, and concise on the objective side. On the subjective side: it had a boldness component, like personality or soul. Can code have those? It can, and, when done right, that can be a considerable enhancement. For me, this was an *aha* moment.

Let me show another example, the [HEY screening system](https:__www.hey.com_features_the-screener_). Internally, the system looks like this: *users* examine *clearance petitions* requested by contacts who send emails.

![](https://dev.37signals.com/assets/images/domain-driven-boldness/clearance-petitions.png)

Again, I found that boldness component. *A petition* is different from a *request* because it implies formality. Screening in HEY is formal by design: people can’t get emails in your inbox without your permission. A *clearance petition* by a *petitioner* that an *examiner* has to approve is a crystal-clear way of explaining to another human what the system is doing, and that’s exactly what the code reflects:

```
class Contact < ApplicationRecord
  include Petitioner
   …
end

module Contact::Petitioner
  extend ActiveSupport::Concern

  included do
    has_many :clearance_petitions, foreign_key: "petitioner_id", class_name: "Clearance", dependent: :destroy
  end
   …
end

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

  …
end
```

This usage of concerns reminded me of roles in the [DCI architectural pattern](https://en.wikipedia.org/wiki/Data,_context_and_interaction). DCI is one of those proposals full of interesting ideas that often don’t translate so well into code. This usage of concerns is a pretty pragmatic implementation of roles.

My favorite tool when building a non-trivial model is writing a plain text description. When I worked on [improving the email analysis system for HEY](https://world.hey.com/jorge/changing-critical-code-paths-with-scientist-a3becb84), I wrote a note for myself about how the new domain model could look. Below you can see both this note (left) and the description I included in the Pull Request once the system was built (right). The note contents and its accuracy aren’t relevant — that was [writing as a thinking tool for myself](https://world.hey.com/jorge/writing-for-yourself-c25a5a13) — but plain text is a wonderful starting point when thinking about a complex system. A dictionary is a great companion for doing this.

![](https://dev.37signals.com/assets/images/domain-driven-boldness/domain-model-1.png)

![](https://dev.37signals.com/assets/images/domain-driven-boldness/domain-model-2.png)

[You can find the texts in the images here](https://gist.github.com/jorgemanrubia/51aeb56bf4ee63715d63b31bf67ce626)

Both HEY and Basecamp bet hard on Domain-Driven design since the first commit. Of course, that doesn’t mean every single corner is shiny and perfect, but, in general, it is a pleasure to read through their codebases. How to create a good domain model is the subject of many books, but here’s a lesson I learned here: don’t be aseptic; double down on boldness.
- [Other posts in the “Code I like” series](/series/code-i-like/)
- Photo by [Brett Jordan](https://unsplash.com/@brett_jordan?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/dictionary?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
- Translations: [Chinese](https://xfyuan.github.io/2023/04/domain-driven-boldness/), [Japanese(zenn)](https://zenn.dev/tkebt/articles/97954bf6775110), [Japanese(techracho)](https://techracho.bpsinc.jp/hachi8833/2023_03_02/126285), [Korean](https://velog.io/@heka1024/번역-Code-I-like-I-Domain-driven-boldness)
