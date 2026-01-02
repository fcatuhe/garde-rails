# Minding the small stuff in pull<br /> request reviews

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** August 30, 2023
**Source:** <https://dev.37signals.com/minding-the-small-stuff-in-pr-reviews/>

---

*Nitpicking* in pull request reviews means offering insight that looks excessive, pedantic, and unimportant. It’s a pejorative term. I don’t question there are cases where the term applies, but many folks assimilate it to paying attention to the *small stuff*. By projecting all these negative connotations, this practice suddenly becomes undesirable: mind the big picture or leave the pull request alone. With that, we disagree. *Small stuff* when writing code matters, a lot.

---

## Names matter

Naming things is a usual suspect when illustrating nitpicking. It’s also the example I have the hardest time swallowing. With the importance of names to make code understandable, how can anyone consider discussing those superfluous?

Check what [Jeffrey Hardy](https://github.com/packagethief) told me in this recent pull request:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-1.png)

Feedback from Jeff to Jorge

As a generic delegation controller a “forward” click is hard to understand, and may even suggest that it’s the opposite of a “back” click. I think it’s intended to mean “forward the click”, but I’d argue that it’s already implied by the controller name.

I’d probably call the action just `click` (usage: `click->click-delegator#click`). I’d also call the target element just the `delegate` or `delegateElement`.

Or what I told [Matt Hutchinson](https://github.com/matthutchinson) in another one:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-2.png)

Feedback from Jorge to Matt

I’d use past forms for these event handlers, such as `linkClicked`, `linkFocused`, `linkBlurred`, to be consistent with other similar handlers in this controller.

---

## Code style matters

Great code aesthetics is a distinctive trait of Ruby, and one of the [Rails pillars](https://rubyonrails.org/doctrine#beautiful-code). Compare:

In Javascript:

```
if (this.hasParamTarget) { this.paramTarget.value = id }
```

In Ruby:

```
param_target.value = id if has_param_target?
```

In Python:

```
datetime.now() - timedelta(days=5)
```

In Rails:

```
5.days.ago
```

Does this attention to aesthetics make you a happier programmer? And this is not a rhetorical question. Tons of outstanding programmers couldn’t care less about this stuff. But if you do, paying attention to code style and aesthetics in pull requests is logical and desirable. This includes suggesting code that could be formatted more cleanly, minor logic simplifications, or slightly more idiomatic alternatives. Precisely the kind of stuff that is often labeled as *nitpicking*.

Now, check these examples:

Jeff points out a conditional that could be formatted more cleanly:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-3.png)

Feedback from Jeff to Jorge

I find the optional option with inline parens a bit hard to parse. This form might be nicer:

```
@calendar = Calendar.new
@calendar.build_subscription if params[:subscription]
```

And, in the same review, he suggests a more idiomatic way of using `enum`:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-4.png)

Feedback from Jeff to Jorge

Since the suffix is named the same as the attribute, can just use `_suffix: true` ✨

Here, I suggest to Jeff to use `.ids` instead of `pluck(:id)`. I learned about this one in another pull request review by [Lewis Buckley](https://github.com/lewispb) back in the day. Nitpicking is transitive.

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-5.png)

Feedback from Jorge to Jeff

You can use `.ids` here and in the `else` condition too.

In this other example, [David](https://dhh.dk) suggests some minor simplifications to me in the logic:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-6.png)

Feedback from David to Jorge

Would inline this as part of `load_calendars_set` and decide whether you’re using the accessor method or the ivar.

Here, Jeff suggests that I get rid of a guard clause:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-7.png)

Feedback from Jeff to Jorge

I’d one-line. I think guard clauses work best when they save you a lot of reading, and prevent you from missing an else branch further down in the code.

```
positioning&.reposition_to(new_position.to_i, default_score: default_score)
```

And I suggest something similar to Matt here:

![](https://dev.37signals.com/assets/images/minding-the-small-stuff-in-pr-reviews/review-8.png)

Feedback from Jorge to Matt

Can remove the `present?` guard if you do `string.to_s.scan`.

---

## Consistency matters

Consistency helps to make code easier to understand and, thus, more maintainable. In other words, it’s priceless. However, consistency is elusive and very hard to achieve in large codebases as time passes and different people work on them. By paying attention to the big picture you may achieve consistency at the architectural level, and by using linters, you can expect consistency at the code-formatting level. But what about the very thick middle ground? This is where coding standards live.

Thorough pull requests that pay attention to the *minutia* are essential to enforce coding standards and transmit them throughout a team. For sure documentation can help, and nothing replaces skills and experience, but I can’t imagine how a codebase can keep a high level of consistency, unless eyes very familiar with the in-house style review changes with a magnifier.

A good example is the preference to avoid guard clauses unless they save you from a long sequence ahead – as showed in the previous section. Whether you agree with a guideline or not, the value of conventions resides in having them in the first place. They help with consistency; and the most interesting and juicy ones can’t be validated automatically.

---

## Conclusion

To consider the small details in code as unimportant and even pedantic reminds me of the vilified *architect* role in our industry: the one that doesn’t really code, just cares about architectural concerns and patterns, the stuff that matters. The problems with this role are apparent to most but, at the same time, many [teams](https://blog.danlew.net/2021/02/23/stop-nitpicking-in-code-reviews/) [consider](https://www.reddit.com/r/cscareerquestions/comments/58n84u/how_do_you_handle_nitpicky_codereviewers/) [nitpicking](https://www.steveonstuff.com/2022/02/09/nitpicky-code-reviews-are-are-drag) [something](https://blog.joetr.com/avoid-nitpicking-in-code-reviews) to [avoid](https://www.mattlayman.com/blog/2017/no-nitpicking-code-reviews/).

The big picture matters, for sure, but computers don’t execute big pictures, they execute lines of code, so how you compose those matters too. And code-aesthetics lover or not, everyone benefits from consistency.

I just had to check a few recent pull requests I was involved in to find many examples to pick from. Isolated and out of context, any of those suggestions are objectively trivial. But their aggregation is not. This approach is part of our development culture. This is how new people learn the in-house coding style, how we keep a high-quality bar in our code, and how we favor consistency.

I have ended up assimilating *nitpicking* in code reviews with being meticulous, thorough, and rigorous. I encourage you to review the threshold beyond which you consider some review advice as nitpicking. Maybe it is not aligned with what you value as a programmer.
