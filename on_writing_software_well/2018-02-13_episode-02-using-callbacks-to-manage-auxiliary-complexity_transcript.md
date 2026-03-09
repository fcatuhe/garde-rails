# On Writing Software Well #2: Using callbacks to manage auxiliary complexity — Transcript

**Author:** David Heinemeier Hansson (DHH)
**Published:** February 13, 2018
**Series:** On Writing Software Well
**Episode:** 2 — Using callbacks to manage auxiliary complexity
**Source:** <https://www.youtube.com/watch?v=m1jOWu7woKM>

---

Welcome to episode 2 of On Writing Software Well. I am David Heinemeier Hansson and this is a follow-up to yesterday's pilot episode that I did on a couple of small changes in Basecamp dealing around comments. You can look for that episode in the playlist if you want to go back and check that out.

But the response to that episode was great. I wasn't really sure whether anyone would have been interested in watching 10–15 minute videos on diving into sort of, in some cases, my minute code considerations. But it seems like there was, so I'm gonna keep going for a while — at least as long as I still have the motivation to do it, and right now I do.

So today we are going to dive into something a little more meaty than what I showed yesterday. We basically talked for what, 10 minutes about changing one line of code yesterday. So let me show you through something a little bit more substantial.

## Callbacks and Side Effects

This is an episode about callbacks — about tracing one callback rabbit hole all the way down and seeing what that does to the code. I'm a big fan of callbacks. I think callbacks allow you to take incidental complexity and considerations and move them sort of off to the side, such that most of the code can just pretend to be in a simple default path. And then when you're up for it, you can look into the more specific details of something that happens as a side effect.

Side effects really is just another one of those words that in some circles has gotten also a bit of a bad rep, right? Like functional programming and so on — it's all about not having these side effects. Well, I think side effects are wonderful for a whole host of uses. And I love having side effects that, as I said, sort of go off and adorn or progressively enhance a main flow that is then not considered or not polluted with that complexity.

So that's what we're gonna look at today. We're gonna try to trace down one specific feature at Basecamp all the way down through all the callbacks that happen. So I hope you enjoy the show.

## The Mentions Feature

So the feature we're going to look at today is the mentions feature. Basecamp 3 — basically you can @mention someone who's already on a Basecamp project and that person will get a notification. The notification can either be an in-app browser-based notification, it can be a native push notification, it could be an email notification. We have a host of different sorts of notifications that we send out. But that's not really so important.

What I want to focus on today is sort of this callback flow — how we find those mentions and send something out.

### The Messages Controller

So what we're looking at here is the messages controller. Messages is sort of one of the main forms of communication in Basecamp, and this is the create method. The create method uses this recording bucket setup that I'm going to dive into more in a future episode. It's a really interesting pattern — I think actually it's the predominant and most powerful pattern that we have in Basecamp 3. But it's kind of a long episode. I want to think about a little bit of how to present that and talk about it, so we're gonna save that for later.

For now, just think about this as though we're creating a new message. It belongs to some parent recording — a specific message board. It has a status which can be draft or it can be active. There's a number of default subscribers or additional subscribers — people who want to know about this. And messages can also have categories.

None of these things really matter for what we're trying to figure out here. But I just want to show you that in none of this code, when you read this controller, is there any talk about mentions. Right? All we do here is we record the new message. Which — let's take a look at that — the new message down here is basically just a subject and the content. Right? These are the things that we allow to be created as part of the new message.

Now, as I said, this doesn't mention anything about mentions. It doesn't say anything about how these things are recorded. All those considerations about how we do that exist somewhere else.

### The Mention Model

And what we try to create is one of these — or several of these, actually. In fact, it's a mention. And a mention is something that belongs to a specific recording. One of these recordings — the one we're looking at in this case is a message. And there's the person who mentioned someone and then there's the person who got mentioned.

And I kind of love to come up with these specific words — "mentionee" and "mentioner" — to describe these things. These are — I don't know if "mentionee" is even a real word. I don't know. But it doesn't really matter. I think the wonderful joy of programming is that we get to invent words sometimes. We get to invent concepts. And as long as we just package them up and they're consistent and coherent, we have the freedom to do so. Wonderful, right?

