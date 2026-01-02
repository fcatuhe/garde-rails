# Compared to what?

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** December 13, 2022
**Source:** <https://dev.37signals.com/compared-to-what/>

*When discussing software design techniques, actual code should be a mandatory ingredient.*

---

I loved [this article by Kent Beck on cohesion](https://tidyfirst.substack.com/p/cohesion). At some point, when discussing how cohesion and coupling are opposing forces, it says:

> Wait!? Isn’t coupling bad? But cohesion is good? But cohesion is coupling of subelements? Isn’t that bad? And good?
> This is where I got stuck for a long time (years). What I was missing was the “compared to what?” perspective…

The “compared to what” reference resonated strongly with me. A recurring thought I have had lately is that most takes on software design techniques make such comparison impossible because they don’t include real code. By *real code* I mean code extracted from *real* apps.

I have written some pieces on Rails design techniques in the [Code I like series](/code-i-like-series-by-jorge), and I intend to address more in upcoming posts. I’m very intentional about sharing fragments of code from our products, which are widely-used Rails applications. No synthetic or contrived examples, but the real thing.

I [loathe the term *real world*](https://world.hey.com/jorge/smelling-rails-smells-0ffacec5) when presenting software design approaches, but I’m using it here very deliberately: I believe actual code should be a mandatory ingredient in these discussions. A reality layer separates the words describing concepts in the abstract and the code that puts those in motion. It’s impossible to have an effective debate if you obviate that.

This reality layer takes many forms. Sometimes, it implies a significant ceremony that makes the system harder to understand and maintain. Others, it comes as a local improvement that deteriorates the system as a whole. And because reality usually brings many exceptions, answers for those often pollute the initial idea to the point it is not that appealing anymore.

Moreover, in practice, it’s normal to have to choose a solution that is not ideal; it’s just the best one compared to the alternatives at hand. You know, context, nuance, tradeoffs, etc. Abstract discussions rightfully ignore those to present concepts as crystal clear as possible. Code from real-world applications, however, exposes them all mercilessly.

I always ask for permission before sharing our code at [37signals](https://37signals.com), and I always get a resounding *YES, PLEASE*. This is not surprising, considering they were talking about [this very same topic ten years ago](https://dhh.dk/2012/the-parley-letter.html). I wish more companies did the same. I would love to see more takes on design techniques that showed real code from actual apps. In other words, I wish more articles made it easier to answer the opening question here:

Compared to what?
