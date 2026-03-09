# On Writing Software Well #3: Using globals when the price is right — Transcript

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 15, 2018
**Series:** On Writing Software Well
**Episode:** 3 — Using globals when the price is right
**Source:** <https://www.youtube.com/watch?v=D7zUOtlpUPw>

---

Hey, this is David Heinemeier Hansson. This is episode 3 of On Writing Software Well. Today I'm going to talk about `Current`, which is a new feature in Rails 5.1 that we've been using in Basecamp for a long time. We only just extracted it into Rails 5.1 after I actually saw a bunch of discussions around globals — not just in Ruby and Rails, but in other communities — and how, as we all know, globals are considered harmful. Right?

Mmm, not so fast.

## The Case for Globals

I think globals have a great place in all sorts of applications, and especially web applications, since a lot of these — all of these — operate within a request cycle where there are just certain things that you've set up at the start of the request that you really would like to have easy access to throughout the request. And that's really sort of the whole lifecycle of the application.

And using globals for these things is a wonderfully good fit. Of course, you can go crazy and you can start using globals everywhere, or too many of them, and it can turn into a big mess. But just because something can turn into a big mess does not mean that it's not something we want to use when it's the right fit.

In Ruby on Rails, we have this idea of providing sharp knives — that it's not a sin of a framework or language to provide you with really sharp knives that could cut your hand off if you use them wrong. But they can also provide just the incision that you need — the precision that you need to make something happen in an easy and readable way. So that's kind of how I feel about globals.

## The Current Class in Basecamp

But let's take a look at how we use globals through `Current` in Basecamp.

So this is the `Current` class that we have in Basecamp. As I said, this was actually something we've been using for many years since the inception of Basecamp 3 in this codebase. But it was only recently that I extracted it out into something that now lives in Ruby on Rails.

It's really a short little class. It doesn't do that much. It's really just dedicated to declaring a handful of attributes that you want to be available through this `Current` class — which is basically our word for globals in this context — and then providing you with a way of delegating to these setups and a hook for what should happen when you want to reset the current after a request is over.

And as you can see here, we really just have two main classes of things that we set as current. We have the current account — which we derive from the URL. Basecamp 3 uses the account ID as part of the path. We take that out in a piece of middleware and stick it into — or do a lookup and set that lookup to `Current.account`.

The one I want to focus on a little more today is `Current.person`. Current person is what's being set when you log in — either explicitly through a form, or through OAuth, or through a cookie, or through any of these other means of authentication that we have.

And then secondarily, we have the request details that are set as part of a callback in the application controller — a concern through the application controller.

And as you can see, we actually do a couple of things when we set this current person here. If the account hasn't already been set — which happens sometimes in jobs that are simply passed a person that should act as current — in that case, then we set it through this overriding of the person accessor. The same thing with the time zone — the time zone is something we record on a person, so people can have their own time zones and we use it for that.

## How Current Flows Through the Application

Let's have a look at how this all flows through. First, this is our application controller in Basecamp. It has a lot of concerns mixed into it and zero other code. We've factored out every piece of consideration in the application controller — which is really just a root controller for all controllers, pretty much, in Basecamp 3. One of the things that they all need to do — and all of these considerations can be encapsulated into concerns.

We're just going to focus on a couple of these today. We're going to focus on `set_current_request_details` and we're going to focus on `authenticate`.

### Setting Request Details

First, if we have a look at `set_current_request_details` — you see it's a very simple concern. When it's included, it just adds a `before_action` to the application controller, and thus to all controllers that inherit from application controller, where we set these globals based off the current request.

That now means that anywhere in the application we have access to — within the request cycle — the request ID (which is a unique identifier that we use to tie things through logging and so forth), the user agent, and the IP address. These are all used for various forms of logging. And I'll show you how this actually ties into our use with the current person in just one second. That's pretty simple.

### Authentication

The one that's not so simple is the authentication concern. So let's look at that. Here we authenticate through a bunch of different ways, but the main path we're going to look at right now is cookie authentication — which is basically someone who's logged into the application and then is able to access protected pages by navigating through the application, or closing their browser and coming back to the application without clearing their cookies.

