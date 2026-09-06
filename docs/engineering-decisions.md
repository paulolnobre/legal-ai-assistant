# Engineering decisions

Each entry states the decision, the failure it prevents, and what it cost. Where
a decision came from a defect found in production or in review, that is said
plainly — those are the ones worth reading.

---

## Gate the action, not the command string

**Decision.** Authorization attaches to what is being done, not to the text the
user typed.

**Why.** A rank restriction was applied to a command that returns an original
document. Asking for the same thing in Portuguese prose — "send me the original
file of document 2" — reached the same code path without the check, because the
check lived on the command string. The natural-language route was never a
second feature; it was a second door to the same room, and only one door was
locked.

**Cost.** Authorization must be resolved after intent is known, which means the
routing layer cannot be a thin `if` on the first token. Worth it: the class of
bug it removes is the class that matters.

---

## Fail closed on rank

**Decision.** Ranks are ordered and checked as "at least this rank". A missing,
unknown or corrupted role value resolves to the lowest rank.

**Why.** The alternative — treating an unrecognised value as a default mid-tier
role, or skipping the check when the value cannot be parsed — turns data damage
into privilege escalation. A record that has been corrupted is exactly the
record you least want to trust.

**Also.** Approving someone who already holds a rank preserves it. An earlier
version overwrote the role on re-approval, so re-approving an existing
administrator silently demoted them — a bug that was invisible until someone
noticed they had lost access they never gave up.

---

## Persist before acknowledging

**Decision.** The webhook writes to a durable queue before returning success.

**Why.** Anything done before the acknowledgement can be lost in a restart, and
the sender has no way to detect it. Retries and dead-lettering happen behind that
boundary, where they can be observed and replayed.

**Cost.** An extra write on the hot path, and a queue to operate. Cheap next to
losing a document a lawyer believes was delivered.

---

## Cache the answer and its evidence as one record

**Decision.** A cached answer and the sources that support it are stored and
restored together.

**Why.** They were stored separately once. A cache hit could then return a
conclusion whose supporting sources had expired, so the answer arrived confident
and unsupported — the exact failure mode this system exists to prevent. Storing
them apart made an invariant ("every answer carries its evidence") depend on two
independent lifetimes agreeing.

---

## Put identity into the cache key

**Decision.** The key includes the task, the client, the model, a schema
version, and a fingerprint of the source policy.

**Why.** With only the question as the key, internal corpus research and
external public research collided on the same string and served each other's
answers. Including the source-policy fingerprint means that widening or
narrowing the allowlist invalidates exactly the answers whose coverage changed,
and nothing else.

**And.** Anything time-sensitive — a docket lookup, a question asking what is
current — bypasses the cache entirely. Freshness is decided from the user's
original wording, not from the rewritten query, because the rewrite can drop the
very word that signalled urgency.

---

## Validate configuration at startup, not at import

**Decision.** Configuration is validated when the application boots, before it
opens the database. An incompatible provider/model pair is fatal.

**Why.** The environment declared one model; a failed provider lookup silently
substituted another. The firm believed it was running a model it was not, and
nothing in the logs contradicted that belief. Silent fallback is worse than a
refusal to start, because it produces confident output from an unintended
system.

**The lesson that cost the most.** The first version of this check ran as a
validator at module import. That made `import config` throw — which broke test
collection everywhere, and hid the very message that explained the problem.
*Failing at startup* means when the application starts, not when a module is
imported.

---

## Business rules as pure libraries

**Decision.** Procedural deadline counting is a pure function: no I/O, no
model, no hidden defaults.

**Why.** A deadline is either right or it is malpractice. The legal regime is a
required argument because a default would be silently wrong half the time, and
only national holidays ship by default — inventing a local court holiday
produces a wrong date that looks exactly as authoritative as a right one.

**Benefit.** It is exhaustively testable without a network, a database or a
model, so it is.

---

## Ask for the one missing detail

**Decision.** A command missing an argument asks for that argument and waits,
instead of printing usage.

**Why.** A usage line tells the person the one thing they already know. Worse,
it sends them out of the conversation to go find an identifier. The command now
lists the eligible options, waits for the reply, parses it deterministically,
and rewrites it into the equivalent command.

**The constraint that shaped it.** The model never resolves the identifier. The
reply is parsed by the same deterministic layer that parses the typed command,
so the authorization gate is identical on both paths. A waiting question expires,
can be cancelled, and never silently reinterprets a stale answer.

---

## Report health as a capability

**Decision.** Readiness distinguishes "cannot serve" from "serving with reduced
external research".

**Why.** Collapsing them means either a research outage takes down a system that
could still answer from the corpus, or a real outage hides behind a green check.
Health checks are configuration-only and never spend API quota, because a health
check that costs money per call gets disabled the first time someone reads the
bill.

---

## Type checking in the gate

**Decision.** A static type checker runs in CI alongside the linter, pinned to
an exact version.

**Why.** The linter in use is deliberately narrow and does not look at types. Two
classes of defect were already sitting in the test suite: an optional value
reaching a call that cannot accept `None`, and a dictionary of the wrong value
type unpacked into a settings model whose fields are numbers and enums. Both read
fine and fail at runtime.

**Pinned.** An unpinned checker adds rules on its own schedule and turns a green
branch red on a commit that changed nothing. Raising the version is its own
reviewable change.
