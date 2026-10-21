# CS 3540 lecture audio — voice AI in use, current state, and what's next

**Date:** 2026-10-08
**Scope:** the TTS narration for the 28 CS 3540 lecture decks served at https://cs3540-lectures.netlify.app
**Companion docs:** `docs/superpowers/specs/2026-08-27-cs3540-lecture-delivery-design.md` §16 (approved settings),
`docs/stage2-bulk-report.md` (the first seven decks), `docs/studio-report.md` (human recording studio).

---

## 1. The voice AI being used

| Item | Value |
|---|---|
| Provider | ElevenLabs, REST API `https://api.elevenlabs.io/v1` |
| Endpoint | `POST /v1/text-to-speech/{voice_id}` (plain, non-timestamped; raw mp3 bytes back) |
| Model | `eleven_v3` for the first 7 decks (5,000-char cap); **`eleven_v4`** from 2026-10-08 on (10,000-char cap) |
| Voice (decks 1–7) | **Michael** — an *instant voice clone* (IVC) of the instructor, `voice_id V7cskzCG9mkMsOof0Nmn`, created 2026-08-03 from one 22.4 s sample (sample id `PUMWt5K5q7wWZZ2YYFRK`). **Locked on the Free tier since the account lapsed.** |
| Voice (2026-10-08 on) | **Eric — "Smooth, Trustworthy"**, ElevenLabs premade library voice, `voice_id cjVigY5qzO86Huf0OWal`, middle-aged American male, conversational. Chosen 2026-10-08 as the standard voice because premade voices work on every tier. |
| Voice settings (clone, decks 1–7) | `stability 0.3 · similarity_boost 0.7 · style 0.5 · use_speaker_boost true` |
| Voice settings (Eric, v4) | `stability 0.5 · similarity_boost 0.75 · style 0 · use_speaker_boost true` — the library defaults; no bake-off budget existed on Free, and the result paces at the same 190–220 wpm as the clone decks |
| Speed | **Not** sent via the API (`speed` returned *slower* audio on v3). Pace comes from `ffmpeg -filter:a atempo=1.2` after synthesis: raw ≈122 wpm → stretched ≈146 wpm target, measured 195–213 wpm median on the shipped decks |
| Auth | `ELEVEN_LABS_KEY` env var (set in `~/.zshrc`); never logged, never committed |
| Code | `~/code/fivex/mod_node/modules/lecture/eleven-labs-client.ts` (client; model + settings per call) and `synthesize-lecture.ts` (per-slide synth → stretch → concat → patch JSON; `--model/--voice-id/--voice-name/--stability/--similarity/--style/--atempo/--gap-ms/--dry-run` flags, defaults = Eric on v4; refuses to start if the deck does not fit the remaining quota). Parser test: `__tests__/synthesize-options.test.ts` |
| Output | `content/cs3540/2026/lectures/slides/_lectures/<deck>.mp3` plus per-slide clips in `_lectures/audio/<deck>/{raw,stretched}/slide-NN.mp3`; `audio{}` and per-slide `audio_span` patched into `_lectures/<deck>.json` (gitignored) |
| Delivery | `lecture_site/scripts/build.ts` copies any `_lectures/<deck>.mp3` that exists into `dist/media/` and flags `has_audio`; `tool/publish_lectures.sh` deploys. Decks without an mp3 open in manual step-through mode |

### How a deck is synthesized

```bash
cd ~/code/fivex/mod_node
npx ts-node modules/lecture/synthesize-lecture.ts <deck-id> --dry-run   # cost + quota, sends nothing
npx ts-node modules/lecture/synthesize-lecture.ts <deck-id>             # Eric on eleven_v4, atempo 1.2
# reproduce the original clone run (needs a paid tier):
npx ts-node modules/lecture/synthesize-lecture.ts <deck-id> --model eleven_v3 \
  --voice-id V7cskzCG9mkMsOof0Nmn --voice-name michael --stability 0.3 --similarity 0.7 --style 0.5
```

Per narrated slide: POST the slide script → write the raw clip to disk immediately → `atempo=1.2` stretch →
`ffprobe` the **stretched** clip for its duration → concatenate all clips with a 350 ms silence gap →
compute `audio_span` from the stretched durations → patch the lecture JSON → validate with ajv.

The run is idempotent: a slide whose stretched clip already exists costs nothing; a slide with a raw
clip but no stretched clip is re-stretched without an API call. Each slide gets three attempts.

### Why it is shaped this way (from spec §16)

The first attempt sent a whole deck as one request, which forced `eleven_multilingual_v2` and
sounded "way too robotic." Per-slide requests fit under v3's 5,000-character cap (largest slide is
1,536 chars), hold their expressive shape, and let one slide be re-done without touching the deck.

