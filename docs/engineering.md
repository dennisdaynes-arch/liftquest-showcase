# Engineering

Decisions, trade-offs, and how the code was actually written.

---

## Scale

| | |
|---|---|
| Production TypeScript | 94,420 lines across 357 files |
| Test code | 23,166 lines · 165 modules · 3,020 assertions |
| Edge functions | 985 lines of Deno TypeScript across 3 functions |
| Migrations | 42 files, 2,644 lines of SQL |
| Screens / components | 39 / 50 |
| Commits | 580 on the release line, single author, first commit 2026-05-08 |

Three months, one person, from empty repository to a release candidate that passes its own gates.

---

## Decisions worth defending

### Pure domain logic, isolated from the framework

Every coaching module and every validation gate imports neither React Native nor Supabase. This is enforced by the fact that the test runner would fail if it did not hold.

**Why:** it makes the safety-critical logic executable in Node in seconds, and it makes the rules provable rather than asserted. **Cost:** a split-module pattern in several places — a pure half holding logic and tests, a thin impure half doing the I/O. That is more files than the naive layout. It is worth it wherever being wrong has a cost.

### A custom test runner instead of Jest

**Why:** the pure modules need no transform, so the runner can be ~60 lines with no configuration to drift. React Native testing setups are a recurring maintenance tax and they buy nothing for logic that already runs in plain Node.

**Cost, stated honestly:** no component rendering, so the UI is verified by running the app rather than by snapshot tests. This is a deliberate allocation — testing effort goes where wrong answers are expensive, not spread evenly across a component tree. A team with more hands would test both.

### Enforcement in code, not in the prompt

**Why:** prompt compliance is probabilistic and untestable, and two coaching surfaces persist their output to memory where a later call reads it back as fact. An invented number does not evaporate; it becomes history.

**Cost:** real complexity — number classification by domain, a claims ledger, section-level trust boundaries. Roughly 3,000 lines exist purely to check another system's output. The alternative was to trust the prompt and hope.

### Rejection over repair

A reply failing a gate is dropped, never edited, and no second model call is asked to fix it.

**Why:** an edited reply is a new unvalidated reply, and a judge model is another probabilistic component in the path that was supposed to make things deterministic. **Cost:** the user sometimes gets the engine's blunter wording instead of the model's. That is the intended failure, not a regression.

### RLS as the trust boundary, not the device

**Why:** client-side filtering cannot make the guarantee that matters. Blocking enforced in the database means content a blocked user posts never reaches the phone. Enforced on the device, it means the phone downloads it and chooses not to render — which is not the same product promise.

**Consequence:** a client-side block filter was later removed because it contradicted the RLS design its own module documented. Two enforcement points that disagree are worse than one.

### Deferring the linter, then landing it

ESLint, CI, and an error boundary were kept on a separate branch through the reconciliation. Adding a linter to a freshly merged 490-file tree buys a large diff and no user-visible safety at exactly the moment the diff needs to stay readable.

They are on the release line now, running in CI. The sequencing was the decision, not the exclusion.

---

## Release discipline

The repository maintains a single canonical release-state document with one governing rule: **where it contradicts another document, it wins, and the contradicting document is marked stale at its top.**

Every claim in it was produced by running something. Where a thing was not verified, it says so — in its own section, not buried in a qualifier. The distinction between "the test suite passes" and "this has never run on a device" is the entire point of writing it down.

Documentation that overstates readiness is worse than no documentation, because it converts an unknown into a false known. The same standard the grounding gate applies to the coach applies to the project's own claims about itself.

### The deploy script

Edge function deploys go through a guarded script that refuses:

- a checkout without the project config file
- a project-id mismatch
- a source missing any of the five required purposes
- uncommitted changes to the function

Each guard exists because the corresponding mistake happened. The Supabase CLI resolves configuration by walking up from the working directory — in a repository with multiple worktrees, it will silently read a *different* checkout's migrations and deploy from the wrong tree. That is not a hypothetical; it is why the first guard is first.

---

## Bugs worth reading about