In any case, a mention is created as I said — belonging to a recording, someone mentions, someone is mentioned. And it's the mentionee who gets the notification. So that's really what we're trying to do. And a mention has a graphical representation in Basecamp — you see the little avatar and you see the name. But what really matters is that the mentionee gets a notification, right?

So here's the class that basically encapsulates that. Mmm, not really terribly interesting. There's not a lot going on. What you can see though is that there's a delivery going on — so there's a callback that, once this mention has been created, we're automatically going to send it out to the mentionee, unless the mentioner is the same as the mentionee — which is sort of one of those little wrinkles. We don't do it in that case. But otherwise we do.

### How Are Mentions Created? — The Mentionable Concern

But how do we create these mentions? That's really what's, I think, the interesting point. And mentions are a concern — a concern of recording.

Recording — maybe I should just mention — is this broad encapsulation of all the different types of content we can have in Basecamp. So a message could be a recording, a chat line can be a recording, a to-do item could be a recording. And that's how we share a lot of these generic concerns between these different types of content that all act in the same way. And all of these pieces of content, whether it's a chat line, whether it's a to-do description — they can all have mentions. And this is a way we get to apply this logic to all of them.

In any case, what I want to focus on here is basically this callback train, right? So we're mixing this into the recording, which means that every single time we save a recording — which, as you remember, is what we did here: we'll do a `bucket.record`, it'll save the new message, create a recording for it — and this recording has its concern for mentions that's going to eavesdrop on the content.

### Eavesdropping

And it's the eavesdropping part that I really find interesting and fascinating to dive into. First of all, I love just the domain language around that. "Eavesdropping" is just such a perfectly delicious term to use for something like this — that you have this system sitting on the side, listening for new recordings, listening for mentions in those recordings, and then acting upon those mentions.

So the first thing to notice here is a little bit of a trick that I've tried to extract into Rails several times but been unable to really find a good way into it, which is the notion of tracking dirty attributes — whether an attribute has been saved or changed in the last update. We have these neat sort of automatic methods for it. For example, `changed_from_drafted?` which is a method that checks basically whether an enum that we have for status changed to the drafted status. Right? We're given this by the enum definition.

And to use these change-set methods, you have to still be within the original transaction. Once the transaction is closed, we clear out the dirty change set and you can't really query it in the same way anymore. So by using this `after_save` callback, we basically take a look at these change sets and then we just make a little note. And that little note happens as an instance variable here called `@eavesdropping` — to remember whether we need to eavesdrop.

Because the thing with eavesdropping is that it has to act on a saved object, right? Like, this has to exist in the database, which means it has to be after the transaction is closed, which means it has to be in an `after_commit` hook, right? And as I just said, once the commit has happened and we've wiped all these things out, we can't query about these things anymore.

Anyway, no really — none of that matters. Just think that we're setting up the decision on whether to eavesdrop or not in advance. This is really kind of an optimization actually. We could have just eavesdropped every single time. But scanning the content if the content didn't change doesn't really make a lot of sense, right? There could be some other change — there could be, for example, a comment could touch this message recording when it gets added, and that triggers a new save. We don't want to eavesdrop in that case because the message itself didn't change, the content didn't change. So that would just be wasteful.

In fact, that's how we launched — we launched with that wasteful version. And then I forget how it popped up, but somewhere along the lines we just realized, oh, we're doing a lot of work that we don't need to do and it's happening all the time, so let's not do that. So that's where `remember_to_eavesdrop` comes from.

And that's basically just looking at: what are the conditions upon which we would want to eavesdrop? Well, if there's an active or an archived recordable — recordable is the concept for messages, to-do's, and whatever — if they have changed, if the message in this case has changed and it's active or archived, we want to eavesdrop.

