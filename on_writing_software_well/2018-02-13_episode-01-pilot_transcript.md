# On Writing Software (well?) #1: Pilot Episode — Transcript

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 13, 2018
**Series:** On Writing Software Well
**Episode:** 1 — Pilot Episode
**Source:** <https://www.youtube.com/watch?v=H5i1gdwe1Ls>

---

Hey everyone, welcome to On Writing Software — and writing software well. I don't know, I don't know what this is going to be called yet. What I do know is I wanted to have an opportunity on software in writing, specifically looking at code while we do it. Not just as an abstract form of discussion, not really in a way of trying to show you a specific library or whatever — just the considerations that I encounter as I'm writing code, predominantly for Basecamp, sometimes for Rails. We'll see where it goes.

I just wanted to have basically a chat with the camera as though I was sitting and having a conversation with another programmer and we were discussing: how can we make this piece of code better? How can we make it clearer, easier to understand? All these good things that we're considering when we're writing software.

## Part 1: Code Comments as a Smell

So the first thing I want to open up with and talk about today is the idea of code comments. Code comments — I have a sort of mixed relationship with that. Sometimes it's really great that you can have a chance to explain something that does not seem obvious from the code. But more often than not, I tend to look at code comments as a bit of a smell. Why are you explaining something that isn't apparent or clear from the code itself? If you're writing in a programming language, right — Ruby — you should be able to encapsulate whatever it is you want to explain in the code itself, such that this does not need additional explanation in prose.

Now here's a piece of code we have in Basecamp 3. This is an encapsulation of the access concept. People have access to what we call buckets, which is this abstraction that covers teams and projects and circles and stuff like that. And this access object basically is what we check for when we check whether a person has access to view a piece of content.

Now the access itself, when it's just up and running — this basically has-too-many relationship. Buckets can have many people accessing them, an individual person can have access to multiple buckets. But when the access is revoked, that's when something interesting actually happens. We need to do two specific things in Basecamp.

We need to reconnect the user from the Action Cable server — that's the thing we use to manage the WebSockets connection that we're sending updates over. Basically, you shouldn't get updates from channels that relate to buckets that you no longer have access to. So what we do is we check whether the person first of all is a user. We have a few different ways of dealing with persons in Basecamp. One of them is that someone is a user where they can log into the system. Another version is where they're a client and they can't log into the system. And a third is actually that they're a tombstone — that they've been deleted — and then of course they can't log into the system either.

But that's one thing we need to do. What I really want to focus on today is this `remove_inaccessible_records` method, because as you can see it is the only method in this class that has code comments.

And I was just reading through the Basecamp code. I started doing a new branch yesterday that I called "DHH refactoring 2018," as I frequently do. I read through the entire codebase at Basecamp 3 and try to make things better that I don't think are good enough, or basically revisit decisions that we've made earlier that I think now I have a better idea of how to do.

And the first file I opened was this access document. As you can see, it's literally the first model in our directory. So I thought — when I stood there and the first thing I really just noticed was this comment. Right? When we're removing inaccessible records, we give you basically 30 seconds of grace period to undo that. When we remove all the inaccessible records, it's kind of a job that takes a while to run — that's why we run it in a job in the first place — and it's also irreversible. All these records that need to be removed, when they're removed it's kind of a pain in the ass to add them back in.

Let's actually just take a really quick look at that inaccessible records job. So it removes a bunch of things. We have these things called readings, which is a recording of whether someone had read a piece of content in the system. We have bookmarkings. We have a bunch of things. It doesn't really matter so much. What matters is that these things need to disappear when someone loses access to a bucket.

But the nice thing we came up with here was that sometimes you accidentally end up removing someone, and if you add them back in 30 seconds, we won't actually have destroyed all these records. And it's kind of just a way of offering an undo.

In any case, I didn't like this comment here. I felt like, why do we need a comment to explain this concept of forgiveness when we could explain it in code?

And the first thing I thought about here was: what does this actually explain? It's not explaining this conditional — it's actually explaining this somewhat magical integer. Here, in Ruby — well actually in Rails, I suppose it's an Active Support addition — you can say `30.seconds` to explain what this 30 actually means. It actually just returns 30, right? But we could have written `30.minutes` and then it would have returned 30 times 60.

Well, we have our 30 seconds here, but it's still a bit of a magic variable, because what are these 30 seconds? Well, that's what we explained up here, right — it's 30 seconds of forgiveness.

Couldn't we just explain that some other way? Well, the standard way is of course to extract an explaining variable. So we could just do it inline. We could do:

```ruby
grace_period_for_removing_inaccessible_records = 30.seconds
```

And then replace the concrete usage down here with this explaining variable. Great, alright — this is actually a step forward already. We could just delete this and that looks better. That's not bad.

I tend to see these inline declarations of variables as another sort of smell, right? Like, this is not actually something that needs to happen every single time we run this method, because it never changes. It's a constant, in fact. Right?

Okay, great — constant. Let's extract that constant and see what we can do. We could take the constant here and we could — what we normally do, what I normally do — is just shove it right up here, and then I think TextMate has this way of changing case... there we go.

Alright, great. We have a constant up here, we can use the same constant down here, we're referencing it. That's an improvement too, right? This is getting better.