A short list, because how a codebase handles being wrong is more informative than its feature list.

**The squat answer.** *"Give a bicep workout"* matched nothing in the keyword table, fell back to the athlete's registered goal lift, and returned a confident, fully-explained squat session. Every number was correct. The fix was not more keywords — it was moving interpretation to a model under a contract where an unsure reading asks a question instead of answering. *→ [ai-system.md](ai-system.md#gate-1--interpretation)*

**The default that became a fact.** Onboarding wrote a "sane default" bodyweight of 78 kg. It was then indistinguishable from a stated value, and it graded bodyweight-relative strength for someone who had told the app nothing. The fix was a provenance layer where unknown stays unknown. *→ [ai-system.md](ai-system.md#the-belief-layer)*

**The revoke that did not revoke.** `revoke all … from public` targets the `PUBLIC` pseudo-role; Postgres had granted `EXECUTE` directly to `anon`, and a direct grant survives a revoke from `PUBLIC`. A function was callable by anyone holding the app's shipped key. Found by probing production, not by reading policy files. *→ [testing.md](testing.md#security-verified-by-probing)*

**The clean merge that was not correct.** Two development lines independently added a watcher on the same counter under different variable names, so they auto-merged instead of conflicting and the teardown ran twice. Survivable only because a guard ref was set synchronously first.

**The avatar that stayed on the internet.** Replacing a profile picture uploaded a new object and never removed the old one. The bucket is public, so every avatar a person had ever set remained readable by URL. The same audit found that account deletion must verify storage is empty rather than assume it.

**The recap that would have lied.** Workouts that fail to save on a connectivity error queue locally. The queue holds only raw sets and dates — never a precomputed recap — because recaps compare against history, and history is unreachable exactly when the queue is used. Baking one in would fabricate "first workout!" and invented personal records.

The last one is the grounding principle applied to a subsystem with no AI in it at all: **when you cannot verify, do not assert.**

---

## Development approach

LiftQuest is built with AI-assisted engineering. Claude Code is the primary implementation tool, with Codex used alongside it. **Roughly 88% of commits carry an AI co-author trailer** — 668 of 757 across all branches.

Stating that plainly is the only version worth stating. It is trivially verifiable from the git history, and a portfolio claim that collapses under `git log` is worth less than no claim.

### What that actually looks like

The work that determines whether a codebase like this holds together is not typing:

**Specification.** Systems are specified to the level of an invariant before implementation. *"A number in a coach reply must appear in engine-authored evidence, within its own domain, with athlete notes explicitly excluded"* is a specification an implementation tool can execute against and a test suite can verify. *"Make the AI more accurate"* is not.

**Architectural decisions.** Where the engine/model boundary falls, that rejection beats repair, that RLS is the trust boundary, that the domain logic stays framework-free — these determine what the system can and cannot get wrong, and they are decided before code exists.

**Review and rejection.** Generated code that is plausible and wrong is the failure mode that matters. The recurring examples: a fallback that manufactures a default where the honest answer is unknown, an enforcement point added client-side where an existing one already lives in the database, a comment asserting a property the code does not have.

**Finding the failures.** The squat answer and the fabricated number were both found by using the product critically and noticing that a *correct-sounding* answer was answering the wrong question. Neither is a bug a coding tool surfaces for you.

**Setting the correctness bar.** The decision that a gate rejects rather than repairs, that a release document distinguishes verified from unverified, that a security boundary is probed rather than read — these are standards, and standards are chosen by a person.

### Why this is the honest framing

The interesting claim is not "I wrote 94,000 lines." It is that modern tooling raises the ceiling on what one person can build, and the binding constraint shifts from implementation speed to **judgment**: knowing what to build, specifying it precisely, recognising plausible-but-wrong output, and deciding what "correct" means for the product.

The validation architecture is the evidence. Nothing in this codebase required a grounding gate, a claims ledger, or a provenance layer in order to demo well. They exist because a coaching product that states a number the user acts on has an obligation the demo does not, and someone had to decide that mattered before anything was built to enforce it.
