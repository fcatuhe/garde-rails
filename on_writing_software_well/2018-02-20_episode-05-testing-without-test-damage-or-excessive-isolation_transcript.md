# On Writing Software Well #5: Testing without test damage or excessive isolation — Transcript

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 20, 2018
**Series:** On Writing Software Well
**Episode:** 5 — Testing without test damage or excessive isolation
**Source:** <https://www.youtube.com/watch?v=Tc5z64XIwIY>

---

Hey, this is David Heinemeier Hansson. Welcome to episode 5 of On Writing Software Well. Today I'm going to show how we do testing in Basecamp. We're gonna focus on two specific aspects: model testing and controller testing.

## Setting the Stage on Testing Philosophy

But before we get into that — and I'm going to show a specific feature and how we do all the tests around that — I just want to set the stage a little bit.

I've talked a fair bit about testing over the years. A couple years back, I had a conversation with Kent Beck and Martin Fowler following a conversation or keynote at RailsConf — the title something along the lines of "TDD is Dead." And that keynote and the following conversation sometimes get misconstrued as though I don't like testing, we don't do testing at Basecamp, or other nonsense like that. Which is, yeah, nonsense. Because we do a lot of testing at Basecamp. We have tens of thousands of written lines of test code. I do not feel comfortable in any way, shape, or form shipping new features without us having a suitable amount of automated testing.

Where my criticism in the past has come in around TDD and other testing techniques has been what they do to the code itself. What happens when you subject model code, controller code, to stringent requirements offered up by testing — such as the idea of unit testing, that you're testing a single thing in total isolation and no other side effects and no other interactions are supposed to happen, all that stuff is to be stubbed out in the name of speedy tests.

Well, I don't think that's a good trade. We talk a lot about trade-offs on this channel. And the trade-off between having super-duper miraculously fast tests where you can perhaps run hundreds of tests in just a few seconds, versus having to service a cadre of mocks and stubs and other forms of dummy methods or dummy objects to be able to do that — I don't think it's a good trade.

I think in the majority of cases you're better off testing the real things — interfacing with the real database, interfacing with the real generation of HTML and controllers. These things that are somewhat slow but can be mitigated through a bunch of different factors and can be made fast enough.

The purpose of developing software is not to have fast tests. The purpose of testing new software is to have high confidence that the features that you're shipping are of high quality and that they don't break — at least amongst the common paths that programmers are likely to test down.

I think there's a whole other episode we could do around QA, especially exploratory testing done by humans — humans that were not part of the development process. And I see great value in that as well. In any case, I'll link to a couple of position pieces on testing and TDD in particular in the show notes.

But with that set, let's get started. Let's look at how we test a specific feature in Basecamp using concerns and all that other stuff we've been talking about in previous episodes.

## The Feature: Documents and Locking

So the feature we're going to look at today is really two features. It is the documents feature and it's the aspect of documents that they can be locked — such that only one user in the system can edit them at a time.

The locking aspect of documents comes to us through a generic consideration that flows through the recordings object. Recordings is this base object in Basecamp 3 that provides a lot of the shared behavior that multiple types of content in the system can use. And documents is one of the types in the system that can use this lockable concern.

### The Document Class

As you can see, the document class here doesn't really have a lot in it, right? Like, it's mostly a container just for the data, not so much about the behavior. Because a lot of the behavior in the system is held in the recordings class, because it's shared by multiple different types of content.

For example, the fact that you can lock a document could then also be a feature that you can lock a to-do list or something else, to prevent concurrent access.

In any case, as you can see here, we are declaring a couple things. The fact that documents have rich and plain text attributes — this is something that ties into our content system, the fact that we use a rich text editor called Trix to provide uploads and formatting and all these things. So that's really sort of inherent to document — these are attributes on the document. These are not generic things — not everything has a content, not everything has a title, but documents do. So this is where we declare that.

We also declare the fact that there is a default title called "Untitled." Then we use this neat little trick of calling `super` — because document is of course an Active Record base. These days, if you had started a new application, it would have been an `ApplicationRecord` in Rails, but we predate that and haven't updated since.

And then we have these three different predicate methods: `auto_positioning?`, `subscribable?`, and `exportable?` — that provide the recording features a way to interrogate the recordable (of which document is one) and ask whether they're capable of doing this stuff that they're doing.

### The Subscribable Pattern

Just to set aside — this isn't really what we're going to be talking about — but I just wanted to show that, for example, the fact that a document is subscribable means that users in the system can subscribe to a document to get comment updates and changes and so on from it.

