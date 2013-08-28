---
layout: post
title:  "Mobile Usability Tests, Quick and Cheap"
tags: ["usability", "ux", "testing"]
---

While many small startups at least pay lip service to unit testing, many neglect usability testing altogether. Which is unfortunate, because it can be a simple process. One designed to overcome a simple problem: Your *curse of knowledge*. [Wikipedia defines this](http://en.wikipedia.org/wiki/Curse_of_knowledge) as:

> "... a cognitive bias according to which better-informed people find it extremely difficult to think about problems from the perspective of lesser-informed people."

Not only are you merely better-informed than your customers, but as the author of the application, you are the *best-informed*. And the new users that you need for traction are not merely lesser-informed, but if they download it blindly or through a terse advertisement, they may be the *worst-informed*. You are at completely opposite ends of the knowledge spectrum. Usability testing is where these ends meet, face-to-face.

## Quick and cheap

[Steve Krug](http://www.sensible.com/) did a huge favor to everyone interested in usability when he wrote [Rocket Surgery Made Easy: The Do-It-Yourself Guide to Finding and Fixing Usability Problems](http://www.amazon.com/Rocket-Surgery-Made-Easy-Do-It-Yourself/dp/0321657292). If you're in this camp, but haven't purchased and read this book, then you should do those things now. His insight is not to be missed.

His setup for performing the test 

Now, if you have the time and resources to do all this, then do so. And as Steve says:

> "If you can afford to hire a usability professional to do your testing for you, do it."

Microphone, speakers, screen recording, screen sharing, an observation room, a list of tasks.

The weeks or months you have spent developing the application has instilled in you knowledge about its workings that new users must recognize and absorb in mere minutes. If this does not happen, they may ignore your application forever.

### Finding someone

You don't want someone who 

### Performing the test

One of you will play the role of a *therapist*. The other one of you will be the *scribe*.

The therapist is the one engaging with the subject.

The scribe writes down everything the subject gets right or wrong on every screen.

* Tell them that there are no right or wrong answers.
* Tell them that you want their honest assessment. If they are surprised or confused by something, speak up. Tell them that your feelings will not be hurt.

All the test subject needs to do on every screen is answer two things:

* Describe what this screen displays. If they came here from pressing a button, does it match their expectation? 



You must maintain neutrality. If they find something confusing, don't fire back with how it's "supposed" to be view. You have a complete mental model and the curse of knowledge. Clarify things in a neutral tone, such as "We were thinking that ...", and move along.

Start by explaining what the app does in one or two sentences, similar to a quick description they'd read on the App Store or Google Play. 

### The review

TODO

### Finding the biggest mistakes

You may think that in order to find your biggest usability mistakes, you must lay out the notes for each subject side-by-side and highlight their similarities. But truthfully, by the third test, your biggest usability problems will be obvious. When the test subject navigates to a screen, you'll already know what usability problems lie in wait, and you'll  stumbles over them.

### Iterating

After you've performed one round of usability tests, don't stop. Fix the biggest mistakes, build a new version of your application, and perform another round. You should always follow fixing usability problems with another round of tests. This ensures that the fixes address the intended problems and don't create newer, larger ones. Fixing "trivial" problems without following up with a test may seem tempting, but avoid it. Err on the side of caution instead. If usability problems and their solutions were so trivial, then you would have caught them all beforehand, and the first round of tests would have revealed nothing.

If the second round goes well, it will only reveal the smaller, ignored usability problems from the first round. But if any of these problems are really worth caring about, then you can fix them and perform another round of tests afterward. Iterate until you feel comfortable.

### Usability tests for ReadyUp!

Despite [ReadyUp!](http://www.readyupapp.com/) never taking off, I wager that the usability tests were a success. We performed two rounds of testing with four users each. In retrospect I doubt that three rounds of usability tests with three users each wouldn't have been much better. I'm convinced that we could have performed two rounds of testing with three users each and reached the same conclusions.

### Usability tests for Burner

When we tested the usability of [Burner for Android](https://play.google.com/store/apps/details?id=com.adhoclabs.burner), one problem repeatedly arose: Subjects whose phone had a hardware menu button never thought of hitting it to reveal additional options for the displayed screen. Therefore the subject never discovered options for renaming burners or conversations, toggling rings, recalling the promotional code, and so on. If the subject's phone lacked a hardware menu button, then Android automatically displays an overflow button on the right side of the action bar, and pressing this button reveals a popup menu with the additional options. These users never had a problem discovering the additional options.

(All new Android phones omit a hardware menu button. This is because while it's always available, it may not reveal any additional options when pressed by the user. After a few times of the user pressing the button to reveal no options, the user may be conditioned to press it fewer times or never again. In the language of Donald Norman, the menu button is a [permanent affordance](http://en.wikipedia.org/wiki/Affordance) that may do nothing. I liken it to playing [Wolfenstein 3D](http://en.wikipedia.org/wiki/Wolfenstein_3D) when I was a child, where the only way to find the secret rooms in the game was to run along every wall pressing the "use" button until one opened up.)

TODO: action bar overflow

After some research on [Stack Overflow](http://stackoverflow.com/), I came across a [two-part solution](http://stackoverflow.com/a/13568024/400717) that forces displaying the overflow button on the action bar, even if the user's phone has a menu button. In this case, pressing either the menu button or the overflow button reveals the popup menu with the additional options. While this solution actually contradicts the Android usability guidelines, we thought that this redundancy for presenting the additional options created less confusion than not being able to find the additional options at all. In the second round of usability testing, the subjects had no problems finding the additional options, and were unaware of the redundancy because they never pressed the hardware menu button anyway.

## Conclusion

Usability doesn't have to be time-consuming, difficult, or expensive. If right now you have TODO.

