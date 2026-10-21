---
track: game
week: 9
title: S10 — The 3D Backend
subtitle: Specify the Submission, Not the Pixels
runtime: 15
---

NOTES:
This is the technique talk for section ten, the 3D backend. Fifteen minutes, same four parts you owe me for your own section: a mental model, a worked example, the failure mode, and the ambiguity.

The lecture already gave you the four matrices, the near plane, and the three rules of transparency. I am not going to repeat any of that. This talk is about the thing the section is actually *about*, which is a line. On one side of the line a renderer can be specified and two implementations can be held to the same answer. On the other side it cannot, and no amount of careful prose will change that. Section ten is an exercise in finding that line and writing the spec right up against it.

The second half is about the section as a prompt. Every one of your sections is read by an agent that builds an engine from it, and the choices you make in the prose are prompt design whether you think of them that way or not. I will show you what mine did when the builds read it, and where it went wrong.

---

# The mental model

```
simulation  →  render view  →  ordered draw list  →  GPU  →  pixels
   hashed        extracted        SPECIFIED         not specifiable
```

- **The simulation must not know rendering exists** — S00's guiding rule
- Pixels are **not** specifiable: GLSL ES values are IEEE 754 *laid out*, but operations need not *be* IEEE 754, and optimizers reorder
- So S10 specifies **what is submitted, in what order**, and says nothing about the framebuffer

NOTES:
Here is the whole section in one line.

The simulation produces state, and that state is hashed. That is the part of the engine the conformance runner can see. From that state the backend extracts a render view, a snapshot, never a reference to the world. From the view it builds a draw list, and then it hands the draw list to the GPU, which turns it into pixels.

The question is where the specification stops. The instinct is to say that it stops at the pixels: two correct renderers should produce the same image. They do not. The GLSL ES spec is explicit that values are laid out as IEEE 754 but operations are not required to follow IEEE 754, and the optimizer may reorder your arithmetic. That is precisely why the `invariant` qualifier exists in the language. Two conforming drivers can shade the same triangle to different bits, and both are correct.

So the line is one step earlier. What gets submitted, and in what order, is fully specifiable, because it is integer sorting on data the simulation already made deterministic. The framebuffer is not. Section ten draws the line there and says so in its first paragraph, because an implementer who does not know where the line is will spend the term trying to make screenshots match.

---

# The render view

Rebuilt every frame from simulation state. **Never a reference to the world.**

| Field | Meaning |
|---|---|
| `entityId` | S05 id — and the final tie-break |
| `layer` | world 0 · overlay 1 · UI 2 |
| `translucency` | opaque 0 · alpha-tested 1 · translucent 2 |
| `materialId`, `meshId` | batching keys |
| `x`, `y`, `z` | interpolated world position |

Plus `alpha` in `[0, 1)` — `accumulator / 50` from S01.

NOTES:
The backend is handed a view and an alpha. Nothing else.

The view is rebuilt each frame rather than mutated, and that is a deliberate choice, not a performance one. An extracted snapshot cannot accidentally carry a pointer back into the simulation. A long-lived view object that gets patched each frame eventually does, because somebody needs one more field and reaches through to the entity. Rebuilding is how you make invariant one physically hard to break rather than just forbidden.

Every field in this table is an integer except the three positions. Integers because they are sort keys, and a sort key that is a float invites the question of what equal means. The one float that matters for ordering, depth, gets quantized before it is compared. We will get to that.

Alpha comes from the loop in section one. It is the fraction of a tick that has elapsed since the last one, and a frame may legitimately arrive with alpha exactly zero, right on the tick boundary. The spec says that outright because an implementer who assumes alpha is strictly positive will divide by it somewhere.

---

# Interpolation lives in the backend

- The backend blends **prior pose → current pose** by `alpha`
- **Interpolation, never extrapolation** — one tick of latency, always correct
- Rotations: **shortest-arc slerp**, not componentwise lerp
- Previous-tick transforms are a **shadow copy in the backend**, keyed by entity id

> A `previousX` field on the entity would enter the S02 hash fold — and two peers at different frame rates would desync.

NOTES:
Here is the design decision in this section that I think is the least obvious and the most important.

Rendering at a rate different from the simulation means the backend has to draw entities *between* ticks. That needs the pose from the last tick and the pose from this one, blended by alpha. The obvious place to keep the previous pose is on the entity. Add a `previousX`, `previousY`, done.

