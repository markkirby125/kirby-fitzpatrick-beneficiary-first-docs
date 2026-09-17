# Beneficiary First Docs — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [Writing Online Is Hard, Until Experts Do This](https://www.youtube.com/watch?v=j1JRSan9CYg)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: The Beneficiary Slot vs. the Permission Frame

A function docstring is written from the *maximum-knowledge position* — inside the implementation, by the person who just finished building the machinery. It is read from the *minimum-knowledge position*: outside, at a call site, by someone who owns a deadline and no internal model. Documentation fails not because it is inaccurate but because it is **written from the wrong side of the call boundary**.

The default artifact that results is a **Permission Frame**: a sentence whose grammatical subject is internal machinery and whose predicate is *access, exposure, allowance, enabling*. It describes what the code is permitted to touch. The reader's actual question — *what do I get?* — is never answered, so the docstring must be decoded against the source before it can be used.

**The Beneficiary-First Standard** is a constraint on grammatical subject and predicate, not a tone preference: *every sentence about a callable names its beneficiary (the caller, or the caller's user) in subject position and states the yield as the predicate.* The mechanism belongs in a subordinate clause or, more often, nowhere — it is already visible in the body, and it is what `git blame` and the profiler are for.

Five load-bearing definitions:

1. **Beneficiary** — the specific actor who consumes the artifact and receives value. For a public SDK it is the integrating developer; for an internal helper it is the maintainer and the 03:00 on-call engineer; for an end-user-facing method it is the *user*, with the caller acting as a courier. Naming the beneficiary is a design decision that must be made once and held consistently across the file.
2. **Yield** — the concrete thing the beneficiary receives: a returned value, a guaranteed state change, a resource that is now released, a failure that is now impossible. A yield is a noun the caller can hold, or a state delta they can assert. *"more control"*, *"flexibility"*, *"the ability to…"* are not yields.
3. **Permission Residue** — mechanism vocabulary that leaked out of the body into the summary: *cache, mutex, pool, adapter, DTO mapper, buffer, handler, transport, internally*.
4. **Beneficiary Distance (BD)** — the number of hops from the sentence's grammatical subject to the actor who receives value. `"Allows the settlement module to access the ledger adapter"` has BD = 3 and terminates at a machine, not a person. Target: **BD = 0** (the beneficiary *is* the subject), ceiling **BD = 1** (*"You get a `SettlementReport`…"*).
5. **Reading Position** — the three places a doc is consumed from, which the sentence must be written *toward*: the call site (composition), the incident (diagnosis), the upgrade (migration). A permission-framed sentence serves none of them; a yield-framed sentence serves the first and third immediately and the second via its failure clause.

```text
[ANTI-PATTERN: Permission Frame — implementation as subject]
  ┌───────────────────────────────────────────────────────────────────────┐
  │ def settle(tenant_id: str) -> SettlementReport:                       │
  │     """Allows the settlement module to access the ledger adapter and  │
  │        permits flush operations on the internal pending-events        │
  │        buffer, enabling the pipeline to continue."""                  │
  └───────────────────────────────────────────────────────────────────────┘
   caller's question:  "...so what do I get back?"
   beneficiary distance:  module ─► adapter ─► buffer ─► pipeline ─► nobody
   yield:                undefined
   reader's next action: open the source, or write a test to find out
   cost at the call site: 4–10 minutes per lookup, × every call site, × every
                         developer, × every version of the SDK

[BENEFICIARY-FIRST: yield as subject, mechanism demoted]
  ┌───────────────────────────────────────────────────────────────────────┐
  │ def settle(tenant_id: str) -> SettlementReport:                       │
  │     """Return this tenant's settled ledger.                           │
  │                                                                       │
  │        You get: entries that are final (they will not move again)     │
  │        and `deferred`, the list of items that need a second run.      │
  │        You give: a tenant_id you already own. Raises TenancyError     │
  │        if the tenant is unknown — settle() never returns partial.     │
  │     """                                                               │
  └───────────────────────────────────────────────────────────────────────┘
   beneficiary distance:  you ─► report           (BD = 1, one hop, human)
   yield:                final entries + deferred list + no-partial guarantee
   reader's next action: compose the call
   cost at the call site: 0 lookups
Result: the docstring is a usage surface, not a description of internals.
```

```text
READING POSITIONS — why the frame must change per surface
                                                    ┌─ call site: "what do I get?"
  author (inside)  ──── publishes ────►  artifact ───┼─ incident:  "what can it do wrong?"
                                                    └─ upgrade:   "what changed for me?"

  Permission Frame answers: none of the three.
  Benefit Frame answers:     call site (yield), incident (failure clause),
                             upgrade (changelog entry restates the same yield).

The SAME system change, correctly framed per beneficiary:
  • integrator  — "your calls now multiplex over one connection"
  • on-call     — "a dead pooled connection surfaces as ConnectionReset, not a hang"
  • maintainer  — "transport selection moved into DialOption"
  One mechanism, three yields. Writing one sentence for all three serves none.
```

**Why this binds harder in software than in prose.** Code is a contract with a machine and a *promise* to a human; the compiler verifies the first and nobody verifies the second, so docstrings rot silently into permission ledgers. Every downstream autocorrect amplifies the error: generated SDK reference pages, IDE hover tooltips, LLM retrieval for code assistants, and changelogs all inherit the docstring verbatim. A permission-framed docstring does not stay a docstring — it becomes the text that an agent feeds back to a developer who cannot see the source at all. There is no reader positioned to notice that it never said what you get.

**The twin-beneficiary rule.** A function has multiple potential beneficiaries; picking the wrong one is the most common source of *technically correct, useless* docs. Public surface → write to the integrator. Private helper → write to the maintainer, and the yield is *"you can now change X without touching Y"*. Nothing is written *to the machine*: `"the function takes a string and returns a report"` is a type signature restated in English and is the purest zero-beneficiary form.

---

## 2. Core Transformation Protocols

1. **Fix the beneficiary before writing the sentence.** Name it aloud: *"This docstring is for the caller integrating v3."* If you cannot name one, you are about to write a type signature in prose. Write the beneficiary down in the file header when the surface is mixed (public module + internal helpers), so every sentence has a position to aim at.

2. **Put the yield in the first 8 words.** The summary line is the only line many readers ever see — it is what the IDE hover shows, what the generated reference page lifts, what the retrieval index stores. `"Return this tenant's settled ledger"` (6 words) before any subordinate clause. Apply the [Locomotive Syntax](../../kirby-fitzpatrick-locomotive-syntax/SKILL.md) engine invariant to documentation exactly as you would to a commit subject.

3. **Answer the caller's three questions in the caller's order.** *What do I get?* → *What do I hand you?* → *What can go wrong?* This is the reading order at a call site, so it is the publication order of the block. Reordering to *parameters → returns → errors* (the generator's default) is machine order, not human order.

4. **Delete permission vocabulary.** *allows, permits, enables, provides the ability to, exposes, grants access to* describe a right, never a result. Replace with the result: `"allows the caller to access the cached user"` → `"Return the cached user record, or None if it was evicted."` See the forbidden-token table (2.3).

5. **Convert mechanism verbs into outcome verbs.** *locks, drains, wraps, delegates, sanitises, normalises* are implementation facts. Each one has a caller-visible consequence; write the consequence. *drains the pending-events buffer* → *folds every pending event into the returned entries, so nothing is left for the next run*.

6. **Name the yield as a noun the caller can bind.** `SettlementReport whose entries are final`, `a Connection you must Close`, `the number of rows that changed`. If you cannot name the type or the state delta, you do not yet know what the function promises, and the docstring is covering for a design defect — escalate that, do not paper over it.

7. **State the guarantee, not the activity, in the return clause.** *"Settles the ledger"* says what the code does. *"entries are final and will not move again"* says what the caller may now rely on. Guarantees are what callers write tests and retries against; activities are what maintainers already know.

8. **Frame every failure as a yield the caller protects.** `Raises TenancyError if the tenant is unknown — never returns a partial report.` The second clause is the load-bearing half: it tells the caller what they *never* have to check. An error clause that only names the exception class is a permission frame wearing an exception.

9. **Demote internals to a labelled clause or delete them.** Internal vocabulary survives in exactly two cases: it names a resource the caller must manage (a `Close`, a lock they hold, a bounded channel), or it explains a cost the caller pays. Everything else is `# Implementation` comment material.

10. **One yield per callable in the summary line.** `"Validate the config and connect to the broker and start the watcher"` is three promises that will diverge. Split the function or pick the promise the caller is actually optimising for and name the rest as scope.

11. **Restate the same yield in every downstream surface.** Docstring, IDE hover, reference page, changelog entry, and migration note must describe the *same* yield in the *same* terms. Divergence means the integrator cannot tell whether two texts describe one change — apply the [Anti-Author-Splain Commenter](../../kirby-fitzpatrick-anti-author-splain-commenter/SKILL.md) test to each restatement.

12. **Write the beneficiary into the negative space.** Say what the caller does *not* get: `"does not retry", "no ordering guarantee across tenants", "not thread-safe"`. Silence reads as *guaranteed*, and an unstated silence is the most expensive misreading in an API surface — the [Cold Reader PR Auditor](../../kirby-fitzpatrick-cold-reader-pr-auditor/SKILL.md) treats every such silence as a finding.

13. **Beneficiary-first is publication-time, not drafting-time.** Draft from inside the implementation, in any voice, with all the mechanism you like. The transformation happens on the second pass, when you already know the yield: hoist it, demote the mechanism, and delete the residue.

### 2.1 Canonical slot order of a beneficiary-first doc block

| Slot | Content | Budget | Failure if missing |
|---|---|---|---|
| Y — Yield | What the beneficiary receives, in subject position | ≤ 8 words | Docstring is a signature in English; caller reads the source |
| G — Give | What the caller must supply or already own | 1 line | Reader reverse-engineers required preconditions from parameter types |
| S — Success shape | The guarantee: what is now final, atomic, idempotent, ordered | 1–2 lines | Caller cannot write a test or a retry against it |
| F — Failure shape | Errors + what is *never* returned (partial, stale, silent) | 1–2 lines | Every failure path is discovered in production |
| C — Cost / blast radius | Latency class, network hop, resource to release, thread-safety | 1 line | Incident diagnosis starts from the profiler |
| N — Negative space | What is explicitly not included, not guaranteed, not done | 1–2 lines | Silence is read as a guarantee |

### 2.2 Transformation table: permission frames and clean replacements

| Anti-Pattern (permission frame) | What it costs the reader | Clean Replacement (beneficiary-first) |
|---|---|---|
| `"""Allows the caller to access the ledger adapter."""` | States a right; the reader still does not know what comes back | `"""Return the tenant's settled ledger. You get final entries plus a deferred list."""` |
| `"""This function is responsible for handling user authentication."""` | "Responsible for" is a job description; no yield, no failure, BD = 1 to a machine | `"""Return a `Session` for valid credentials. Raises `AuthFailed` on bad password — never returns an anonymous session."""` |
| `"""Wraps the fetch call and exposes retry configuration."""` | Mechanism-first; the reader must infer what retries *mean for them* | `"""Return the response, retrying idempotent GETs up to 3 times (≈900 ms worst case)."""` |
| `"""The config is validated and the connection pool is initialised."""` | Passive voice, zero beneficiary, activity-not-guarantee | `"""You get a client whose pool is warm: the first call costs no handshake. Raises `ConfigError` before any socket is opened."""` |
| `"""Provides utility helpers for date formatting."""` | Symmetrical non-information; true of every module ever written | `"""Format a UTC instant as the tenant's local calendar date, so a nightly report never straddles midnight."""` |
| `"""Permits callers to pass a custom logger."""` | Permission frame; the consequence of passing one is unstated | `"""Route this module's logs to your logger. Without it, logs go to the std logger and are dropped in tests."""` |
| `"""Improved performance and refactored internals."""` (changelog) | No beneficiary, no yield, no action required by anyone | `"""**You get** p95 480 ms → 210 ms on `reconcile()` at 100k invoices. **You do nothing** — no signature, config, or wire change."""` |
| `"""Enables HTTP/2 support."""` (changelog) | Names a protocol, not what the integrator now holds | `"""**You get** one connection multiplexing all in-flight calls, so a 40-parallel fan-out stops exhausting the pool."""` |
| `"""See source for details."""` | Defers the promise to the implementation, which cannot state intent | One yield sentence + the guarantee the diff cannot express |
| `"""Thread-safe."""` (unqualified) | Reads as a blanket guarantee; usually scoped to one method | `"""Safe to call from multiple goroutines; the returned handle is not — one handle, one goroutine."""` |

### 2.3 Forbidden permission tokens

| Banned token | Why it fails | Required replacement |
|---|---|---|
| *allows*, *permits*, *lets you* | Names a right, not a result | The result: "returns X", "you get Y" |
| *enables*, *makes it possible to* | Zero-information modal; any function enables something | The outcome the caller can assert |
| *provides the ability to*, *gives you access to* | Permission frame in noun form | The object returned, or the state delta |
| *is responsible for*, *handles*, *deals with* | Job description; usually a one-to-many mapping | The single promise this call makes |
| *exposes*, *wraps*, *delegates to*, *sits on top of* | Mechanism-first; BD ≥ 2 | The caller-visible consequence |
| *internally*, *under the hood*, *behind the scenes* | Invites the reader into the implementation | Delete, or label as `# Implementation` |
| *utility*, *helper*, *various*, *general-purpose* | Non-information booster | The concrete yield of the one thing it does |
| *should*, *can be used to* | Hedge; an API promise stated as a possibility | The guarantee, in the present tense |
| *seamlessly*, *easily*, *simply*, *just* | Difficulty is the caller's to judge; asserting it destroys trust | Delete; state the cost instead |
| *various options*, *flexible* | Flexibility is unpriced; the caller must read the source to find it | Name each option and when each is correct |

### 2.4 Failure diagnostics

| Symptom | Beneficiary diagnosis | Fix |
|---|---|---|
| "What does this actually return?" | Yield slot missing; summary states a right or an activity | Hoist the yield into the first 8 words |
| Hover tooltip is useless, everyone opens the source | Summary line is a permission frame | Rewrite summary only — it is the only line they see |
| Integrator asks about preconditions in issues | Give slot missing | Add one line: what the caller must already own or acquire |
| Same bug reported as "silent data loss" | Failure shape missing; silence read as guarantee | Add "never returns partial" / "does not retry" clause |
| Changelog is skimmed and nothing gets upgraded | Entry names internals, not the integrator's new capability | Restate the yield + the zero-action ("you do nothing") |
| Generated reference pages look machine-written | Docstring mirrors the signature | Apply the three-question order; delete type restatements |
| Two reviews disagree about what a method promises | Beneficiary never chosen; public and internal readers in one sentence | Split public docstring (integrator) from `# Implementation` note (maintainer) |
| Docstring accurate but the answer arrives 4 paragraphs down | Yield is present but demoted under mechanism | Apply slot order 2.1; mechanism to the last clause or out |

**Related dispatchers.** Hoist the yield into the opening subject position with [Reverse Question Inversion](../../kirby-fitzpatrick-reverse-question-inversion/SKILL.md); bind it to its warrant with [Here's Why Inversion](../../kirby-fitzpatrick-heres-why-inversion/SKILL.md); front-load the beneficiary within the first 7 words via [Locomotive Syntax](../../kirby-fitzpatrick-locomotive-syntax/SKILL.md); strip the modal and intensifier residue with the [Empty Verb Extractor](../../kirby-fitzpatrick-empty-verb-extractor/SKILL.md) and [Lexical Anti-Bloat Filter](../../kirby-fitzpatrick-lexical-anti-bloat-filter/SKILL.md); remove "In this section I will describe…" wrappers with [Zero Meta Discourse](../../kirby-fitzpatrick-zero-meta-discourse/SKILL.md); rebuild the surrounding document skeleton with [Cathedral Taxonomy](../../kirby-fitzpatrick-cathedral-taxonomy/SKILL.md); and verify the yield survives a 15-second skim with the [Skim Test Outliner](../../kirby-fitzpatrick-skim-test-outliner/SKILL.md).

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — Auditing Docstring Diffs and Framing the Comment

Two applications. First, the reviewer's own comment is an artifact with the same constraint: state what the *author* receives (a demanded state, in subject position), then the mechanism, then the check. Second, a docstring diff is reviewable evidence in the same way a test diff is — and it is the cheapest place in the codebase to catch a design defect, because a function whose yield cannot be named is usually doing too much.

**Comment rule.** Open with the demanded yield for the author, not the reading order you walked. Never *"I was reading settlement and noticed it seems like…"* — the author needs the state you are asking for.

**Before — permission-framed diff, comment that narrates the reviewer's path:**

```python
def settle(tenant_id: str) -> SettlementReport:
    """Allows the settlement module to access the ledger adapter and
    permits flush operations on the internal pending-events buffer."""
```

> So I was looking through the settlement module and it looks like this docstring is kind of describing internals rather than what it returns, which makes it hard to use. Maybe rewrite it?

**After — the comment is itself beneficiary-first, with a falsifiable check:**

```markdown
**Blocking — name the yield in the first 8 words.**
The summary describes what the module *may touch* (adapter, buffer); the caller only needs
what comes back. Ask: if `SettlementReport` were deleted tomorrow, would this sentence be
wrong? Today, no — which means the sentence is not about the return value at all.

Required shape:
    """Return this tenant's settled ledger.

    You get: entries that are final, plus `deferred` for a second run.
    You give: a tenant_id you already own. Raises `TenancyError` — never
    returns a partial report.
    """

Check: `grep -rn "allows\|permits\|enables\|responsible for" src/settlement/` → 10 hits today.
Also verifiable: every `# Implementation` note we deleted should appear in the body's comments,
which they already do (`settle.py:41-96`). No information is lost, only relocated.
```

**Author's response rule (the strong form).** Reply with the amended promise, not the deliberation: *"Rewritten — `settle()` now opens with 'Return this tenant's settled ledger'; the buffer and adapter are named only in the `# Implementation` note (commit `c4a91f2`)."* A reply that opens with *"I wrote it that way because the flush ordering is subtle…"* restarts the loop and makes the reviewer audit a rationale that belongs in a code comment.

**Thread acceptance rule.** A docstring thread closes on a stated yield, not a sentiment: the caller-visible promise, in which commit, verified by which usability check (`grep` for banned tokens, hover-text read-through, one new call site written from the docstring alone). The last of those is the real test — if you cannot write a call without opening the source, the docstring is still a permission ledger.

### 3.2 PR Descriptions & SDK Changelogs — "What you get" Replacing "What we enabled"

The PR body and the changelog entry are the *durability layer* for the integrator: the diff records mechanics forever; the yield survives only if someone writes it down. This is the highest-leverage beneficiary-first surface in the repository, because the changelog is the only document an integrator reads before upgrading — and it is very often the only document an LLM retrieves when answering *"what changed in v3.2?"*.

**Before — a changelog entry with no beneficiary and no yield:**

```markdown
## 3.2.0
- Refactored transport internals; enabled HTTP/2 support.
- Improved performance for the settlement path.
- `settle()` now returns a `SettlementReport` instead of a tuple.
- Internal: pool sizing moved to `DialOption`.
```

Diagnostics: four changes, zero yields, one silent breaking change (`tuple` → `SettlementReport` is listed as a fact, not as *the thing the integrator must fix*), and one item (`DialOption`) that no integrator can act on.

**After — each entry answers the same three questions in the integrator's order:**

```markdown
## 3.2.0

### What you get
- **One connection instead of a pool of sockets.** All in-flight calls now multiplex over a
  single HTTP/2 connection, so a 40-parallel fan-out no longer trips the 32-connection limit.
  You do nothing — this is on by default.
- **Failures instead of hangs.** A dropped connection now surfaces as `ConnectionReset`
  within ~2 s (was: indefinite block until the caller's deadline). If you already wrap calls
  in a retry, your retry now runs 20 minutes sooner.

### What you must change
- **`settle()` returns a `SettlementReport`, not a 3-tuple.** Unpacking code must switch to
  `.entries`, `.deferred`, `.total`. Codemod: `npx @acme/codemod settle-3.2` (dry-run first).
  This is the only breaking change in 3.2.0.

### What you must know but need not act on
- `entries` are *final*: the same call twice returns identical entries, no event is folded
  twice. If you relied on re-settling to pick up late events, use `deferred` instead.
```

**PR-body rule for SDK work.** Open with a **What you get / What you must change / What you may ignore** triptych before any mechanism. The mechanism section then exists to make those statements defensible, not to lead the reviewer through the design journey — that is the [Here's Why Inversion](../../kirby-fitzpatrick-heres-why-inversion/SKILL.md) pass applied to a docs-bearing PR. Reviewers accept the change or reject the *promise*; a reviewer who has to reconstruct the promise from the diff cannot do either, and the thread degenerates into terminology.

**Parity rule.** The promise text must be byte-compatible across four surfaces: PR body, changelog entry, docstring summary line, and migration guide. If the docstring says *"final entries"* and the changelog says *"improved settlement reliability"*, no integrator can tell whether those describe one change or two — and every other claim in both documents loses credibility. Enforce at review time with a grep for the summary line of each changed public callable.

### 3.3 Architecture RFCs / ADRs — Decisions Stated as Beneficiary Outcomes

The conventional ADR template is written from the architecture's point of view: it explains the system, then the choice. The beneficiary-first variant keeps the same sections and rewrites the *decision line* as a yield statement, so a reader can answer "does this affect me, and how?" without reading the drivers. This matters most in an organization where the ADR is the only artifact a downstream team will ever see.

**Decision line rule.** The `Decision` field names the beneficiary and the yield, in that order, before naming the mechanism: *"Consumers of `reconcile()` get identical results from repeated calls"* — not *"We adopt idempotency keys in the reconciliation read path."* Mechanism goes in the drivers.

**Rejected alternatives rule.** Each rejected option is recorded as *what the beneficiary would have given up*, plus the axis measured and the condition that revives it. This is the highest-value table in the document: it is what prevents the same option being re-proposed nine months later, and it is what lets the reader who liked the rejected option understand the trade they cannot see.

```markdown
# ADR-014 — Idempotent settlement for SDK consumers

## Status
Accepted — 2026-04-02 · Owner: billing-svc · Reverses if reconciliation moves off the
read path, or if `deferred` grows beyond 1% of entries in production (see Open questions).

## Decision
**SDK consumers get the same `SettlementReport` from a repeated `settle()` call**, so a
retry after a network timeout can no longer double-post a ledger entry. Callers who relied
on re-settling to pick up late events use `deferred` instead.

## What each beneficiary receives
| Beneficiary | Yield | What they must do |
|---|---|---|
| SDK integrator | Identical results on retry; no double-post | None — signature unchanged, `.entries` now final |
| On-call engineer | `ConnectionReset` at ~2 s instead of an indefinite hang | Nothing; existing retries now run 20 min sooner |
| Maintainer | Duplicate-detection lives in one place (`SettlementKey`) | Delete the ad-hoc dedupe at `handler.ts:88` |

## Drivers
1. 11 of 14 settlement incidents in FY26 were retry-induced duplicates (`INC-4501`, `INC-4522`).
2. Cost of the fix is bounded to the read path: writes, refunds, and payouts are untouched.
3. The alternative — asking integrators to carry a client-side idempotency key — moves the
   promise onto the party least able to keep it.

## Rejected alternatives (and what the beneficiary would have lost)
| Option | Axis measured | What you would give up | Revive if |
|---|---|---|---|
| Client-supplied idempotency key | 0 server changes | Your retry safety becomes your bug; 3 of 5 pilot teams skipped it | We ship an HTTP layer without a shared key store |
| Response cache | p95 unchanged | Stale payouts on read-after-write; the multiplier is hidden, not removed | A read-only reporting endpoint appears |
| Cap-and-warn instead of fixing | 4 days vs 14 | 1-in-9 retries still double-posts; on-call keeps absorbing it | Duplicate rate falls below 1 incident/quarter |
```

**RFC variant.** A long-form RFC opens with a **Beneficiary Summary** block — `Who is affected`, `What they receive`, `What they must change`, `Date of effect` — before any motivation or design. The design section then becomes the warrant: it exists to make the beneficiary summary defensible. Design journals, spike logs, and transcript fragments stay out of the RFC; anything the reader genuinely needs from them compresses into the drivers list, one line per item.

---

## 4. Verification Checklist

- [ ] **The yield is in the first 8 words of every public summary.** The IDE hover, the generated reference page, and the retrieval index all lift the summary line alone; a reader who stops there can name what they receive without opening the source, and no summary leads with mechanism or with a permission verb.
- [ ] **Zero permission-token residue.** A grep for `allows |permits |enables |provides the ability to|responsible for|exposes |internally|under the hood|utility|flexible` returns no hits in public docstrings, and no sentence names a beneficiary more than one hop from its subject (BD ≤ 1).
- [ ] **The caller's three questions are answered in the caller's order.** Every doc block resolves *what do I get* → *what do I give* → *what can go wrong*, with the failure clause stating what is **never** returned (partial, stale, silent) rather than only naming an exception class.
- [ ] **Every documented callable names a yield a caller can bind and assert.** The return is described as a value or a state delta (`report whose entries are final`, `a Connection you must Close`), and no docstring is a restatement of the type signature in prose; any callable whose yield cannot be named is escalated as a design finding, not documented around.
- [ ] **Parity and negative space hold across surfaces.** The docstring summary, PR body, changelog entry, and migration note describe the same yield in the same terms; each changelog entry states the integrator's new capability *and* whether any action is required; and scope, cost, thread-safety, and not-guaranteed items are explicit rather than silent.