And I just wanted to show this subscribable concern that we have on recordings and the delegation here — the fact that we only subscribe the creator, for example, to a recording if we say that the recordable is subscribable. This is a way of disconnecting the generic concerns (of which subscribable is one) from the specific content types such as documents, and whether a document is able to be subscribed to or not. We don't have to maintain a master list of types here that can be subscribable — we sort of flip that around and let the document answer whether it's subscribable or not.

### The Lockable Concern

In any case, let's have a look at the lockable concern, which is something we're going to test here — the actual piece of behavior.

And it comes to us through the recording, as I just said. The recording is this base type that provides a ton of behavior to all the different content types that we have in Basecamp — whether it's a document or to-do list or file upload or whatever. And the concern we're going to talk about is the lockable concern.

So that's one out of this long list of all sorts of things that recordings can do. They could be moved, they could be mentioned, they can notify, they can be published, they can be a bunch of things — a bunch of behaviors that are generic, applying to multiple recordable types.

The lockable concern is really quite simple. It just basically maintains a little bit of nicety of DSL around the fact that there's an association here — that a recording can have a lock, and a lock can be owned by a given user. And when you try to lock a recording — lock a document — then we first check whether it's already locked and whether we can query whether it's locked by the same user. So that's basically the feature here.

### The Feature in Action

Let me actually show you how that looks in the app itself. So if we go over here and take a look at the app: on the left-hand side I have Victor logged in, and on the right-hand side I have a private session (which is basically just a way of getting another cookie domain such that I can be logged in as two different people at the same time) — this is Cheryl.

If Victor goes here and says he wants to edit this document, well, he'll be let in right away because there's no one already editing this — he gets access right away.

If Cheryl, on the other hand, goes to try to edit the document at the same time as Victor is editing — well, she's going to get this screen: "This document is locked while Victor Cooper makes edits. Blah blah. Do you want to break the lock? Do you want to nevermind, no bother with it?"

So that's basically the feature we're trying to expose here.

## Model Tests

But before we get to that, let's have a quick look at just the simple tests that we do around a document alone — not considering the lockable concern.

### Document Test

And you'll notice a bunch of things I'll sort of call out here. One is the fact that the setup method — the only thing we do is to set who the current person is. I talked about this concept of `Current` globals in a previous episode — you can go back and look at that in the playlist. We set that up such that whatever is being done, interacted with the system here, all those actions would be recorded as me being the creator or editor or whatever.

And then what you'll notice is that there's no factory methods here. We use the stock, vanilla Rails idea of **fixtures**. And fixtures is this wonderful concept where you basically configure a base set of characters in a world that you can pull upon whenever you want to play a certain part. These are sort of just defaults that can be used across different forms of testing, without the sort of particulars that correspond to that specific test.

For example, we have a bucket — which is this generic concept that's used for teams or projects. In this case, it's a project, an anniversary project, that's just a standard project. When we need to test something that needs to interact with a project, we don't need to first go through the trouble of setting that up — we just pluck out a fixture for that and start using it.

So that's what we're doing in this simple test. It basically records a new document onto the bucket, which gives us a recording back. And that recording has a correlation to our recordable, which is the document itself. And we just do a few simple asserts on that — that these values we've said go all the way through.

### Testing Versions

This next test is a little bit more interesting. Documents have this feature of having their previous versions stick around. When you record a new document onto a bucket, you'll get a recording. But if you update that recording, we still keep the old version of the document around.

I promise a few times that I was going to talk about our unique setup of buckets and recordings in a future episode, and I swear that episode is going to come soon. Because I think a lot of the examples that I go through in Basecamp will make more sense once you sort of have that connection.

But for now, just know the fact that recordables — such as documents — are immutable, versus recordings are mutable. So recording is sort of the current version and that will point to the most recent immutable document, and then it'll still keep a record of the past versions, the past recordable versions, as we test here.

So this test is basically just showing that when you make an update, we can still have access to the past versions of these recordables. First, we test that the first version of the recordables is the latest update that we made ("new order" is what we updated here), but then also that the previous document — the second version — is what it was up here.

One curious little funky thing here is we do this `travel 5.seconds`. Why do we travel 5 seconds? We do it because recordable versions are ordered by `created_at` times. So if these were created at the same time, the system would kind of get a bit confused. That's not something that's going to happen in real life — in real life, what is going to happen is you post a document and edit it at least a second later. So this is a fine way of doing that.

### No Mocking, No Stubbing

What you will also notice here is that these are not unit tests in the classical, pure sense, because we are not stubbing anything out. These things all go through to the database. When we're recording this new document, it goes through the database, it creates the stuff in the database. There's a bunch of callbacks actually that happen in other concerns, and we just let those run. And it doesn't really matter.