That is a determinism bug, and a subtle one. The entity's fields are what section two folds into the state hash. Put a previous-tick position there and it is now part of the simulation's state. Two peers in a multiplayer game run the same simulation, but one renders at sixty frames a second and the other at thirty. Their hashes now differ on a value that has nothing to do with the game. The replay still works. The desync detector fires anyway.

So the spec is categorical: previous-tick transforms live in the backend, as a shadow copy keyed by entity id, and never in the simulation. This is what Godot does, keeping the prior transform inside the rendering server rather than on the scene node, and for the same reason.

Two more things. Interpolate, do not extrapolate. Extrapolation predicts from velocity and lets geometry pass through a collider for one frame and then snap back. Interpolation costs exactly one tick of latency and is never wrong. And rotations blend by shortest-arc slerp, because a componentwise lerp of a quaternion is not a rotation, and without the sign fix two implementations can spin the long way round on the same pair of orientations while both believe they are correct.

---

# Draw order — the six-field key

| Rank | Field | Direction |
|---|---|---|
| 1 | `layer` | ascending |
| 2 | `translucency` | ascending |
| 3 | `depthKey` | **front-to-back** for opaque, **back-to-front** for translucent |
| 4 | `materialId` | ascending |
| 5 | `meshId` | ascending |
| 6 | `entityId` | ascending — **the tie-break that makes it total** |

`depthKey = quantize(z, 1000)` per S00, **saturated** to `0 .. 65535`.

NOTES:
This is the part of the section that two implementations can actually disagree about, so it is the part the section makes executable. There is a reference implementation in the repo and a test file that asserts every acceptance criterion against it.

Six fields, most significant first. Layer first, because the UI must never be occluded by the world, whatever the depth values say. Translucency second, so all opaque work precedes all blended work. Then depth, and the direction flips per class: opaque sorts front to back so the depth test rejects hidden fragments early and the GPU does less work, translucent sorts back to front because blending is not commutative. Then material and mesh, which exist for batching, so consecutive draws share state.

And then entity id. This is the field that matters most and that nobody puts in. Depth is quantized to a thousandth of a unit and then clamped, so exact ties are not rare, they are constant. Anything resting on a shared ground plane has the same depth key. Two implementations that follow every one of the first five rules perfectly will still order those draws differently, because the comparator returned zero and handed the decision to whatever sort algorithm the language ships. Entity id is the only field guaranteed unique, so it goes last, and now the comparator can never return zero for two distinct renderables. The order is total. Stability of the sort stops mattering, which is the point.

Saturation on the depth key, not wrapping. A wrap makes the most distant geometry sort as the nearest. That is the kind of bug that only shows up on a large map, which is to say, in December.

---

# Worked example

Six renderables, all in `layer 0` unless noted. Sort them.

| id | translucency | z | mat | mesh |
|---|---|---|---|---|
| 7 | opaque | 2.0 | 1 | 1 |
| 3 | opaque | 2.0 | 1 | 1 |
| 12 | opaque | 0.5 | 4 | 2 |
| 9 | translucent | 1.0 | 2 | 1 |
| 2 | translucent | 3.0 | 2 | 1 |
| 5 | opaque, **UI layer** | 0.0 | 0 | 0 |

**Submission order: 12, 3, 7, 2, 9, 5**

NOTES:
Try this one yourself before I walk it.

Layer first. Entity five is on the UI layer, so it goes last no matter what else is true about it. Everything else is world layer.

Translucency next. Seven, three, and twelve are opaque, so they all precede nine and two.

Now depth among the opaque three. Twelve is at half a unit, depth key five hundred. Seven and three are both at two units, depth key two thousand. Front to back, so twelve first. Seven and three tie on depth, tie on material, tie on mesh, and the tie-break is entity id ascending. Three, then seven. If your comparator stopped at mesh, this is the pair it would have handed to the sort implementation.

The translucent pair sorts back to front. Two is at three units, nine is at one. So two before nine.

Twelve, three, seven, two, nine, five. And the test I would point you at is the one that shuffles a thousand renderables two different ways and asserts the sorted output is identical both times. That is the property the whole section rests on, and it is one line to check.

---

# The failure modes