We also want to eavesdrop if we had a draft that became active. You can make a draft message in Basecamp, but that's supposed to be secret, right? So if you @mentioned someone in that draft, we don't want to alert or notify these mentionees once the draft is saved, because we want to wait until the draft is published to do so. So that's what `draft_became_active?` is all about.

And then we basically have a check here about whether we should be eavesdropping at all. There's another few ways of sort of getting out of this. This is first of all the `@eavesdropping` instance variable we set up here in `remember_to_eavesdrop`. But there's also this notion of "suppressed," which I will come back to — which is a really key concept for dealing with heavy callback scenarios.

### Callbacks + Jobs

Because the thing about callbacks is you want to use them on the default path — you want it to happen most of the time. But there are scenarios where you don't want them to happen. And I think perhaps that's what has bitten people in the past — that they didn't really consider how they could opt out of these callbacks. Well, we have a concept for this called "suppressed" that we will look at at the end.

Anyway, we're eavesdropping. So if we start looking for mentions — and this is another interesting point here — I love using callbacks together with jobs. You basically have this body of work that you need to perform. In this case you have to scan a piece of content for mentions in order to create these mention objects that then get delivered and so on so forth. But that work does not have to happen at the same time that we save the message, right?

Like, if we go back here — we do the record of the new message. It's not a terrible amount of work that happens here, it's just a few savings to the database. But if we start scanning the content every single time — well, we're sort of just holding this up, right? We're holding up this line until all the work here has been completed. We can't respond to the user and send them on their way.

Now if we take some of that work that is not required to happen at the same time as the recording of the message and shove it up on basically a different queue and have it happen in a job, then we're doing less work that's blocking. Which mentions are a great case for, because the whole reason we're doing these mentions, as I said, was to notify the mentionees. Well, whether they're notified the second something is posted or a second and a half later doesn't really matter. But it makes a big difference in terms of the workload that we have to perform within the scope of a single request.

### The Eavesdropper

So we're setting up an eavesdropping job. Really, that's what this mentions concern is mostly about. If you see, what is the work here actually being performed? It doesn't actually do the real work. What it does is it checks: should we be doing the work? Obviously. It sets up the association to mentions once the work is done. And then it kicks off a job to do the work.

That job to do the work is this eavesdropping job, which really is a pretty thin shell too. We don't actually do the work in the eavesdropping job itself. We have a dedicated class for this called the eavesdropper, which we just seed with the variables that we get from queuing this job. And we create these mentions.

So the real work itself happens down in the eavesdropper.

So this is already interesting, right? Like, we start here in the messages controller — we record a new message, no mention of these eavesdropping things going on. Then we have the mentionable concern that says, "Oh, let me just check this new recording that's come in. Do we need to do the eavesdropping? Do we not need to do the eavesdropping? If we do, let's enqueue a job to actually run the code."

So this is one of those things where you're like, "There's a fair amount of indirection here." But it provides a very clear path of reading what's going on in the messages controller. Most of the time you're not really concerned about this auxiliary complexity of the mentions themselves, right?

Anyway, let's have a look at the eavesdropper itself. The eavesdropper is a class that takes a recording and a mentioner. The mentioner is basically just the person currently doing the work. I think we're getting the mentioner all the way back from here — from `current_person`, which is another one of these interesting points.

`Current` is this new concept in Rails 5.2 — or did we already have it in 5.1? I forget — a concept to basically make globals pretty. And globals is another example here of a concept that needs to be treated with some respect, right? Like, globals is not something you should just litter all over your application. But the notion of a current person or a current account is something you need to keep using over and over again. Passing this stuff around all over the place isn't actually that helpful.

In this case we do need to pass it because we're issuing a job which won't be in the same request as the current person has been defined in, so we need to pass it along. I'll do another episode, I think, on `Current` and the idea of globals and how they can be helpful — because I think it's a bit of a misunderstood and overly feared consideration in a lot of forms of web programming. Just like callbacks. Actually, yes — we talk about writing one thing at a time.

### Scanning for Mentions