Because first of all, it's fast enough. And I think it's better — as we talked about in the introduction — that you're testing the real thing, mocking and stubbing as little as possible, as you can get away with, such that your tests remain as close to the metal, as close to the real thing as I said, and as readable as possible — that they don't have all this accidental complexity of setting things up that aren't necessary for the test itself.

And that's also the reason why I like this idea of fixtures — that you have a base world that you can sort of interact with, and then you can do specific things. Like here, the new documents that we're recording for testing the versions feature — they're not fixtures, because they're something new, something particular to testing this feature.

### Running the Tests

Okay, so that's documents. Let's actually just — let's run that test. Let's go here and see the document test, just sort of run that. Because I want to show one thing — it actually runs pretty quick, right? Like, this is not that slow of a test.

Yes, not as fast perhaps as it could be. These four tests run in two seconds, so that's half a second per test. When you have hundreds and hundreds of tests, that's going to take some time. And there are ways you can get down to 100 milliseconds or less per test. But for me, that's not worth the trade-off.

Part of this is because we use this thing called the Spring preloader, which keeps a persistent process for your Rails application running in the background so you can run tests again — so it doesn't have to spin up and evaluate and instantiate an entirely new Rails environment with your application loaded every single time we run through tests. So they're pretty quick.

## Testing the Lockable Concern

But let's go back and look at the concern — and testing that concern. What we wanted to test was this idea of locking.

First of all, you'll see that the document test just lives over here under `test/models`. Here's the document test right here. And then the lock test lives over under `test/models/recording/lock_test`. Right? So that's where we structure our tests.

And the lock test is similar in many ways to the document test. It sets up first the current person who's going to do all these interactions, and then it also refers to fixtures in the same way. And it also doesn't care anything about all these other concerns that are present on the recording.

As you see — like, we have a lot of different concerns. It just doesn't matter. They're allowed to run as they please. And again, this might slow things down a little bit, but hey — look at this. This is a pretty long list of stuff that recording is capable of doing. It doesn't matter that all that stuff is there, is accessible, and in some cases adds callbacks and so on so forth. Because we just focus on this one aspect — we just focus on the locks when we want to test locking.

It's really not that complicated.

So as you can see, what these locking tests are is just basically a handful of them, and they're trying to test this very simple behavior which really is just a nice wrapping of the DSL around the single association of the lock.

But nonetheless — same story: straight usage of fixtures, straight usage of these models, straight usage of just this consideration for the lockable concern and for nothing else. Really.

And if I go back over and take a stab at running those, you'll see that that also actually runs pretty fast. Again, you have four tests that run in about a second, or half. Granted, this is a relatively fast computer, but you're not gonna get that different results. These are all currently, at least, single-core tests. So you only get to use a single core on them.

That's changing in Rails 6, by the way — just had an announcement about that. Going to do parallelized execution of tests. Not that you really need it for this case, but when you run a larger suite, that can kind of make sense.

## Assertions: Keep It Simple

So that's the model. We have our document, we have our lockable concern that's applied through the recordings via the recording object to documents, and we've tested both of those things. We found the tests run pretty quick, that they're easy to write.

Something else you'll notice here is most of the assertions that we write are extremely simple. I basically prefer to use only two different assertions: I use `assert` and I use `assert_equal`. And I can make everything pretty much happen within that.

There are cases where we extract sort of domain-specific assertions if they need a lot of setup or complicated things to test. But in terms of the stock setup assertions when it comes to model testing at least, I prefer to stick to just these two.

## Controller Tests

All right, well let's look at the controllers — how controller testing works, how controllers work with this specific feature.

### Documents Controller

You can start with the documents controller, which is the way we create these document recordables. And let's just focus on one method here — the create method. As you can see, we're recording a new document on the bucket, and the bucket is then giving us a recording back. That is almost identical to the tests we ran here in the document test — that's what's happening in the controller as well. And then we redirect to either the subscriptions or the recordable URL. This is this concept of sometimes you want to be prompted right after you record a new document as to who should be alerted about this document, or should we go straight to the recordable. Really not relevant for this discussion here.

### Testing Document Creation

So the way the tests for this work — they're even more sort of integrated. Actually, let's use that word: controllers are even further away from this idea of being a unit test, where we just test a controller and nothing else.

You could just test this create action and you could stub out everything and you can make that controller test run really fast. It wouldn't test a whole lot, and wouldn't test whether the pieces actually work together. But it would run fast.

Well, I prefer to have them be more comprehensive, even if it then again means that they're slower.

So let's look at this notion of testing the creation of a new document. The first thing I test is actually not the create action — it's the `new` action. Which, on the controller side of things, doesn't really do much. It just assigns this instance variable to a new document.