- **A comparator that can return 0.** Order of tied draws depends on the sort algorithm. *Looks like: flicker between two objects on the same floor, only on some machines.*
- **Previous pose stored on the entity.** Enters the hash; peers at different frame rates desync. *Looks like: a replay that passes and a multiplayer game that does not.*
- **Depth key wraps instead of saturating.** The horizon draws in front of the player. *Looks like: fine on the test map.*
- **Context lost without `preventDefault()`.** The context is **never restored.** *Looks like: a black canvas after a tab switch, forever.*

NOTES:
Four ways this goes wrong, and each one is a single omission.

The comparator that can return zero. You followed five rules and skipped the sixth because entity id did not seem like it had anything to do with rendering. Now two crates on the same floor flicker against each other, but only on the machine whose sort is not stable, which is not yours.

The previous pose on the entity. Your replay passes every vector, because a replay runs at one frame rate. The multiplayer game desyncs the moment one player's laptop throttles. You will look for the bug in the netcode for a day. It is in the renderer.

The wrap. Your depth key is a sixteen-bit integer and you let it overflow. Everything is fine on the test map, which is small. The first large level puts the far hills in front of the player.

And context loss. The GPU can take the context away at any moment, a driver update, a background tab, VRAM pressure. Every buffer, texture, program, and vertex array object becomes a corpse at once. The browser fires an event, and if your handler does not call `preventDefault`, the context is never restored. Not restored late. Never. The spec says this in bold because it is the single most common way the whole thing is got wrong, and the symptom is a black canvas with no error.

Context loss is also why asset handles are opaque ids owned by the backend. Nothing outside the backend may hold a raw texture object, because after a loss that object is dead while the handle is still valid and can be rebuilt from section eight's assets.

---

# Part two: the section is a prompt

The generator reads, in this order:

1. `spec/S00-overview.md` — the contract and the rounding rule
2. `spec/OWNERS.md` — status, which decides what is built
3. Every section marked `stable` or `draft`

Then it must **implement what the prose says, not what it would have designed**, and mark every gap:

```ts
// AMBIGUITY: S10 does not say … Using the simplest reading.
```

Every `AMBIGUITY:` is a finding against the section's owner.

NOTES:
Now the other half. The spec is a prompt. It is read by an agent that has never seen it before, that will be built against by another agent that has also never seen it, and the two of them will be compared.

This is how it is read. The overview first, because that is where the hash and the rounding rule live. Then the owners table, because status decides whether a section is built at all, whether its vectors are replayed, and whether a disagreement blocks promotion. Then every section that is stable or draft.

The rule the agent is given is the one that makes the experiment work: implement what the prose actually says, not what you would have designed. Where the prose is silent, take the simplest reading and mark it with an ambiguity comment. Those comments are collected and they are the report. If the agent silently picks the sensible answer, the builds agree, the report says the section is clean, and the ambiguity survives to bite someone in the capstone.

And there is a hook that makes `spec/` and `conformance/` unwritable. The prompt asks the agent not to edit the vectors it is measured against. The hook makes it impossible. An agent asked to make tests pass, and able to edit the tests, will edit the tests. Declined versus blocked.

---

# Prompt design choices in S10

- **The claim names what is *not* claimed.** "No claim whatsoever that the pixels are identical" — stops the agent chasing screenshots
- **Every tie-break, direction, unit, and clamp is named** — rule 4 of S00
- **The contentious part is executable.** `src/draw_order.ts` + `test/draw_order.test.ts` are prose the agent can run
- **`Conformance state: nothing` is declared outright** — rule 5: an undeclared contribution is unverifiable by construction
- **An honest list of what is asserted by review, not by test** — reset marking, no sim mutation, no-op while lost

NOTES:
Five choices in the prose that are there for the reader that is an agent.

First, the claim says what it does not claim. Two sentences in, the section says it makes no claim whatsoever that pixels are identical. Without that, an agent will reasonably try to specify the framebuffer, and it will produce a confident and wrong section of work. Telling a model what not to attempt is as much of the prompt as telling it what to do.

Second, every place an implementer could choose differently has the choice made. Ascending or descending, per class. Quantized at a thousand, half away from zero, saturated at sixty-five thousand. The final tie-break. Rule four of the overview says name the rounding, the units, and the tie-break, and this section is mostly an exercise in doing that.

Third, the part two builds could disagree about is not prose at all. It is a reference implementation with a test file, and the acceptance criteria in the section are the test names. An agent can run it. A second-language implementer can read it. That is the strongest form a specification can take, and you should use it for the one part of your section where you most expect disagreement.

