# Product

The problem, the workflows, and the principles the code is held to.

---

## The problem

Strength training software splits into two categories, both of which fail.

**Logging apps** record what you did with high fidelity and tell you nothing about what to do next. The athlete accumulates a detailed history and still makes the decisions unaided — which is fine if they already know how to programme, and most people do not.

**AI fitness apps** generate a plausible-sounding programme from a chat prompt. The output reads well. It has no memory of what you actually lifted, no model of how you respond to load, and — critically — no way to distinguish an observation from an invention.

The second failure is the one worth solving. Asked *"how is my bench going?"*, a language model will answer *"your bench e1RM is 102.5 kg, up 8% since May"* with total confidence whether or not any of those figures exist. In a coaching product that is not a cosmetic error. It gets written to memory, read back as history, and used to prescribe load.

LiftQuest's position: **a language model should never be the source of a fact about your training.** It can interpret what you asked and it can say the answer well. The answer itself comes from your logged data through deterministic code.

---

## Core workflows

### Log a session

Start a workout, log sets as you do them, finish. Per-set rest timing owned by the set that started it, RPE capture, plate maths, warm-up handling, custom exercises.

Two durability properties the user never sees unless something goes wrong: a session in progress survives a crash, a backgrounding, or an OS kill; and a finish that cannot reach the network is queued on the device and replayed later, so a workout logged in a gym basement is never lost.

### Get today's session

The coach drafts a session from movement-pattern slot templates, per-lift progression state, RPE calibration, accumulated workload, plateau signals, available equipment, and active limitations. Every prescription carries a receipt — the numbers the decision was made from — so the athlete can see *why* the bar is at that weight rather than being told to trust it.

### Ask the coach

Free-text chat grounded in logged training. The athlete can ask about progress, request a session, modify the one on screen, report an injury, or state a preference. Answers come from the engine; the model interprets the question and phrases the reply. A question the system does not understand produces a clarifying question rather than a confident wrong answer.

### See what the coach sees

A dedicated screen showing every value the coach reads, the rule it feeds, and the verdict it produces. Not a debug view — a product surface.

An athlete asked to accept a load prescription from an opaque system has no basis for judging it. Showing the inputs converts *"trust the AI"* into *"here is the reasoning, and here is where you disagree."* It also makes the system falsifiable by its own user, which is the strongest correctness pressure available.

### Track progress

Per-lift estimated 1RM trends, plateau detection with cause attribution, weekly volume, body measurements, recovery check-ins, and a weekly review written from the same gated evidence as everything else.

### Social

Feed, friends, reactions, comments, direct messages, leaderboards. Blocking is enforced in the database, so content from a blocked user never reaches the device.

---

## Design principles

### 1. When you cannot verify, do not assert

The rule the whole product is organised around, and it applies well outside the AI layer.

The coach does not state a number it cannot trace. The offline queue does not precompute a recap it cannot check against history, because that would fabricate "first workout!" and invented personal records. The belief layer has no default that stands in for a fact — unknown stays unknown. A month of missing rows is a question about logging, not a verdict about detraining.

### 2. Silence beats a manufactured insight

The proactive coach card is explicitly permitted to return nothing. If the evidence shows nothing that genuinely needs attention, the card does not render.

An app that produces an insight every day teaches the athlete that its insights are decoration. Withholding is what makes the ones that do appear worth reading.

### 3. Never pay for behaviour that makes training worse

Every reward surface is audited against one question:

> **What behaviour does this pay for, and would a lifter who maximised it train better or worse?**

A reward that can be maximised through clearly worse training is a defect regardless of how good it looks.

This is not abstract. Consecutive-day streaks pay for training when you should rest — so consistency is measured in **weeks**, never consecutive days, and a day the schedule calls rest renders as *planned*, not as a gap. Volume badges pay for junk sets. Anything that makes a deload feel like a failure is working against the product's stated outcome.

The declared anti-goals are screen time, app opens, notification taps, streak preservation, and emotional dependence. *A great session is: open, get one clear decision, log efficiently, leave.* **Lower screen time can be a win** — which is an unusual thing for a consumer app to write down and hold itself to.

### 4. Say what the product does not know

The coach distinguishes a measured recovery signal from an absent one. Injury notes expire rather than silently constraining training forever. A generated plan built on thin evidence says the evidence is thin instead of projecting confidence it has not earned.

Under-claiming is cheap. Over-claiming costs the athlete's trust exactly once.

### 5. Do not imply clinical authority

The constraint system is conservative exercise exclusion — not diagnosis, not rehab prescription — and the code states this rather than letting the interface imply more. A fitness app is not a clinician and should not be built as though the distinction is a formality.

---

## Where the business thinking shows up

The engineering decisions in this repository are mostly product decisions wearing engineering clothes.

**Graceful degradation is a cost decision.** Every AI failure path falls back to the deterministic coach, and greetings are answered without a model call at all. The product stays usable if the AI budget is cut to zero — which means the LLM layer is a margin lever rather than a dependency.

**Provider abstraction is a supplier decision.** Three providers behind one precedence order; switching is an operational change, not an engineering project. The correctness contract lives in the client's validators, not in the vendor.

**Trust is the retention mechanism.** A coach that states one confident wrong number loses the athlete permanently, and no engagement feature recovers that. The grounding architecture is the retention strategy — it is not a compliance exercise bolted on afterwards.

**Verified deletion is a regulatory position taken early.** Account deletion refuses to complete while any storage object survives, because both buckets are public and an orphan stays readable by URL forever. Building this before it is demanded is cheaper than retrofitting it after.

**The unverified list is a risk register.** Knowing that the signed-in cross-user probe has never run, that nothing has been on a physical device, and that Sign in with Apple has never completed a real sign-in is what makes a launch decision a decision rather than a hope.
