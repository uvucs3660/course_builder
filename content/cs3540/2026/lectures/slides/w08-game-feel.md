---
track: game
week: 8
title: Feel
subtitle: Lies the Renderer Tells, the MDA Gap, and Watching People Play
runtime: 22
---

NOTES:
Week eight, game track, and this is the least technical lecture of the term. It is also, in my experience, the one that most changes people's games.

Everything up to now has been about correctness. Does the simulation agree with itself, does it agree with somebody else's build, does the hash match. This week is about whether hitting something feels like hitting something — which is not a correctness property, and which no conformance vector will ever check.

And because nothing checks it, most people never engineer it. They wait to feel inspired. This lecture is a list of things you can type instead.
---

# What you'll know after this

- Why feel lives in the **renderer** — and the one category that does not
- The juice list, ordered by **value per line of code**
- The **MDA asymmetry**: you edit mechanics, you care about aesthetics
- How to translate what a playtester says into something you can change

NOTES:
Four things, and the first is an architectural rule that stops this entire lecture from wrecking everything we built in weeks two and three. Feel is cheap. Feel in the wrong file is expensive.
---

# Where feel lives

![](game-feel-and-juice-where-it-lives.svg)

NOTES:
The rule first, because it is the one that costs you if you get it wrong.

Screen shake, hitstop, flash, squash, particles — every one of those is a lie. The world did not shake. The simulation subtracted ten from a health value, and everything else is presentation told on top.

So they live in the renderer. The moment you put one of them inside tick, you have broken replay, multiplayer, and every conformance vector at once. Hitstop inside the simulation means the simulation's notion of time now depends on whether something got hit, which means two builds with different collision resolution order end up with different tick counts. It is a catastrophe, and it is three lines of extremely tempting code.

Now the right-hand column, which is the exception and the interesting part. Coyote time — a few frames of grace after you walk off a ledge, where the jump still works. Input buffering — a jump pressed just before you land still fires when you land. Lenient hitboxes — the player's hurtbox drawn smaller than the player's sprite.

These are not lies. They change what the game does. Whether you can jump is a simulation question, not a presentation question. So they go in the simulation, which means they go in your specification and into the hash.

And the thing about all three is that nobody notices them when they are there. Nobody has ever walked out of a room praising the coyote time. Plenty of people have said a game felt unresponsive and been completely unable to tell you why.
---

# Juice, by value per line

- **Hitstop** — freeze 40–80ms on impact. **Three lines. The single biggest contributor.**
- **Screen shake** — decay by `t²`, not `t`. Linear decay feels mushy.
- Flash the sprite white for two frames
- Knockback, even a few pixels
- **Easing on every UI element** — nothing in a good game moves linearly
- Squash and stretch on jump and land
- Sound with **±10% pitch variance** — without it, repeats become a machine gun

NOTES:
This list is roughly ordered by how much feel you get per line of code, and the ordering is not the one people expect.

Hitstop is first and it is not close. Freeze everything for forty to eighty milliseconds on impact. Three lines. It is the whole difference between a hit registering as an event and a hit being a number that changed, and almost nobody implements it first.

Screen shake — decay by t squared, not by t. Linear decay reads as mushy, because the shake lingers past the point your brain expected it to stop. Quadratic snaps.

Easing on your interface elements is the one people skip, because it is not gameplay. Nothing in a well-made game moves linearly. A health bar that jumps straight to the new value is information. A health bar that eases into it is a game.

The pitch variance one is the cheapest thing on the list. The same sound sample fired twenty times in two seconds is a machine gun, and your ear catches the loop instantly. Randomise the pitch ten percent either way and it becomes twenty separate impacts again.

One footnote, and it is load-bearing. That randomisation lives in the renderer, so it uses ordinary Math dot random, not the simulation's seeded generator. Draw from the sim's generator for audio and the simulation now depends on how many sounds played. That is a real bug and it is on the audio cheat sheet.
---

# The MDA asymmetry