---

## 2. Current state (measured 2026-10-08)

**8 of 28 decks have audio.** Seven (weeks 1–3 plus `w04-game-engine-seams`) are the Michael clone on
`eleven_v3`; `w04-ai-writing-a-spec` was synthesized 2026-10-08 with Eric on `eleven_v4` (9 slides,
8,176 chars, 7 m 07.7 s, median 203 wpm, spans contiguous, ajv PASSED — the whole Free-tier month in one deck).
**20 decks remain — 134,020 characters.**

| Deck | Slides | Chars | Audio |
|---|---|---|---|
| w01-ai-eleven-pillars | 7 | 6,084 | ✅ v3 |
| w01-game-first-contact | 9 | 8,776 | ✅ v3 |
| w02-ai-claude-md | 15 | 12,887 | ✅ v3 |
| w02-game-the-loop | 20 | 17,658 | ✅ v3 |
| w03-ai-skills | 10 | 8,648 | ✅ v3 |
| w03-game-determinism | 15 | 11,700 | ✅ v3 |
| w04-game-engine-seams | 10 | 8,891 | ✅ v3 |
| w04-ai-writing-a-spec | 9 | 8,176 | ✅ v4 Eric (2026-10-08) |
| w05-ai-subagents | 10 | 8,483 | — |
| w05-game-patterns-and-components | 12 | 10,488 | — |
| w06-ai-hooks | 7 | 6,093 | — |
| w06-game-pixels | 8 | 6,296 | — |
| w07-ai-mcp | 7 | 5,223 | — |
| w07-game-contact | 8 | 6,757 | — |
| w08-ai-divergence | 7 | 5,467 | — |
| w08-game-feel | 8 | 7,587 | — |
| w09-ai-model-selection | 6 | 4,782 | — |
| w09-game-space | 9 | 8,238 | — |
| w10-ai-spec-plan-execution | 7 | 5,798 | — |
| w10-game-minds | 8 | 7,487 | — |
| w11-ai-soul-prompts | 7 | 5,803 | — |
| w11-game-worlds | 9 | 7,910 | — |
| w12-ai-the-council | 6 | 4,988 | — |
| w12-game-story | 10 | 8,727 | — |
| w13-ai-plugins | 6 | 3,960 | — |
| w13-game-distance | 10 | 8,625 | — |
| w14-ai-guarded-agent | 6 | 5,510 | — |
| w14-game-shipping | 7 | 5,798 | — |
| **Done** | 95 | **82,820** | |
| **Remaining** | 158 | **134,020** | |
| **Whole corpus** | 253 | **216,840** | |

No human takes have been recorded in the studio yet (`recorded/` is empty for every deck).

### Account state — this is the blocker

`GET /v1/user/subscription` on 2026-10-08:

| Field | Value |
|---|---|
| tier / status | **free** |
| character_count / limit | 8,176 / **10,000** per month (after the 2026-10-08 deck; resets around the 1st) |
| can_use_instant_voice_cloning | **false** |

The bulk run on 2026-08-29 was on a paid tier with a 90,000 quota; the account has since
lapsed to Free. A live probe (one short v4 request with the Michael voice) returned:

```
HTTP 401  subscription_required / ivc_not_permitted
"Instantly cloned voices are not available on your current plan. Please upgrade your subscription."
```

So **nothing can be synthesized in the instructor's voice — on any model — until the plan is
upgraded.** The clone itself still exists (it is not deleted, only locked). **Decision 2026-10-08:
stay on Free, no Creator upgrade, switch to a standard (premade) voice.** Premade voices work on
every tier; the only constraint is the 10,000-character monthly quota.

---

## 3. Eleven v4 — what it is and what the offer actually is

