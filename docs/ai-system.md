# The AI system

What sits between a language model and a user who will act on what it says.

---

## The position

LiftQuest gives training advice. Wrong advice has a cost — a bad load prescription, a claim that you have plateaued when you have not, a confident number that never existed. So the system is built on one rule:

> **The engine owns the diagnosis. The model owns the phrasing.**

Every number, verdict, and prescription is computed deterministically from logged training data. The model interprets what was asked and says the engine's answer like a coach would. It is a phrasing layer with bounded authority, wrapped in validation that can reject.

This is not a hedge against bad models. It is a structural claim: for a product where the output is *acted on*, correctness has to be a property of the system, not a property of the prompt.

---

## Why prompt discipline is not enough

The system prompt does tell the model to ground every number in the evidence. That instruction is necessary and it is not sufficient, for three reasons:

1. **Compliance is probabilistic.** An instruction that holds 99% of the time fails once per hundred coaching interactions, and each failure is a user acting on a fabricated number.
2. **Prompt behaviour is not testable.** You cannot write a regression test for "the model kept its promise." You can write one for "the validator rejected this reply."
3. **Errors persist.** Two coaching surfaces write their output to memory, where a later call reads it back as history. An invented figure does not evaporate — it becomes a durable fact the system reasons from.

Prevention lives in the prompt. **Enforcement lives in code**, and the code is what the tests exercise.

---

## The context pack

Before any prose call, the engine assembles one grounding block from data it has already computed:

```
TODAY: <date>

LAST SESSION:
  <session summary with real figures>

RECENT TRAINING (loads in kg; per lift: newest session first):
  <per-lift session history, e1RM, session-over-session deltas>

MEASURES (body, logged by the user):
  <body measurements with dates and deltas>

CLAIM PERMISSIONS:
  <what the engine's ledger permits the model to conclude>

PLATEAUS / PLATEAU CAUSE / TRAINING PATTERN / WATCH:
  <engine-computed signals>

ATHLETE NOTES (what they told you, newest first):
  <the athlete's own words>

MEMORY:
  <stored injuries, preferences, notes>

DETERMINISTIC READ:
  <the engine's own answer, already written out>
```

Two properties of this construction matter.

**It is pure.** No queries, no clock reads — today's date is a parameter. The same input always builds the same pack, so the gate can be tested against exactly the text the model receives rather than a convenient fiction.

**The headings are a trust boundary.** `ATHLETE NOTES` and `MEMORY` are in the pack because the coach must be able to *use* them — the model needs to know about the shoulder injury and the dislike of barbell squats. They are absent from the evidence list because the athlete's words are the question, never the proof.

`MEASURES` sits in a third category: engine-authored, but body facts rather than training loads. It is tracked separately so that *"you weigh 82 kg"* stays sayable while *"load the bar to 82 kg"* does not inherit that grounding.

---

## Gate 1 — Interpretation

**Problem.** Understanding what the athlete asked was a hand-maintained keyword table. It failed the way keyword tables fail: *"give a bicep workout"* matched no phrase, fell back to the athlete's registered goal lift, and returned a **squat session** explained in confident detail. Every number in that reply was correct. It answered a different question.

Four muscle keywords were patched after that bug. A hundred other sentences would have missed identically. Classification and extraction are things language models are genuinely reliable at — so that job moved to the model, under a contract.

**Contract.** The interpret pass returns strict JSON:

```
intent       build | edit | start | question | report | smalltalk | unclear
topic        progress | plateau | plan | recovery | nutrition | review |
             injury | injury_cleared | preference | gratitude | greeting | other
constraints  focus, minutes, equipment, exclude[]
references   theWorkoutOnScreen, exercise
confidence   the model's own read of how sure it is
```

Three rules make this safe:

1. **Every field is narrowed** against a closed vocabulary before anything acts on it. The output is untrusted structure, not trusted content.
2. **A malformed shape rejects the entire interpretation.** The caller degrades to the keyword fallback rather than acting on half a reading. Half-parsed intent is worse than none.
3. **Unsure routes to a question.** This is the load-bearing rule. A reading that does not know what was meant produces a clarifying question, not an answer.

That third rule is the actual fix for the squat bug. The failure was never "the keyword table lacked *biceps*" — it was that a sentence nobody understood got answered anyway. **A coach that guesses in silence is worse than one that asks.**

Note also what interpretation does *not* receive: no context pack, no evidence, no honesty rules. It cannot cite a number because it is never shown one. Separating "what was asked" from "what is true" means a misread produces a wrong topic, not a wrong fact.