**Mechanics** — the rules you write. The only thing you can edit.

**Dynamics** — what emerges when people play them.

**Aesthetics** — what it feels like. The only thing you actually care about.

> You cannot implement "tense." You implement a mana curve and hope.

NOTES:
Here is the framework, and the reason it is worth knowing is entirely in the asymmetry.

You can only edit mechanics. Rules, numbers, systems, the contents of your source tree. That is the whole surface you have.

You only care about aesthetics. Nobody sets out to build a game with a well-tuned mana curve. They set out to build a game that feels tense, and the mana curve is a means to it.

And between the thing you can touch and the thing you want sits dynamics, which is what actually happens when real people meet your rules — and which you do not control. You influence it. That is all you get.

That gap is where every surprise in game design lives, and it runs in both directions. The mechanic you were proudest of produces nothing. The throwaway one produces the thing people talk about afterwards.

Read the quote and take it completely literally. There is no tension variable. You build a resource that runs out at an awkward moment, and then you find out.
---

# Translating feedback

![](mda-framework-translating-feedback.svg)

NOTES:
This is the most immediately useful thing in the lecture, so if you have been half-listening, come back now.

A playtester says the boss is unfair. That is an aesthetic report. It is accurate — they did feel that — and it is entirely unactionable, because there is no unfairness parameter for you to lower.

Your job is to walk it down the ladder. What actually happened? I die before I can react to the slam. Now it is a dynamic, and it is specific enough to be wrong about.

Then down to the mechanic. The slam's wind-up is eight frames. At sixty frames a second that is about a hundred and thirty milliseconds, which is roughly human reaction time — so the player is not reacting, they are guessing, and losing a guess feels like being cheated. Make the wind-up twenty frames and the fight is transformed without touching damage or health at all.

Now read the line underneath, because that is the part that saves you a week. The obvious fix, lowering the boss's health, would not have touched the problem. The player still dies to the slam. They just die to fewer of them.

Players report problems accurately and propose solutions badly. Take the report, discard the proposal. That is the correct division of labour, because they have information you cannot have and you have the systems view they cannot have.
---

# The playtest protocol

![](playtesting-protocol.svg)

NOTES:
And here is how to collect that information without contaminating it on the way in.

Say nothing. This is the hard one. You will want to explain the controls, and the instant you do, you have destroyed the only chance you will ever get to find out whether the controls are discoverable. Sit on your hands.

Write, do not talk. Timestamps and what they did, not how you felt about it.

Note every pause. Hesitation is the signal. It marks the exact spot where the game failed to teach something. A player who stops for four seconds in front of a door is telling you that your door does not look like a door.

Ask about intent afterwards, never during. What were you trying to do there. Not why didn't you jump.

And three outside testers is enough — the returns fall off fast. They must not be your teammates, because your teammates have already learned where the door is and cannot unlearn it. That is the single most common way to run a useless playtest and come away reassured by it.
---

# Before Thursday

- **Add hitstop.** Tonight. It is three lines and it will change how your game feels.
- Find one thing in your game a player would call "unfair" and walk it down to a **mechanic**
- Line up **three outside testers** — not teammates
- `cheatsheet-game-feel-and-juice`, `cheatsheet-mda-framework`, `cheatsheet-playtesting`

Thursday, AI: **Reading the Divergence Report.** The Act II response is due **Thu Oct 22.**

NOTES:
Do the hitstop tonight. Genuinely, tonight. It is three lines, and the ratio of effect to effort is the best in the course.

The second one is the discussion. Find the unfair thing in your own game and bring the walk-down: aesthetic, dynamic, mechanic. I want to see the middle step, because the middle step is the one everybody skips straight past on the way to a number they wanted to change anyway.

And start lining up testers now, because three outside testers is a scheduling problem rather than a technical one, and it is due in November. Scheduling problems do not compress.

Thursday is the divergence report, and your Act II response is due the following Thursday, the twenty-second.