So eavesdropper here — where the actual mentions are created — it's basically this `create_mentions` method that's called straight from the job. See here, we're instantiating the eavesdropper then calling `create_mentions`, which relies on a method called `mentionees`, which is where the real scanning is happening.

And here you can see the rabbit hole keeps going, right? Like, does the eavesdropper that's going through and creating these mentions actually also need to have the work to actually scan the content itself? I don't think it does. So we have another dedicated method just for that — just for extracting out the mentionees from a recording.

There's been a consideration here — we have different kinds of recordings. Some recordings just have plain text and some recordings have rich text. And there's two different ways of extracting these mentions out. Doesn't really matter that much. What we want to do here is we want to find these mentions in either form of content.

In the rich text, these contents are already encapsulated because when you do these @mentions, we have some JavaScript that creates a little bit of HTML and you extract it out from that. When you're extracting call signs straight out of plain text content, you're doing basically a regex scan. Right?

Anyway, doesn't really matter. When you're reading to understand what the eavesdropper does, all you basically have to trust is the fact that we have a scanner. It can take a recording and that scanner can extract some mentionees. And then once we have the mentionees, we have the recording, the mentioner, and the mentionees — so now we have all the three components that we need to create mentions with.

So this is what's happening in `create_mentions`. We have a recording with mentions and we `find_and_initialize_by` mentionee. We only want one mention per mentionee. Set up everything and save it.

### Delivering Mentions

Now, as I said, we've created one of these mentions. Now, when that happens, the work is still not done, right? Now we have the domain model kind of set up. We've scanned the recording, we have a bunch of mentions. We need to deliver these too, right? That's the whole purpose — that's why we're doing this.

So there's another `after_commit` deliver hook here that triggers on both create and update, that then runs the mention delivery. And the mention delivery takes two forms. We have a notification — which is yet to be extracted as a framework within Basecamp 3 that I've been calling Action Notifier. It's a framework for dealing with notifying people through web notifications in the browser or native notifications on both Android and iOS. Anyway, this is another encapsulation. And of course we have mail — missions. And this is just a standard Action Mailer that sends out an email notification to people.

### Suppression

All right, that's sort of a lot of weaving back and forth here. I hope that makes some sort of sense. I want to come back to the point we mentioned at the beginning, which was the notion about suppression.

I have a class here called the project copier, which is a way of copying things from an earlier version of Basecamp into the current version of Basecamp. And as it's doing this copying stuff, it's creating a bunch of new recordings that may have mentions. But we don't want to actually notify people — we don't want to create these mentions on all this copied content.

So what we do, as we're doing these copies of uploads and documents and whatever, we suppress a bunch of things that happen. We basically suppress a bunch of callbacks. And one of the callbacks we want to suppress is the idea of eavesdropping, right?

So that's what this line does. The eavesdropper just yields a block that, within that block, anything that would have called the eavesdropper to trigger will just get suppressed and essentially ignored.

That's all coming from a concept — what do we have? Yes, in the eavesdropper we have `suppressible`, which basically just sets up this variable to say whether something should be suppressed or not. And then we can skip doing the work if we don't need to do it.

## Wrapping Up

So that's one example of how a single feature — mentions — works from the perspective of callbacks and hooks. And I hope it's clear. I don't know — I mean, an easier way to make it clear would be to take basically all this logic that we have — the eavesdropping, the scanning, whatever — then jam it into what? The controller? A service object? Some other form of transaction script?

I don't think any of these things get clearer if you do that. The fact is, mentions is an auxiliary concern. It's not the main purpose of what we're doing here when we're setting up new recordings. So it deserves to be on the side. It does not deserve center stage. But of course it needs to be there, and we need to be able to dig into it when we want to — which is where these concerns are just wonderful.

We don't have to burden the recording class itself with all this logic and all this setup. We can treat this whole aspect as one cohesive unit, even though it's mixed into a class, even though recording gets a bunch of new private methods. It doesn't really matter. When we're reading through the code and trying to follow the flow of it, this is a much easier way to read through things.

So that's it. I hope you enjoyed it, and I will talk to you in the next On Writing Software Well.
