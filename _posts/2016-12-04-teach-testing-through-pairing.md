---
layout: post
title: "Teach Testing Through Pairing"
date: 2016-12-04 09:32:00 -0400
categories: teaching
---

I have [written before]({% post_url 2016-12-03-junior-devs-never-learned-how-to-test %})
about junior developers and why testing, in my opinion, is the skill that they
tend to lack.  Fortunately, honing a junior dev's testing savvy provides an
excellent pair programming opportunity.

I also have some guidelines to help make the pairing exercise as productive as
possible.  Please note that these guidelines should not apply to _all_ pairing
sessions.  This particular type of pairing is very much a teaching exercise, and
should therefore be handled slightly differently than normal pair programming.

{: .ordered-list }
1. The junior dev should drive; this ensures that the pairing session goes at a
   pace suitable for the person doing the most learning.

1. The junior dev should approach the session with a blank slate.  Be open to
   anything and everything that your senior dev has to offer.  You can evaluate
   what you like and dislike about their approach after you have some testing
   experience of your own.

1. The senior dev should eliminate any potential distractions.  For me, this
   means that my laptop should be closed, and my phone remains at my desk.

1. The senior dev should be ready to explain everything, but answering a question
   with something like, "Let's ignore that for now" is okay.  Answering, "I don't
   know" is also okay; in fact, it may put the junior dev's mind at ease.  He
   or she may realize that they don't have to know everything in order to write
   sufficient tests.

1. The tests should be written for code that the junior developer has already
   implemented unless your team exclusively practices TDD or BDD.  It can be
   overwhelming to push a new software methodology and new testing techniques on
   a green developer, so stick to writing the tests.  TDD or BDD can be pushed
   later.

1. You may find opportunities to refactor some code as you write the tests.
   Good!  Do the refactoring, it will provide perfect insight into how to write
   more testable code.

1. The tests should be approached as methodically as possible.  I prefer to write
   tests from "outside-in" starting at the request or controller layer and
   working my way down to the model layer.  Working "inside-out" from the models
   up to the controllers or request layer works, too.

1. Give the pairing session as much time as necessary.  If it means you burn an
   entire morning or an entire day teaching or pairing, that is perfectly fine.
   Learning to write complete tests is one of the most valuable lessons that a
   developer can learn.

![sad panda]({{"/images/sad-panda.jpg" | relative_url}}){: .centered }