Released 2026-09-28 (`eleven_v4` and `eleven_v4_turbo`). Confirmed on this account via
`GET /v1/models`: both are listed, `can_do_text_to_speech: true`, **10,000-character cap** per request
(double v3's 5,000; every slide in the corpus fits either way).

### What changes vs v3

- **New architecture, more stable speaker identity** across regenerations and long text; ElevenLabs
  ranks it #1 on the Artificial Analysis Speech Arena (their claim).
- **Natural-language direction inline** in square brackets: `[warm]`, `[long pause]`, `[whispers]`,
  `[said slowly, like explaining to a beginner]`. Tags are open-ended, not a fixed list. Place a tag
  right before the words it should affect. SSML `<break>` is **disabled** in v4 — use `[pause]`.
- **Instant voice clones are supported** and ElevenLabs says v4 IVCs beat Multilingual v2 PVCs.
  **But:** "clones created before v4 must be retrained on v4 to work well." The Michael clone
  (2026-08-03) predates v4. Its 22 s source sample is retrievable from the API, so a retrain does
  not need the original recording.
- `voice_settings` fields are the same shape (`stability`, `similarity_boost`, `style`,
  `use_speaker_boost`, `speed`). The docs do **not** say whether `speed` works on v4 — on v3 it
  made audio slower. Must be re-tested; keep the ffmpeg stretch until proven otherwise.
- `previous_text` / `next_text` (context stitching) is available and would help per-slide
  continuity; the current pipeline does not send it.
- Pricing per character is the same as other TTS models (1 credit = 1 character).

### The limited-time offer (ends **October 12, 2026** — four days from today)

What ElevenLabs is actually offering, from their own pages:

| Offer | Who gets it | Ends |
|---|---|---|
| **3× monthly credits** | Creator plan and above | Oct 12 |
| 2× monthly TTS credits on v4 | Creator+, **web/mobile app only** (ElevenCreative), not API | ~Oct 12 |
| Creator first month 50% off ($11 instead of $22) | new Creator subscribers | shown alongside |
| Starter first month $1 (83% off) | new Starter subscribers | Oct 18 |
| Discounted API rate ($22 / 1M chars v4, $11 v4 Turbo) | reported by press at launch, "two weeks" | ~Oct 12 |

**v4 is not "free" in the sense of zero cost.** It is free to *use* on every plan including Free
(10,000 chars/month), but the Michael clone is not usable on Free. The time-limited part that
matters here is the **3× credits on Creator+**.

### Plans that unlock the clone

| Plan | $/mo | Credits/mo | With 3× (until Oct 12) | IVC | Covers remaining 142,196? |
|---|---|---|---|---|---|
| Free (current) | 0 | 10,000 | — | ✗ | ✗ clone locked |
| Starter | 6 ($1 first month until Oct 18) | 30,000 | not eligible | ✓ | ✗ ~5 months |
| **Creator** | **22 ($11 first month)** | 121,000 | **363,000** | ✓ (+1 PVC) | **✓ all 21 decks, plus re-doing the 7 v3 decks on v4 (216,840 total) with ~146k spare** |
| Pro | 99 | 600,000 | 1,800,000 | ✓ | ✓ (overkill) |

Without the 3× promo, Creator's 121,000 is ~21,000 short of the remaining corpus, which would
mean a second month ($22 more) or Pro. **Upgrading to Creator before Oct 12 is the whole game.**

One thing to verify immediately after upgrading, before spending anything: that the 3× pool is
spendable through the API, not only in the web app. `GET /v1/user/subscription` →
`character_limit` should read ~363,000. If it reads 121,000 the promo is app-only and the plan
must be sized accordingly (two Creator months, or Pro for one month).

---

## 4. Decision (2026-10-08): Free tier, standard voice

The instructor chose not to buy Creator and to narrate the remaining decks in a standard voice.
Consequences, stated plainly:

- **Voice mix.** Weeks 1–3 (+ `w04-game-engine-seams`) are the instructor's cloned voice; everything
  from `w04-ai-writing-a-spec` on is Eric. Making the course uniform means re-synthesizing the 7
  clone decks in Eric (74,644 chars) — only realistic on a paid month.
- **Throughput.** Free is 10,000 characters a month. Every remaining deck except
  `w05-game-patterns-and-components` (10,488) fits in one month, and in week order no two consecutive
  decks fit together, so it is **one deck a month: about 19 months** (schedule below). The 10,488
  deck needs a Starter month or a studio recording.
- **Cheap accelerators that are not Creator:** Starter is $1 for the first month (offer shown until
  Oct 18) then $6/month for 30,000 credits — three decks a month, done in ~5 months for ~$25. It does
  not unlock the clone's full use for the corpus either, so the voice stays Eric.
- Free-tier output is licensed for non-commercial use with attribution; a university course page
  should carry an "AI narration by ElevenLabs" note somewhere visible (the player footer is the
  natural spot).

### Monthly schedule on Free (10,000 chars/period, week order, two decks when they fit)

| Period | Decks | Chars |
|---|---|---|
| Oct 2026 (done) | w04-ai-writing-a-spec | 8,176 |
| Nov 2026 | w05-ai-subagents | 8,483 |
| Dec 2026 | w06-ai-hooks | 6,093 |
| Jan 2027 | w06-game-pixels | 6,296 |
| Feb 2027 | w07-ai-mcp | 5,223 |
| Mar 2027 | w07-game-contact | 6,757 |
| Apr 2027 | w08-ai-divergence | 5,467 |
| May 2027 | w08-game-feel | 7,587 |
| Jun 2027 | w09-ai-model-selection | 4,782 |
| Jul 2027 | w09-game-space | 8,238 |
| Aug 2027 | w10-ai-spec-plan-execution | 5,798 |
| Sep 2027 | w10-game-minds | 7,487 |
| Oct 2027 | w11-ai-soul-prompts | 5,803 |
| Nov 2027 | w11-game-worlds | 7,910 |
| Dec 2027 | w12-ai-the-council | 4,988 |
| Jan 2028 | w12-game-story | 8,727 |
| Feb 2028 | w13-ai-plugins | 3,960 |
| Mar 2028 | w13-game-distance | 8,625 |
| Apr 2028 | w14-ai-guarded-agent | 5,510 |
| May 2028 | w14-game-shipping | 5,798 |

`w05-game-patterns-and-components` (10,488 chars) is **not** in the table: it exceeds a Free month
and the synthesizer refuses to start a run that does not fit, so it needs either one Starter month
(30,000 credits) or a studio recording. Everything else finishes in 19 further periods,
i.e. by May 2028. That is the honest Free-tier timeline; the
CS 3540 term the decks were written for ends long before it. The studio (human recording) remains
the way to narrate this semester's remaining decks without spending anything.

---

## 5. Next steps — ordered checklist

### Now

- [x] Pipeline takes model/voice/settings/atempo as flags; `--dry-run`; refuses to overspend
      (`~/code/fivex/mod_node/modules/lecture/`, tests pass: 11/11, `tsc` clean). **Not committed.**
- [x] `w04-ai-writing-a-spec` synthesized with Eric on `eleven_v4`; JSON patched; `lecture_site`
      local build reports 8/28 decks with audio.
- [ ] **Listen to `_lectures/w04-ai-writing-a-spec.mp3`** (or open the local player) and decide
      whether Eric at the library defaults is acceptable. If not, the next month's 10,000 is the only
      budget for trying another premade voice (Chris `iP95p4xoKVk53GoZ742B`, Brian `nPczCjzI2devNBz1zQrb`,
      Matilda `XrExE9yKIg1WjnnlVkGX`) — pick by their free preview mp3s first.
- [ ] **Deploy** when satisfied: `cd tools/course_builder/lecture_site && npm run deploy`. Do **not**
      use `tool/publish_lectures.sh` while audio lives in `_lectures/` — `bin/build_lectures.dart`
      refuses to clear a `_lectures/` that contains non-JSON files (mp3s, `audio/`, `slides/`), so
      the script fails before the site step. Fixing that generator guard (preserve `audio{}`/spans
      and non-JSON subtrees on regeneration) is its own task.
- [ ] Commit the mod_node lecture-module changes (and push the Aug 29 commit that was never pushed).
- [ ] Add an "AI narration by ElevenLabs" attribution line to the player (Free-tier licence term).

### Each month on the 1st (Free tier)

- [ ] `synthesize-lecture.ts <next deck> --dry-run` → confirm `fits: true`.
- [ ] Run it for real; check `ajv validation PASSED` and the span/duration diff (±2 ms).
- [ ] `npm run deploy` in `lecture_site`.
- [ ] Tick the deck off in the schedule above.

### If the timeline is unacceptable

- [ ] Starter at $1 first month (until Oct 18) → 30,000 credits → three decks immediately
      (`w05-ai-subagents`, `w05-game-patterns-and-components`, `w06-ai-hooks` = 25,064).
- [ ] Or one Creator month ($11 first month; 3× credits only until Oct 12) → entire remainder plus
      re-doing the 7 clone decks in Eric for a uniform voice.
- [ ] Or record in the studio (`npx ts-node modules/lecture/studio-server.ts`, Chrome,
      http://localhost:8532/studio) — zero credits, the player already prefers recorded takes.

---

## 6. Sources

- ElevenLabs — Eleven v4 product page: https://elevenlabs.io/v4
- ElevenLabs — Eleven v4 announcement: https://elevenlabs.io/blog/eleven-v4
- ElevenLabs — models overview (character limits): https://elevenlabs.io/docs/overview/models
- ElevenLabs — pricing (plans, credits, 3× until Oct 12): https://elevenlabs.io/pricing
- ElevenLabs — Create speech API reference (voice_settings, previous_text/next_text): https://elevenlabs.io/docs/api-reference/text-to-speech/convert
- ElevenLabs — audio tags list: https://elevenlabs.io/blog/elevenlabs-audio-tags-list
- Launch coverage with promo terms: https://www.testingcatalog.com/elevenlabs-launches-eleven-v4-and-v4-turbo-voice-models/ ,
  https://www.unite.ai/elevenlabs-launches-eleven-v4-with-low-latency-turbo-variant/
- Live API checks on 2026-10-08: `GET /v1/user/subscription`, `GET /v1/models`, `GET /v1/voices/{id}`,
  and one `POST /v1/text-to-speech/{id}` probe (401, quoted above).