So the main thing that this routes through — again, this is a concern, it's mixed into application controller, thus all controllers that inherit from application controller get this logic — the first thing we're gonna do here is we're gonna authenticate with full authorization. And as you can see here, we have two main paths that we're going to check on when we do that. First, we're going to check on OAuth — that's not the one we're gonna focus on right now, so let's leave that aside. And then we're gonna check on cookies.

And if this call returns true, then great — we're in. The request is authenticated and things can proceed. If that hasn't happened, and it's not because we've already sort of sent you somewhere else (the OAuth process requires a back and forth), then we're going to request that you supply API authentication — generally with a `401 Unauthorized` response — or we're going to request cookie authentication, which requires taking you back to a login screen.

### Cookie Authentication

Cool. Let's have a look at what actually happens in the cookie authentication setup. As you remember, over here when we do `authenticate_with_full_authorization`, we call `authenticate_with_cookies`, which is provided by this other concern — a concern of a concern — called `ByCookie`.

And in here we're gonna do two things. We have two different types of cookies — we have an identity cookie and a remember-me cookie. This ties into a shared authentication system that we have for all the applications at Basecamp — whether you use Basecamp or Backpack or Highrise or whatever. We have an identity system that's shared between these things.

But let's focus on the identity cookie here. It's all the same. We have this identity ID that's hidden in a signed cookie, which means that it can't be tampered with on the user side. But we can use it to look up the verified user. And once we've looked the verified user up, we're going to call `authenticated_user_by_cookie`.

Basically we're gonna say how this person was authenticated and set by cookie. This is a shared `authenticate` action that chooses whether someone authenticates through cookies or OAuth or whatever — it's kind of the same thing that happens. And as you see here, we record in which way the authentication method happened — whether it was by cookie or whatever — and we basically just use that for logging, to say how someone authenticated.

### Setting Current.person

Here's the more interesting point — the whole reason I'm going through and showing you all this. This is not an episode about authentication. It's this line — that by the time we've gone through this whole invocation path, we end up setting `Current.person` to the person of the user that's logging in. And once we have this current person — just as with the request details — that's accessible anywhere in the application, to any class, at any time. It's a global, right?

## Using Globals Deep in the System: Event Tracking

So what can we use these globals for? And I'm going to show you another concept of Basecamp — it's something I'm gonna come back to in another episode and explain in more detail. I'm just gonna sort of glance over a bunch of the details here. It's event tracking.

Event tracking is a really powerful idea where you basically record the changes that are going on in the system in some other sort of class or some other tracking system, rather than just trying to deduce what the changes were by looking at the objects themselves.

The interesting part as it relates to globals here is that event tracking is something that happens as a side effect. In Episode 2, I discussed the side effects around callbacks through the implementation of the mentions feature. This is another one of these side effects. It's an auxiliary concern, it's on one of these concerns. Event tracking is something that's an auxiliary complexity — something that happens on the side, something we don't want to necessarily focus on all the time.

So we've implemented this in much the same way that we've implemented the mentions feature. It's a concern, it uses callbacks, and it declares an association. So you can see the structure here is quite similar. This is really a structure that we use for a lot of the concerns in Basecamp and in all our applications — that you have these encapsulated concepts, like mentions, like events. We implement these through concerns and we mix these concerns into a base class. In this case, the base class is Recording.

Recording is our sort of master metadata class for all the concrete classes that we have in Basecamp — whether it's messages or to-do's or documents or whatever have you. I'm also going to come back to that setup — recordings and buckets, the idea of using a concrete metadata class to encapsulate shared behavior — it's a really interesting one and it's probably the most powerful pattern that we have in all of Basecamp. So that's well deserving of a whole episode.

Anyway, I'm gonna sidetrack here, my apologies.

### Current.person in Event Creation

What I want to show you is how we use these globals and why they are so powerful as globals and it's not something we need to pass around all the time.