---

## Gate 2 — Number grounding

Every quantity in a model reply must appear in the engine-authored sections of the context pack **and match its domain**.

### Domains, not digits

A naive check — "does this digit appear anywhere in the evidence?" — is close to useless, because small integers are everywhere. A rep count of 3 would silently ground *"you only slept 3 hours."*

So numbers are classified into domains and grounded only within their own:

| Domain | Grounds |
|---|---|
| `weight` | loads on the bar |
| `reps` | repetitions |
| `rpe` | rated effort |
| `e1rm` | estimated one-rep maxes |
| `percent` | trends and changes |
| `body` | bodyweight, measurements |
| `duration` | time spans |
| `count` | sets, sessions, days |
| `volume` | session tonnage |

An RPE of 8 does not license *"up 8%"*. An e1RM does not license a working load. A bodyweight does not license a weight on the bar.

### Fail closed

Only blocks with a recognised engine-authored heading are evidence. Everything else — including a note that contains a blank line and therefore splits into its own block — is untrusted. When the classifier cannot tell what it is looking at, it refuses.

### Dates are not quantities

Date digits are stripped before extraction. Without this, *"Jul 10"* grounds *"10 sets"*.

### Rejection, not repair

A reply that fails is **dropped**. It is never edited, and no second model call is asked to fix it. The caller falls back to the deterministic engine reply for chat, or to silence for the proactive coach card and the weekly review. Silence is the cheaper error — the engine's own receipts still render either way.

### Stated limit

This gate answers *"is this number real"*, not *"is it current."* Freshness of a cited body fact is the belief layer's job. Knowing which question a gate does not answer is part of the design.

---

## Gate 3 — Claim grounding

The harder half. These carry no digits and pass any number check:

> *"You've plateaued."* · *"Fatigue is accumulating."* · *"Recovery looks good."* · *"That adjustment worked."*

A `ClaimsLedger` is built deterministically from what the engine actually computed — plateau flags, per-lift e1RM trends, the deload verdict, the recovery signal **and whether it is measured or merely absent**, graded prescription outcomes. Each of nine claim topics is either permitted or not, scoped per exercise.

One rule eliminates most false rejections: **any claim the engine's own copy already states is automatically permitted.** Receipts, deterministic read, digests. The model may echo the engine; it may never outrun it.

The ledger also rides *into* the evidence as a `CLAIM PERMISSIONS` block, so prevention and enforcement use the same source of truth. The model is told what it may conclude, and separately checked on it.

### The detector is a narrow net, and says so

Enforcement is a scoped detector for high-risk claim categories, with negation, question, and conditional guards so honest denials and hypotheticals pass — *"no plateau signal is active"* is not a plateau claim.

It will not catch every possible phrasing, and the module's own documentation states this rather than implying completeness. What it does is make unsupported claims **structurally difficult, detectable, and testable**. Prevention lives in the prompt; this is the backstop.

Deterministic and synchronous. No judge LLM, no second call, no added latency.

---

## Gate 4 — Bounded authority over plans

The plan path is where a model could do real damage, so its authority is enumerated rather than restricted.

The deterministic engine drafts the session — exercises, loads, sets, reps, alternatives — and folds the draft plus the athlete's history, calibration, and patterns into an evidence JSON. The model reviews it and may do exactly three things:

1. **Choose among alternatives the engine already listed.** Swap targets are validated against that exercise's own alternatives; the first exercise can never be swapped; at most two swaps, and the documented expectation is that the right answer is usually none.
2. **Write one per-exercise note** — the single highest-value user-specific point, only where it adds something the draft's own receipt does not already say.
3. **Headline the session**, or return null when the draft's own explanation already covers it.

It has no authority over loads, sets, or reps. Every number in a note is checked against the evidence, every claim against the ledger. Anything failing validation degrades to the engine's own receipt — which was always going to render anyway.

---

## Gate 5 — The constraint contract

Available equipment and physical limitations resolve to a small closed vocabulary — five limitation tags, a coarse equipment tier — with one normalisation boundary mapping every input alias to a canonical tag. Unknown or malformed input degrades to null; it never throws, and it never silently disables the safety check.

The exclusion list is the same hand-curated body-part mapping the injury system uses, so there is one source of truth rather than two that drift.

The contract is enforced **at the engine's output and again at the AI boundary**. An exercise the athlete cannot perform cannot survive into an executable plan regardless of which layer proposed it.