But what it does on the view side of things — it renders a ton of things. It renders the application layout, it renders the specific show template. And that stuff involves a bunch of different things that I really want to know whether they're working enough, right?

So before I'm creating the document, I'm asserting that the form we're using to create that document also works. And that does take some time, as I'll show you in a second. These tests do run just slower because not only do they render the whole view, they also actually compile all the assets that we need to present those views, so that we can see whether everything is working or not. So we'll compile all the JavaScript stuff and the CSS and so forth, such that we get the entire view.

This is really not lying when I say that there's inheritance from an integration test. This is a kind of integration testing. It's stopping just short of using the browser to drive the interaction, because using the browser to drive the interaction adds sort of an order of magnitude of delay to it.

I think it's still a great thing to do when you need to test JavaScript behavior that you can't test this way. Here we basically just deal with HTTP requests and responses and interrogating the HTML that we get back. We can do that all programmatically without involving an actual real browser. And when you can, I think that's a good idea and it's a fair trade-off.

Yes, you could have written a real integration test here that ran through a browser. But it would be overkill, because we're not really in need of testing any JavaScript-specific logic.

In any case, here we just assert that we're getting a proper response back (that means a 200). And then we have a custom assertion in here where we test whether the breadcrumbs are correct — the breadcrumbs are just so you can see what it is, it's this thing up here.

And then we do basically the creation — we run through the creation. We post towards the controller with our full setup of everything we need — where this document needs to go, with the parent, blah blah blah. We then follow the redirect, because as you will see here, the redirect is this thing. And then in this case we're asserting that we end up on a page that's basically the new document — that the new document has an h1 that says "Hello World." That's it.

### Speed Trade-off

If we have a look — I just want to show you that this is definitely a different order of magnitude in terms of speed of running. Two tests here — this is slower. But I think it's a very worthy trade-off that the documents controller tests the whole stack of integration, all the way from the HTTP request to writing to the database. And okay, we spend seven seconds to run ten different tests. Fair enough.

### Multiple Assertions Per Test

The other thing you'll see here that might stand out to you is — if you've read something on testing, some people subscribe to the idea that you should only have one assertion per test. I don't ascribe to that idea at all.

I think you should have all the assertions that you need to feel comfortable about testing one aspect of what you're doing. In this case, the smallest thing we can test when we're testing a controller as a whole is a whole action, and we should test the whole thing. So that ends up being three assertions for this specific test.

And even with the model tests we were talking about before, in many cases we have multiple assertions. Most of these had two, and this one, for example, had — what is that — four. There's no upper ceiling here to how many you should have. You should be able to have as many assertions as you need to feel comfortable about having tested that one aspect in depth and in full.

### The Locks Controller

All right, let's have a look at the locks controller. The locks controller shows some of the power we have of this recordings/recordable pattern — the fact that we can implement a generic locks controller that could have worked not just for documents but could have worked for to-do lists or any other piece of content in the system, and acts in a generic way. It shows the power of using concerns through generic objects to apply them to concrete objects very nicely.

And as you can see, it's also a pretty simple controller, right? Like, the whole interaction we're interested in here is this notion of being able to show if something is already locked and providing the ability to force-unlock something that's already locked.

### Testing the Locks Controller

If we go to the tests of that, it goes through all these scenarios that we would want to test. And in that case — first, that there will be a lock if we've already locked this by someone else.

So you can see here, we lock a recording that we fetched out of the list of fixtures that we have. Again, doesn't really matter what this fixture is — we're just testing the locking feature on it.

We assert that if you then access that document after it's been locked by Jason — remember who's logged in. It's David here. Here we have a slightly different way of setting up the current user — a setup method called `sign_in` that sets up a few things including the current user that we're using for integration tests. But just think through when you're reading through this thing: this is David accessing a page that was locked by Jason. And what it'll say is that this document's locked.

Now, if you're accessing your own page, it'll say that you're already locking this page. And then we test the three different directives — things were not locked — and so forth.

Here you'll also see there's several assertions per test. Actually, I think the average here is — what — two.

## Fixtures

I mentioned these fixtures — let me just show you that. So there are only four different documents that we use as the basic fixtures, and that's really all you need in most cases. This is a specific type of content, so you don't need that many.

When we go to the recordings — which is basically every piece of content in the entire system has a corresponding recording — well, we have quite a few more. So when we're doing the lookups here, looking for the recording for the introduction document, there's a few more of those, right?

So we can go back here and see the introduction document — it belongs to a bucket, recorder, blah blah blah blah. It's just all set up for us so that we don't have to do that busy work and we don't have to read through that when we're reading through the tests.

That's basically it — yep. That's a testing setup for one generic feature as applied to a concrete class in Basecamp 3.
