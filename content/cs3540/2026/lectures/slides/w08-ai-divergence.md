---
track: ai
week: 8
title: Reading the Divergence Report
subtitle: What It Means When Independent Builds Disagree
runtime: 18
---

NOTES:
Week eight, AI track, and this one is about your own words coming back to you with a number attached.

By now the scheduled agent has been building the specification independently, several times over, and comparing the results. Where the builds disagree, there is a report. Your Act Two response to that report is due on Thursday the twenty-second.

This lecture is how to read it, and — more importantly — what it is actually telling you, because the natural reading of that report is the wrong one, and almost everybody arrives at it within about four seconds of opening the file.
---

# What you'll know after this

- What the report **measures** — and what it does not
- Why a disagreement is **evidence about your prose**, not about a build
- The four questions to ask of a failing vector
- What a good response looks like, and what a bad one looks like

NOTES:
Four things. The second one is the reframe, and it is the one people resist hardest, so that is where I am going to spend the time.
---

# What the report is

![](conformance-vectors-divergence-report.svg)

NOTES:
Here is the mechanism.

The same vector runs against several independent builds of the specification. A vector pins the seed, it pins the commands, and it pins the tick count, so the only thing left that can vary is the implementation. That is the entire design. Remove every source of difference except the one you want to measure.

Three builds agree. One does not. The majority is promoted, versioned, and tagged, and that tagged engine is the one everyone's game runs on.

And then the loop closes. The failing vector belongs to a section, and the section has an owner. That is not a search and it is not an investigation. It is a lookup, and it has your name in it.
---

# It is a measurement, not a verdict

The natural reading: **"build D has a bug."**

The correct reading: **"the prose permitted two readings, and D took the other one."**

- Every build implemented your section faithfully
- They disagree because **your section allowed them to**
- A build cannot be wrong about an unspecified thing — there is nothing to be wrong about

NOTES:
Now the reframe, and I am going to be emphatic here, because the instinct is strong and the instinct is wrong.

You see three builds agreeing and one disagreeing, and the conclusion arrives on its own. The odd one out is broken. Three against one. Majority rules. That is how we settle almost everything else.

But look at what actually happened. Four independent implementations read your prose and every one of them did what it said. Three interpreted an ambiguous sentence the same way — probably because that reading is the more natural one, or the more common convention — and one took the other reading.

The minority build is not wrong. It is legal. It did something your specification permitted, and it did it deliberately, on the strength of your sentence.

The fact that it is outnumbered is a fact about which reading is more popular. It is not a fact about which reading is correct, because you never said which one was correct. There is no correct on the table.

That is why we call it divergence and not failure. It is a measurement of ambiguity, and the number of builds that agree measures how obvious your intended reading was — not whether you had one.
---

# Four questions for a failing vector

1. **What exactly diverged?** Which claim, at which tick, in which quantity
2. **What are the two legal readings?** Name both. If you cannot name the second, you have not found the ambiguity yet
3. **Which one did you mean?** Decide. This is a design decision, not a lookup
4. **What sentence forecloses the other?** Write it. That sentence is the fix

Question 2 is where the work is.

NOTES:
Here is the procedure.

One: what diverged. Not S11 failed. Which claim, at which tick, in which quantity. The hash usually diverges at one specific tick and you can bisect to it, so do that first and stop reasoning in the dark.

Two, and this is the whole job: name both legal readings. If you cannot articulate what the other build thought you meant, you have not found the ambiguity yet. You are still assuming it made a mistake. Keep reading your own sentence until the second interpretation shows up — and it will show up, because a build already found it.

Three: decide which one you meant. This is a genuine design decision rather than a lookup, and sometimes the minority reading is the better one and you should take it.

Four: write the sentence that forecloses the other reading. Not clearer. Narrower. Round becomes round half away from zero. The boxes overlap becomes overlap is non-strict, touching edges collide. Resolve each pair becomes resolve pairs in ascending order of the lower id, then the higher.
---

# Good and bad responses

**Bad**

- "Build D is broken." — you were not asked about the build
- "Fixed the prose." — which sentence, permitting what?
- "There was no real ambiguity." — the report says otherwise, and it is a measurement

**Good**

- Names both readings, picks one, quotes the **new sentence**, and adds the **vector** that would now catch it

NOTES:
What the rubric is looking for.

The three bad ones are the three I actually see. Blaming the build sidesteps the question, and nobody asked you about the build. Fixed the prose, with no detail attached, is unassessable, because I cannot tell from it whether you understood the ambiguity or reworded things at random until the vector went green. And claiming there was no real ambiguity, when a build demonstrably found one, scores zero — the report is evidence and your assertion is not.

A good response has four parts and fits inside a paragraph. Both readings, named. The one you chose. The exact new sentence, quoted. And the vector that would now catch this, because if it happened once it can happen again, and a fix with no test is a hope.

That last part should be familiar by now. Same standard as an anti-goal you can show firing, and a hook you have watched block something. A claim with no vector is an opinion, three weeks running.
---

# Divergence Response · Act II — due Thu Oct 22

- One response per divergence attributed to **your** section
- Both readings named; the sentence quoted; the new vector included
- If your section had **no** divergence: say what you pinned that others did not

Read `cheatsheet-conformance-vectors` and `cheatsheet-writing-a-spec-agents-can-build`.

Next Tuesday, Game: **Space** — the 3D pipeline.

NOTES:
Due Thursday the twenty-second.

The third bullet is for the few of you whose sections came through clean. That is a real result and it deserves a real answer. What did you pin that other sections did not? Go find a sentence in your section that names a rounding, an order, or a tie-break, and explain what would have happened without it. That is the same intellectual work as fixing a divergence, done in advance and for free.

One last thing, and it is the reason this assignment exists at all.

Nothing else in your education does this. You have written hundreds of pages of prose and not once has a machine come back to tell you that a particular sentence had two meanings, and shown you the proof. Human readers do not do that. Human readers patch the gap silently, out of politeness or out of momentum, and you never learn the gap was there.

So read the report as the rarest thing you will be handed all semester. Unambiguous feedback about ambiguity.