There's something else though I find when I extract these constants: how far away are they from their usage, and is this a general constant that someone should consider as a public constant, or is it more of a private thing? In this case, I think it's actually more of a private thing. We're not going to need to refer to this constant directly somewhere else, so we don't actually need to have it declared publicly. So we could remove this, put it down on the private. Okay, even better still — it's closer, but it's not really fully close enough to the method that we're using.

So let's swap the order of these two methods, and now it's pretty good, I'd say. You have the method that references this constant right next to where the constant is declared, so you read them sort of together.

That introduces sort of a separate problem here. I love to have my private methods in basically the order of a table of contents — table of contents is what I call it — and the table of contents is declared by when they're called. And we're declaring them when they're called through a couple of callbacks here. We have `after_destroy` that does the reconnection of the user, and then we have the `after_destroy_commit` that once the access is destroyed and that transaction is committed, then we remove the records from it.

So they're not really in the right order here. `reconnect` is called first so it should be first. But if we just took basically this block and moved it down here, that sort of works, but then we have this constant floating here in midair, which doesn't really seem that aesthetically pleasing to me either. I like it when it's just right on to the private scope.

And this is sort of what I'm talking about. Like, these are the considerations where you have a general principle that you want your private methods to appear in the order in which they're called — the table of contents — and then that conflicts with another aesthetic consideration that you want your constant to be declared first and you want them to be declared as close to the usage of that constant as possible.

So when you have these two considerations that are somewhat in conflict, you have to choose which one is more important to you. And in this particular case, I think that it's actually more important to have this look good and then violate the idea of the table of contents, especially since we just have these two methods here.

It's a good example too of where, if you just have this automatic style checker, it can't really encapsulate and deal with this and what we're trying to do. And what I'm trying to do, basically, is just make this easier for me to read, more aesthetically pleasing for me to read, easier to come back to.

Yeah, so that's basically the considerations that went into just extracting one simple constant because we didn't like the — or I didn't like the — use of a code comment.

## Part 2: Extracting Patterns into Rails

And that's it. The second thing I wanted to talk about today is basically how I go about extracting things into Rails. A lot of that comes from reading through the codebase. As I said, this is this mission I'm on right now, reading through the entire Basecamp 3 codebase and finding things that would be well suited to go into Ruby and Rails.

And the thing I've stumbled upon here is this block — the `grant` on `person_administratorships`. So administratorships is basically a join model, again, between an account and a person, and who has the right to administer that account — who has the right to grant other people access to things and remove people and add people to the account and all the stuff that you normally allow admins to do.

Well, we've encapsulated that in the idea of an administratorship, rather than just having an admin boolean or something else like this. I really like encapsulating these domain objects and being able to reason about them and giving them a place to live and giving them a whole concept rather than just giving them a boolean.

And in any case, here we have an administratorship association — it has-many association — but we want to do something a little nicer on the DSL side of things when you want to give people an administratorship or revoke it. And in the `grant` method here, you see, what we're basically doing is we're adding methods onto the association. So it'll be `account.administratorships.grant` and then you pass it a person.

Now, an administratorship — you only need one. So if a person already has one, we shouldn't allow them to create another record in the database. They shouldn't confuse things. So first of all, we have a unique index on this pair between the account and the person in the migration that we set up for the administratorship. But we also want to basically deal with that if it pops up in the code — if somehow someone triggers or tries to grant an administratorship that already has been granted.

And we catch that. Well, you see — it's a little odd here, right? So we're actually using exceptions as control flow. And what this block of code really wants to be is something much shorter than this, right? It shouldn't be all these lines. It shouldn't have this code comment, as we just talked about in the session before — code comments are a bit of a smell. The fact that we need to explain what's going on means that it's not quite obvious what's going on.

Well, what's going on is that Rails has a default method basically called `find_or_create_by` some pair of attributes — this will be person. But Rails does this in a sort of way that encourages people to get stale reads if they have a really busy application. Because `find_or_create_by` will start a transaction, then it will try to do a `where` call on the association for a person — see if there's already an administratorship between the account we're already on and the person we're passing in — and if so, just return that. Otherwise, create something.

Well, those two separate steps of course are separate steps, and you're liable to end up with a stale read. And on an application like Basecamp that has a fair amount of usage, this certainly can happen. And there's just a better way of doing it.

And the better way of doing it is to basically try to create this administratorship association up front, and if it does not work, well, that means that it already happened. Right? So we're basically just doing one insert and relying on the unique index in the database to throw an exception if this person already exists, and then just return the record we're looking for.

We didn't have that — or we don't have that right now — so it needs all of this. What we want really is what we have in Rails right now, just flipped. We want `create_or_find_by` person. And then we can encapsulate this whole pattern of doing the create, catching the exception, and just returning the person that's already in the database if that happens.

And then if we have that, we can reduce this whole thing to basically an alias. `grant` becomes an alias just to `create_or_find_by` person. Makes the DSL just slightly a bit nicer.

And there we go. So this is one of those things I'm looking to extract — just a small extension, put it into Rails — and then we'll have this pattern go forward. In fact, perhaps this pattern should be the default and we should consider whether we want to deprecate the `find_or_create_by` signature instead.

Just one other thing I'm noticing here — I don't know how we ended up with this, but we can actually just deal with records directly. It doesn't need to deal with just IDs. So let's fix that at the same time.

And there we go. That's pretty much it.