The module states its own scope explicitly: *this is not a medical model, it is conservative exercise exclusion, never a diagnosis or a rehab prescription.* A fitness app that implies clinical authority is making a claim it cannot support.

---

## The belief layer

Grounding answers *"is this real."* Provenance answers *"how do we know, and is it still true."*

**The failure it prevents:** onboarding once wrote a "sane default" bodyweight of 78 kg. From that moment the column was indistinguishable from a number the athlete had stated — and it graded bodyweight goals and bodyweight-relative strength for someone who had never told the app anything.

Every meaningful belief now carries a source, a confidence, and a last-true timestamp. Sources are ranked by authority:

```
user_stated → user_corrected → imported → observed → derived → llm_inferred
```

Three rules follow:

1. **Unknown stays unknown.** There is no default that stands in for a fact.
2. **What the user stated outranks what was inferred** — and a later inference never silently overwrites an explicit correction.
3. **Beliefs go stale.** A bodyweight from eight months ago is not current truth.

`llm_inferred` sits at the bottom and is never authoritative. A model may propose a belief; it cannot establish one.

### Pain has a lifecycle

The same reasoning applied to injuries. *"My shoulder hurts today"* is not a permanent property of an athlete, but stored as a note it behaves like one — quietly excluding overhead work for the rest of the account's life.

Injury notes now expire by category, and one shared filter is the single path every coach surface uses to read them. A lapsed note stops constraining the plan and stops being quotable as current fact.

---

## Failure modes

| Failure | Behaviour |
|---|---|
| No provider key configured | 503 → deterministic reply |
| Not entitled | 403 → deterministic reply (the entitlement mirror fails **open**) |
| Rate limited (40 chat calls/day/user) | 429 → deterministic reply |
| Upstream error or timeout | 502 → deterministic reply |
| Malformed interpretation JSON | Whole interpretation rejected → keyword fallback |
| Low-confidence interpretation | Clarifying question, no answer |
| Ungrounded number in prose | Reply dropped → engine wording, or silence |
| Unsupported claim | Reply dropped → engine wording, or silence |
| Invalid plan note or swap | That note/swap discarded → engine's own receipt |
| Exercise violates constraints | Removed and substituted with a same-role permitted exercise |

Every path degrades to the deterministic coach. **The worst realistic outcome is that the coach sounds blunter than usual.**

That is the property the whole design is for: the LLM layer is an enhancement that can fail without taking correctness with it.

---

## Cost and latency

- Greetings and thanks cost **zero model calls** — short-circuited client-side before interpretation.
- A real chat turn costs **two** — interpret, then speak.
- `interpret` is blocked by the daily cap but does not consume it, so interpretation can never starve the answer.
- All gates are synchronous pure functions. Validation adds no network round trip.
- Every input is bounded: question length, turn count, per-turn length, context size, upstream timeout.

Provider precedence is Anthropic → OpenAI → Gemini, resolved by which secret is set. Changing model or provider is an operational change, not a code change — the contract each purpose must satisfy lives in the client's validators, not in the choice of vendor.

---

## What this system does not have yet

The gates are thoroughly tested. **The model's behaviour is not systematically evaluated**, and that is the most significant gap in the current design.

What exists is adversarial testing of the *validators* — given a reply containing an ungrounded number, does the gate reject it. What does not exist is an evaluation harness over real model output: a fixed set of athlete states and questions, run against each provider, scored for how often replies are rejected, how often a rejection was correct, and how often a wrong claim slipped past the detector's narrow net.

Three consequences follow, and they are worth naming rather than leaving for someone to find:

1. **The false-rejection rate is unmeasured.** Every rejection costs the user the model's better wording. If the gate is rejecting 30% of otherwise-fine replies, the LLM layer is paying for itself far less than it appears to.
2. **The claim detector's coverage is unknown.** It is documented as a narrow net for high-risk categories, not a complete one. How narrow is an empirical question that has not been asked.
3. **Provider swaps are untested against the contract.** Precedence makes switching trivial operationally; nothing currently verifies that a different model satisfies the JSON contracts and grounding discipline at a comparable rate.

The intended fix is a golden-set harness — recorded athlete states, a fixed question battery, per-provider scoring on rejection rate and rejection correctness, run as a periodic check rather than in CI, since it costs real tokens and is inherently noisy.

The reason it does not exist is sequencing rather than oversight: the gates had to be correct before measuring how often they fire meant anything. But a system whose whole argument is *"do not trust unverified output"* should hold its own probabilistic component to that standard, and right now it does not.
