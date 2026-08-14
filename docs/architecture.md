# Architecture

How LiftQuest is put together, and why the boundaries fall where they do.

---

## The organising constraint

One decision shapes the whole system: **a language model is never the source of a fact about your training.**

That forces a specific structure. The coaching logic cannot live inside a prompt, because a prompt cannot be unit-tested and its output cannot be verified. So the domain logic is pure TypeScript, the model is a phrasing layer bolted onto the outside of it, and the seam between them is a set of gates that can reject.

Everything below follows from that.

---

## Layers

```
iOS client (React Native / Expo)
├── Screens & components          39 screens, 50 shared components
├── Providers                     auth, active workout, preferences, onboarding, achievements
├── Deterministic coach engine    pure TypeScript — no React, no Supabase
├── Validation gates              pure TypeScript — synchronous, testable
└── Data access                   supabase-js, typed against the schema

Supabase
├── PostgreSQL                    37 tables, RLS on all 37, 78 policies, 42 migrations
├── Auth                          email + Sign in with Apple
├── Storage                       avatars, feed photos
└── Edge Functions (Deno)         ask-coach · delete-account · redeem-referral

LLM provider                      reached only through ask-coach
```

### Why the engine is pure

Every module in the coaching engine and every validation gate is written with no React Native and no Supabase imports. This is a deliberate constraint, not an accident of layout, and it buys three things:

1. **The safety-critical logic runs in Node under `tsx`** — no Jest, no Babel, no transform config, no simulator. Tests execute in seconds against the real modules.
2. **The rules are provable rather than asserted.** "An exercise the athlete cannot perform cannot appear in a generated plan" is a property the test matrix demonstrates over the actual function, not a claim in a comment.
3. **The seam stays honest.** A module that cannot query the database cannot quietly invent a fallback when data is missing; it has to return "unknown" and let the caller decide.

Where a module genuinely needs I/O, it is split: the pure half holds the logic and the tests, the thin impure half does the invoking. The coach's context-pack assembly and its LLM call are separated exactly this way — the pack builder takes today's date as a parameter rather than reading the clock, so the same input always produces the same pack.

---

## The coaching engine

The deterministic core, roughly in the order a session flows through it.

| Concern | What it computes |
|---|---|
| **Session blueprint** | Movement-pattern slot templates (main → secondary compound → accessories → isolation → core), varied by goal, mode, and available time, using recent history to avoid hammering the same lifts consecutively |
| **Progression** | Double progression from logged sets: hit the top of the rep range within the RPE cap → add load; inside the range → add reps; over the cap → back off; flat for three sessions → rebuild from a lower load |
| **RPE calibration** | Whether this athlete's reported RPE is trustworthy, derived from how their reported effort tracks their actual performance |
| **Prescription grading** | Whether the last prescription worked, graded against what was actually logged — the input to any "that adjustment worked" claim |
| **Load management** | Accumulated workload, deload verdicts, training-gap accounting |
| **Plateau detection** | Per-lift stall detection with a cause attribution pass |
| **Constraint contract** | Equipment tier and physical limitations → the set of exercises that may appear in an automatic plan |
| **Belief layer** | Provenance, confidence, and staleness for everything the app thinks it knows about the athlete |

Every prescription carries a **receipt** — the actual numbers the decision was made from. This is not a debugging affordance; it is what makes the model's claims checkable. A claim the engine's own receipt already states is automatically permitted. The model may echo the engine. It may never outrun it.

### Training gap vs. logging gap

One example of the domain modelling being more careful than it looks. A month with no rows can mean the athlete stopped training, or it can mean they stopped logging. These call for opposite responses: the first justifies reducing load, the second does not.

The engine treats missing rows as **a question, not a verdict**. Load stays conservative either way — that is the safe direction — but the *claim* stays honest: the coach does not tell you that you detrained when all it knows is that you did not open the app.

---

## The AI path

A chat turn, end to end:

1. **Trivial short-circuit.** Greetings and thanks are answered instantly from templates, client-side. Zero model calls — reaching "warm" is not worth a second of latency or a slice of the daily budget.
2. **Interpret.** The message goes to the model as a cheap structured pass. It returns intent, topic, constraints, and references as strict JSON. It asserts nothing and cites nothing, so it receives neither the evidence pack nor the honesty rules — it decides *what was asked*, never *what is true*.
3. **Narrow.** Every field of that JSON is validated against a closed vocabulary. An unrecognised shape rejects the whole interpretation and the caller degrades to a keyword fallback, rather than acting on half a reading. An interpretation that is unsure of itself routes to a **clarifying question**.
4. **Compose evidence.** The engine assembles the answer: the live diagnosis readout, coach memory, per-lift trends, plateau detail, the cached plan, recent load. This is the only source of any figure.
5. **Build the context pack.** Evidence is folded into one grounding block with engine-authored section headings. Those headings are load-bearing — see below.
6. **Speak.** The pack, the question, and a few prior turns go to `ask-coach`, which calls the provider and returns prose.
7. **Gate.** Number grounding, then claim gating, then constraint checks. Pass → render. Fail → **drop the reply** and render the engine's own wording.

Steps 4 and 7 are the architecture. The model sits between an evidence composer it cannot influence and a validator it cannot see.

### Why section headings carry weight

The context pack is a single text block, so the gate needs to know which parts of it are evidence. Only blocks starting with a known engine-authored heading count. `ATHLETE NOTES` and `MEMORY` are deliberately absent from that list — they carry the lifter's own words, and a note reading *"my bench is basically 500 kg on a good day"* must never license the coach to say it back as fact.