Fourth, the conformance state paragraph says, in one sentence, that this section contributes nothing to the hash. Rule five requires that sentence. A section that is silent about its contribution is unverifiable, because a vector that expects state the section never declared cannot be checked against anything.

And fifth, the section ends with a short list of things that are asserted by review rather than by a test, because no reference implementation covers them. That is the honest part. It tells the reader exactly where the measurement stops.

---

# What the builds did with it

Divergence runs 7 through 12, every build, the same finding:

> **S10 — presentation. Contributes nothing to the hash, defines no command kinds. Nothing to build.**

- Zero ambiguity comments
- Zero disagreements
- Zero vectors replayed

**That is not a clean section. That is an invisible one.**

NOTES:
So I ran it. Six builds across three nights read section ten, and every one of them wrote the same sentence in its notes: presentation, nothing in the hash, no command kinds, nothing to build.

No ambiguity markers. No disagreements. Perfect agreement.

And I want you to notice what that means, because it is the thing I most want you to take from this talk. Perfect agreement across independent builds is the signal this course uses to say a section is unambiguous. Here it is saying nothing of the kind. The builds agreed because the conformance surface, the state hash, cannot see anything the section specifies. The draw order, the interpolation, the context loss handling, none of it is inside the hash, so none of it was built, so none of it could disagree.

A section the generator cannot observe is not a stable section. It is an unmeasured one. And when the report for your own section comes back clean, the first question to ask is not "did I write it well" but "could it have failed."

---

# My ambiguity

The section is caught between two rules it cannot both satisfy:

- **S00:** *the simulation must not know that rendering exists* — so nothing in S10 may touch the hash
- **S00 rule 3:** *a claim with no vector is an opinion* — so S10 ships no vectors and is an opinion

The draft records this as an exception. **An exception is not a resolution.**

The fix under review: a **second conformance surface** for presentation — a vector that supplies a render view and an alpha and expects a **digest of the ordered draw list** alongside `stateHash`. Then two builds that order draws differently *can* disagree, and the runner can see it.

NOTES:
Here is the ambiguity, and it is not a sentence I phrased badly. It is a contradiction between two rules in the overview, and the section I wrote tried to resolve it by declaring an exception.

The guiding rule says the simulation must not know rendering exists. So nothing in section ten may enter the state hash. Correct, and I stand by it. Rule three says a claim with no vector is an opinion. Section ten has no vectors, so by the course's own standard it is an opinion. Also correct. The draft says, in effect, these two rules collide here and we will record the collision and move on. That is a note, not a fix.

The fix I am now proposing is a second surface. The state hash stays exactly what it is, and the simulation stays blind. But a vector may also supply a render view and an alpha, and expect a digest of the ordered draw list, the entity ids in submission order, folded with the same hash the rest of the engine uses. That costs a field in the vector format and a method on the engine interface, both of which are in the core I own. In return, the generator has to actually build the backend's ordering, and two builds that sort the ties differently produce two different digests, and the report finally has something to say about section ten.

That change has not shipped. The section stays draft until it does and until independent builds agree on it, which is the same bar you are held to. I am telling you this because point four of your own talk is the same point. The divergence report is honest with you or it is nothing, and "my section had no findings" is only good news if your section could have had some.

---

# Try this

- **Find the line in your own section.** What can two builds be held to the same answer on, and where does that stop? Write the spec up to the line and say so.
- **Make the contentious part executable.** One reference function, one test file. The acceptance criteria are the test names.
- **Ask whether your clean report could have failed.** If the runner cannot see your section, zero findings means zero measurement.
- `cheatsheet-3d-rendering-webgl` · `cheatsheet-shaders-and-materials` · `spec/S10-3d-backend-webgl2.md` · `src/draw_order.ts`

NOTES:
Three things to take back to your own section.

Find the line. Every section has a part that is deterministic and specifiable, and a part that is not, or that belongs to someone else. Write the spec right up to that line and say where it is. The sentence "this section makes no claim about X" is one of the most useful sentences you can write, for a human reader and for an agent.

Make the part you expect disagreement on executable. A reference implementation and a test file are a specification an agent can run, and they are the one form of prose that cannot be read two ways.

And interrogate your clean report. If the generator cannot observe your section, it will never disagree about it, and the absence of findings is the absence of measurement. The question to ask is not whether you had divergence but whether you could have.

The section, the reference implementation, and the two cheat sheets are in the usual places. Questions.
