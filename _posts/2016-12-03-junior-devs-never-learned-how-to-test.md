---
layout: post
title: "Junior Devs (Probably) Never Learned How To Test"
date: 2016-12-03 18:13:00 -0400
categories: testing
---

I find that, in general, junior developers have a difficult time
building their confidence in their ability to write "good" tests.
My kneejerk reaction was that junior developers simply find testing boring;
however, upon further reflection, I am convinced that is not the case.

As I recalled my undergraduate education, it occurred to me that I had written
maybe two tests in my entire undergraduate career.  4 years, two tests.  I
imagine that most junior developers experienced a similiar education.  There
are so many "important" things to learn to earn a CS degree that testing could
perceivably be deprioritized.

These shortcomings are not restricted to those educated at universities either.
I asked a colleague of mine who learned to code from a bootcamp, and he told
me that the bootcamp never taught testing.  The instructors had mentioned that
it was important, but still never took the time to teach their
students how to write tests.

So, if you imagine the mindset of a junior developer, freshly hired by your
company, you see a somewhat scary picture.  The junior dev is most likely
overwhelmed by the firehose of information regarding your codebase, your
technology stack, your system architecture, your developer workflow, the
team's culture, and the company's culture.

![Drinking from a firehose]({{"/images/drinking-from-a-firehose.jpg" | relative_url}}){: .centered }

So, as the junior dev takes on their first user story, kanban card,
or whatever, they just want to beat back their inevitable imposter syndrome and
prove that they are a capable, and intelligent developer. Inevitably, the junior
developer submits their code for review and one of the following scenarios
occurs:

{: .ordered-list }
1. The code has no tests.
1. The code has incomplete tests.
1. The code has complete tests.

For a junior dev to write complete tests is extremely rare in my experience.
Hell, it might even be rare for senior developers.

Notice, that I refrained from using the term "bad tests".  I do not like to call
tests "bad".  There are so many holy wars with regards to testing (e.g. testing
behavior vs. testing state, BDD vs TDD).  What one team might consider bad
another team might consider great.  It is more objective, in my opinion, to
call tests complete or incomplete.

Okay, so why do junior developers tend to omit tests or write incomplete tests?
Uh, probably because they have written little to no tests in their lifetime and
don't want to look incompetent or "stupid" in their first code submission.
Plus, they probably wanted to impress the rest of their team by knocking their
code out as quickly as possible and testing would have slowed them down
considerably.

So, how can junior developers build the confidence and skills to write tests
that adequately test their code?

In my opinion, a developer's testing skillset should be built through a
combination of pair programming with a more experienced developer and
trial-and-error.  Now, for trial and error to be successful, your development
team needs to achieve a culture of constructive feedback.  After all, nobody
will be willing to try and fail at something new if they are just going to be
torn to shreds.

I have some suggestions to help achieve a productive teaching session; you can
find those in my
[Teach Testing Through Pairing]({% post_url 2016-12-03-teach-testing-through-pairing %})
post.

Finally, as a piece of advice and encouragement to junior developers, your
senior devs will most likely never think that you are incompetent.  Thus,
if you are struggling with something, ask for help.  As senior developers, we
have our own user stories to finish and others' code to review.  We honestly
don't have the time to try and decipher when junior developers have hit a
roadblock.  That being said, senior developers understand that unblocking and
teaching junior developers are our most important responsibilities because
they hold the greatest value for the team as a whole.  In my experience, a good
senior developer will drop whatever they are working on to help a colleague;
that is ultimately how such developers became "senior".