Anything unrecognised fails closed, including a note containing a blank line that splits into its own block. The failure mode of a fail-open design here is a fabricated number becoming durable history.

---

## The edge function boundary

`ask-coach` is the only path from LiftQuest to any language model.

- The client **never holds an API key.** Provider credentials live in Supabase secrets, set by the owner, read via `Deno.env.get`.
- Calls carry the caller's JWT. Reads and writes touch only that caller's chat log; the rate limit and the recorded usage are per user.
- Five purposes share the function — `interpret`, `chat`, `coach_read`, `weekly_review`, `plan_notes` — each with its own output contract. Three return prose; `plan_notes` and `interpret` return strict JSON the client re-validates.
- Provider is chosen by which secret is set, in precedence order: Anthropic → OpenAI → Gemini. Swapping providers is an operational change, not a code change.
- Every input is bounded: question length, turn count, per-turn length, context size, daily call ceiling.
- **Any non-200 is "use the deterministic reply."** No key set, not entitled, rate limited, bad input, upstream failure — all the same to the client. An outage makes the coach blunter, never broken.

The other two functions exist because they need privileges the client must not have. `delete-account` performs a full erasure with service-role access; `redeem-referral` writes across two users' rows atomically.

---

## Data model

37 tables in PostgreSQL, grouped by what they are for.

| Group | Tables |
|---|---|
| Training | `workouts`, `workout_sets`, `workout_recaps`, `workout_achievements` |
| Generated plans | `generated_workouts`, `generated_workout_exercises`, `generated_workout_sets` |
| Coach state | `coach_memory`, `coach_beliefs`, `coach_chat_log`, `workout_memories` |
| Athlete model | `profiles`, `user_goals`, `user_training_preferences`, `body_measurements`, `recovery_checkins` |
| Wearables | `watch_connections`, `watch_metrics` |
| Social | `feed_events`, `feed_comments`, `reactions`, `friendships`, `direct_messages`, `blocked_users`, `notifications` |
| Gyms | `gyms`, `user_gyms`, `user_gym_equipment`, `gym_equipment_contributions` |
| Nutrition | `nutrition_targets`, `daily_nutrition_summaries` |
| Import | `import_jobs`, `imported_exercise_mappings`, `imported_personal_records` |
| Ops | `live_sessions`, `referrals`, `crash_reports` |

### Security posture

**RLS is enabled on all 37 tables, with 78 policies.** The device is not a trust boundary; the database is.

Two consequences worth naming:

- **Blocking is enforced in RLS.** A shared predicate returns false when either party has blocked the other, so the feed and every friends-gated policy refuse the rows outright, and direct messages carry the check explicitly. Content a blocked user posts never reaches the phone — a property client-side filtering structurally cannot provide.
- **Cross-user reads go through a restricted view.** An early policy made profiles readable by any authenticated user; it was dropped. Other-user reads now go through a six-column non-PII view, granted to `authenticated` and revoked from `anon`.

35 of 37 tables cascade-delete from `auth.users`. The two that do not are deliberate: crash reports de-identify rather than vanish, and the gyms table carries no identity column at all.

### Deletion is verified, not assumed

`delete-account` clears storage folders page by page and **refuses to delete the account if any object survives.** Both buckets are public by design — an orphaned object stays readable by URL for the life of the internet, so "we deleted the row" is not deletion.

The same reasoning fixed a quieter bug: replacing an avatar did not remove the object it superseded, so every profile picture a person had ever set remained publicly readable.

---

## Client state

State is deliberately layered rather than centralised in one store.

| Scope | Mechanism |
|---|---|
| Session/auth | React context provider over the Supabase session |
| Cross-screen slices | Small purpose-built context providers (active workout, preferences, onboarding, achievements) |
| Screen-local | React state, kept local where nothing else needs it |
| Durable local | Thin AsyncStorage stores, each a pure I/O module tests can stub |

The active-workout provider mirrors only the small slice the floating pill needs — duration and current exercise — rather than lifting the whole session into global state. Global state that nothing reads is a liability.

### Two durability mechanisms

**Session snapshotting.** A workout in progress is persisted on every state change, tagged with user and timestamp. A crash, a backgrounding, or an OS kill restores sets, pause state, and timer. Snapshots belonging to another user or older than 14 hours are rejected on load.

**A finish queue that refuses to lie.** If saving a completed workout fails on a connectivity error, the raw session is queued locally and replayed on the next foreground. The queue deliberately holds **only raw facts** — sets, title, dates — never a precomputed recap. Recaps compare against history, and history is unreachable exactly when the queue is used; baking one in would fabricate "first workout!" and invented personal records. The drainer recomputes the recap honestly once online.

That is the same principle as the grounding gate, applied to a completely different subsystem: when you cannot verify, do not assert.

---

## Build and release

- **Expo SDK 54** with the New Architecture enabled, `expo-updates` for OTA delivery.
- **EAS Build** with remote version source and auto-increment; preview and production channels share a base environment definition.
- **GitHub Actions CI** runs typecheck, lint, and the full test suite on push and pull request.
- **Edge function deploys go through a guarded script** that refuses a checkout without the project config, a project-id mismatch, a source missing any of the five purposes, or uncommitted changes to the function. This exists because the CLI will happily deploy from the wrong worktree, and it did.