So let's focus on one flow here out of events — the fact that we track an event every single time a recording is created. We have this general `track_event` method that could be called for any kind of action — whether when something is created or when it's updated or other changes are made. And we have also such things for all sorts of actions and also two different kinds of events that we track in Basecamp 3.

But the primary one — the one that's tracked for pretty much everything — is the creation event. So let's go down to that. We call `track_created`. As you see here, after we've created a recording, we're going to track that that recording was created. That basically just calls `track_event` with the status of "created." I'm not gonna go into the status thing — we have an enum that tracks the different kinds of status of a recording. Not really important here.

What is important is to sort of flow down here and see what's getting passed on. The fact that when we create this event here, we're passing in the creator — not from an action up here, or not from a parameter to `track_event` — but picking it straight out of this global.

And the neat thing about that is, as you can see, `after_create` is a callback that doesn't take any variables. Right? Like, you don't have to pass anything into it to be able to call this. And the reason we don't have to pass anything into it is because we can just pluck this concept — this idea of the current person — out of the globals, because that was already set up.

And that's incredibly convenient. It would be a real annoyance if when we were creating a new recording, we had to specifically pass in, "Oh, by the way, when you track this event, the creator needs to be so-and-so." Here we could just pluck it out.

### Current Request Details in Events

Well, it gets even better in some cases when you look at the request details. When an event is created — this was what we did in `track_event` — we're basically creating one of these events that ties together a recording that belongs to a bucket and the creator of that event.

We also have this idea of the request details. These request details are set in sort of a hash called the `detail`. And this detail includes a bunch of different things that we want to track around the event — the recordable, and all sorts of things.

But what's interesting here is this `request_id`. That when we track or when we create an event, we want to know who actually created that event — not just the person who created it, but all those request details that we were setting up in `set_current_request_details`. These are the things we want to tie to every single event in the system, because it provides a really powerful way of doing basically sort of investigations after the fact.

Who changed something? In what circumstances? What was the IP? It provides us, through an admin interface for Basecamp, the ability to look up who created this thing simply just through an IP address — which is a really powerful tool for support, especially to try to discern what went wrong if a customer writes in and needs help.

### The Requested Concern

In any case, this `Requested` concern basically just says: an event has a request. And as we're saving a new event, before validation, we'll build this request. And this request basically just plucks out the aspects of the current that we have — the request ID, the user agent, and the IP address.

And what's interesting here is just how deep in the belly this is. Imagine having to pass this amount of detail — the request ID, user agent, and IP address — all the way from the top of the chain, all the way up from the controller action, down through the creation of a recording, down through the creation of the event, and so on. It would just be a real hassle.

## Guidelines for Using Globals

And as I said at the intro, this isn't something you should do for everything. Perhaps there is a temptation that, "Oh, wouldn't it be easier if you could just stuff something into a global and then you can access it everywhere?" And that certainly can be a smell — a code smell — that you haven't factored your code correctly and you're using globals as a crutch.

In this case, we're using globals as that sharp knife — that this is absolutely something we intend to do, that we've examined the alternatives and said, "Passing this stuff around all the time is actually not an advantage. It doesn't make the code clearer. It makes it more convoluted." And it takes such an auxiliary consideration as being able to look up an event by its request ID and so on, and puts it way too center stage.

This is a minor detail that enables us to do one specific thing — on-call and other forms of investigation. This is not the main thing you should be concerned about when you're reading the code of what's going on with event tracking. And certainly not if you bubble it even up further to what's going on when you create a message, right? You don't want to be bothered with these auxiliary details.

So that's basically `Current`. That's the power of `Current`.

I think this `Current` implementation is a good guideline perhaps to just how much stuff you should shove in there. If you end up with a `Current` that has 10, 15, 20 different things — yeah, okay, maybe you're off into the weeds somewhere and you should take a step back and see if there's a different way you can factor your code so that you don't rely on globals so much.

But again — globals, sharp knife, really useful for these precision incisions. Use them for that, and use them with pride. You shouldn't be ashamed of using globals for these kinds of things. But be mindful of sort of the drawbacks of it as well.
