# Testing and verification

How a system with a probabilistic component is held to a deterministic standard.

---

## Numbers

Derived from the repository at the pilot release candidate.

| | |
|---|---|
| Test modules | 165 |
| Assertions | 3,020 |
| Test code | 23,166 lines |
| Result | **165 passed, 0 failed** |
| Typecheck | `tsc --noEmit` clean under `strict`, `noUnusedLocals`, `noUnusedParameters` |
| Lint | 0 errors |
| CI | GitHub Actions — typecheck, lint, test on push and pull request |

Roughly one line of test code for every four lines of production code, concentrated on the modules where being wrong has a cost.

---

## A purpose-built runner

Tests run under `tsx` in Node. No Jest, no Babel, no preset, no transform config.

Each `*.test.ts` module exports a `run…Tests()` function that throws on failure; the runner discovers every test file under `src/`, imports it, calls the runner, and reports pass/fail with an exit code. React Native asset extensions are stubbed so modules referencing bundled illustrations can load — Metro hands back an opaque numeric asset id, so the stub is a faithful stand-in.

This is not minimalism for its own sake. It follows from the architectural constraint that the safety-critical modules import neither React Native nor Supabase: if the logic is pure, the runner can be trivial, and a trivial runner has no configuration that can drift out of sync with the build.

The trade-off is explicit. This runner does not render components, so the UI layer is verified by running the app rather than by snapshot tests. The choice was to put the testing effort where wrong answers are expensive — grounding, claims, progression, constraints, provenance — rather than spread thin across a component tree.

---

## Adversarial validation suites

The gates are tested by attacking them.

**Grounding tests build the context pack the way the engine builds one** — so the gate is exercised against the text the model actually receives, not a convenient fixture. The pack deliberately contains traps:

- An athlete note reading *"my bench is basically 500 kg on a good day"*
- A stored memory reading *"my squat max is 300 kg"*
- Real engine-authored figures alongside both

The suite then asserts what must **not** be groundable: that 500 and 300 are not evidence; that an unrecognised heading fails closed; that a note containing a blank line does not become quotable by splitting into its own block; that a later legitimate block after an untrusted one is still trusted; that body measurements do not ground training loads; that date digits do not ground counts.

**Claim-gate tests** cover the negation, question, and conditional guards — *"no plateau signal is active"* must pass, *"you've plateaued"* must not, unless the ledger permits it for that exercise.

**Constraint tests** are a matrix: for each equipment tier crossed with each limitation set, no excluded exercise survives into an executable plan, and every removal produces a permitted same-role substitute. The property is proved over the real function regardless of what proposed the exercise.

**Interpretation tests** cover the failure that motivated the rewrite — a low-confidence read must produce a question, and a malformed shape must reject the whole interpretation rather than a field.

The pattern throughout: test the invariant, not the example. *"An exercise the athlete cannot perform cannot appear in a generated plan"* is a property, and it is checked as one.

---

## Regression suites that pin decisions

Two suites exist to stop a specific class of silent regression.

When two development lines were reconciled into one release tree, the merge made decisions — which of two implementations survived, which duplicate was removed. A `reconciliation` suite asserts those outcomes, so the deleted path cannot quietly return through a later merge.

A `pilotFlags` suite does the same for feature gating. A purchase-availability flag was honoured at three of six routes to the paywall; two unguarded doors were closed centrally, and the suite now fails if either reopens.

This is what caught the bug in the first place. Both development lines had independently added a watcher on the same counter under different variable names — so they **auto-merged instead of conflicting**, and the teardown ran twice. It was survivable only because a guard ref was set synchronously first. A clean merge is not a correct merge.

---

## Security verified by probing

Reading a policy file tells you what was intended. Querying production tells you what is true.

A probe script queries **all 37 owned tables** plus the shared views using nothing but the publishable key that ships inside the app bundle. Signed out, nothing comes back: direct messages, coach memory, and the public profile view refuse outright; the rest return empty.

### What that found

A deferred migration was applied under supervision, and applying it immediately surfaced a real defect — which is exactly what its own header had asked someone to look for.

A summary function was **executable by `anon`**. The migration wrote `revoke all … from public`, which targets the `PUBLIC` pseudo-role. Supabase's default privileges grant `EXECUTE` on new functions **directly to `anon`**, and a direct grant survives a revoke from `PUBLIC`. The function was callable by anyone holding the key that ships inside the app.

| | before | after |
|---|---|---|
| Function called as `anon` | executed, returned `[]` | permission denied |

Closed by a follow-up migration, verified against production before and after.

**Two transferable lessons**, both recorded in the repository:

1. Grant and revoke **by role name**, and verify against production. `revoke … from public` reads like it closes the door and does not.
2. From the app's side, an empty result and a refusal are indistinguishable. **"It returned nothing" is never evidence that access is denied.** The function returned `[]` because the table was empty, not because the boundary held.

The first lesson is a Postgres detail. The second is the methodology, and it is the one that generalises.

---

## Verification contamination

An earlier verification pass wrote a workout to the production account and left it there. It then appeared in the coach's history and shifted every downstream calculation.

Two things came out of that:

- The row was identified and removed by **set semantics** — the provenance fields recording how a set was created — not by inspecting size or date, which would have caught the wrong row.
- A dev-write guard now gates writes from verification paths, so the tool used to check the system stops being able to change it.

Observability that mutates the thing it observes is not observability.

---

## Release gates

Run on the pilot release candidate.

| Gate | Result |
|---|---|
| `tsc --noEmit` | pass, clean |
| `npm test` | 165 passed, 0 failed |
| ESLint | 0 errors |
| `expo-doctor` | 18/18 checks passed |
| Production iOS export | pass — 7.09 MB Hermes bundle |
| Clean install (`rm -rf node_modules && npm ci`) | pass — the path EAS takes |
| Secret scan, source | clean — provider keys are `Deno.env.get` reads in edge functions only |
| Secret scan, shipped bundle | clean — no service-role key, no private API key, no dev endpoint |
| Cross-user probe, signed out | pass, against production |
| Migrations vs. production | 42 of 42 applied |
| Edge function | `ask-coach` deployed and serving |

### A trap worth documenting

`expo export` reuses Metro's transform cache, **including the inlined values of public environment variables.** Two consecutive exports with different environments produced byte-identical credentials — which reads exactly like "the environment is not being injected."

It is not. Always export with `--clear` when checking what credentials a bundle carries. Confirmed with a sentinel value. EAS builds clean, so real builds are unaffected.

---

## What is not verified

Stated plainly, because the rest of this document is verified and the difference is the point.

- **Nothing has run on a physical device.** The app has launched in the iOS Simulator and screens render; that is not device verification.
- **The signed-in cross-user probe has not run.** It needs two accounts. The harness is written and skips with the exact command to run it. This is the boundary test that matters most and it is the one still outstanding.
- **Sign in with Apple has never completed a real sign-in.** It is audited at source and the nonce pairing is correct — Apple receives the SHA-256, Supabase receives the raw — but audited is not working.
- **No EAS build has ever been produced** from this tree, because the project is not yet linked.

A test suite that passes tells you about the code. It tells you nothing about the parts of the system that have never been exercised, and a release document that blurs those two things is worse than no document.
