# 12 — Compute derived members on read inside a transaction

**Depends on:** 02, 06, 10, 11 (and therefore, transitively, 05 and 07–09).
**Reference:** `cli-test-fixes` — `recompute_def_in_txn` (`manager/mod.rs`
~line 1022) is the piece to **salvage**. `propagate_in_txn` (~line 908),
`refresh_remote_cross_deps_in_txn` (~line 985) and the `propagate_in_txn` call
in `assign` (~line 730) are the pieces **not** to reproduce.

## Rationale

A `def` is an eagerly evaluated, cached value, so a plain lookup inside a
transaction returns whatever the last committed propagation stored. Task 11
states the contract this breaks.

PR #189 solved it by **pushing**: `assign` walked the listener graph and
recomputed every dependent def into `txn.read_cache`. That was rejected. Its
entire durable product is cache entries and read locks — `recompute_def_in_txn`
writes to `txn.read_cache` and never to `txn.written`, so no def value is ever
committed by it, and `store_committed_writes` commits vars only. If the def is
never read, all of that work is discarded at commit. Worse, it is not cheap
work: a purely local write to `x`, where some `def y = x + remote.z` exists,
forces a network round trip and a remote read lock — per assignment, so once per
iteration of a loop.

**Pull instead.** Compute a def when it is read, under the transaction, from the
transaction's own view. Work and locks then scale with what the transaction
actually observes, and propagation is left to do what it is for: notifying
listeners outside the transaction.

Three consequences worth stating, because they are the point:

- Unread defs cost nothing and lock nothing.
- A def spanning a touched service and an unavailable one no longer aborts a
  transaction that never reads it. Note *how*: by not doing the work, not by
  weakening serializability. Do not reintroduce `dep_cache` fallback to achieve
  this.
- Pull evaluates in dependency order by construction, so glitch-freedom stops
  depending on the listener graph and a `changed` flag.

## Specification

### 1. Hook the read path

`Manager::lookup` currently resolves a local member by returning
`service.vars[name]` under a read lock. Defs live in `vars` too, so it cannot
currently tell them apart.

Inside a transaction, when the member is a def (`service.defs.contains_key`):

1. **Take the read lock on the def member itself**, exactly as the current code
   does before returning a stored value — `(service_net_id, name)` recorded in
   `txn.locked`. Do not drop this step on the grounds that §2 locks the
   dependencies: an `update` transaction write-locks **defs as well as vars**
   (`runtime/update.rs:141-147` puts both `Decl::VarDecl` and `Decl::DefDecl`
   into the lock group's write set), so without it a concurrent update can
   replace the def's *expression* while this transaction is evaluating and
   memoising it. Locking dependencies protects the inputs; this protects the
   definition.
2. Serve from `txn.read_cache` if present (see §3a and §3b on what
   invalidates it).
3. Otherwise recompute it (§2), insert the result into `txn.read_cache`, and
   return it.

Outside a transaction, behaviour is unchanged: return the stored value.

`remote_read_participant` already routes through `lookup`, so a participant
serving a transactional read of one of its defs gets this for free. Confirm with
`test_participant_transactional_read_sees_buffered_def`; do not add a second
path.

### 2. The recompute

Reuse the body of `recompute_def_in_txn` from the reference branch. It is most
of the work and each part of it is load-bearing:

- Evaluate with an **empty environment**, so every same-service dependency goes
  through `lookup` and takes a read lock. An earlier version seeded `env` from
  the service's members; `Expr::Variable` then resolves from `env` and never
  reaches `lookup`, which is the only thing that takes a lock. The members a
  derived value rests on are part of the transactional view it commits under, so
  under 2PL they must be held.
- **Install no `reactive_cache` at all** — set it to `None` for the duration,
  saving and restoring the outer value.

  This is where the reference branch's shape must not be copied. `MemberAccess`
  consults `reactive_cache` *before* `lookup`
  (`interpreter/evaluator.rs:301-309`), so anything left in that map is served
  without a read lock and without reaching the owner. The reference branch
  installs the def's `dep_cache` and then computes a `live` set to delete the
  entries it must not serve; under fully transactional semantics every
  cross-service entry is one it must not serve, so the whole mechanism reduces
  to installing nothing. Copying the install without the deletion would quietly
  serve stale pushed values and violate §3 and
  `test_remote_dependency_is_never_served_from_dep_cache`.

  The save/restore still matters, for the reason task 02 gives: evaluation
  awaits remote reads, and an inbound `Update` can re-enter the
  non-transactional `recompute_def` while this one is suspended. Leaving an
  *outer* recompute's cache installed would let it satisfy this def's member
  accesses.

  Memoisation is not lost: `lookup` serves from `txn.read_cache` (§3), which is
  the transactional cache and does carry locks.
- A recompute failure **aborts**. It must not be swallowed: that would commit a
  state whose derived member does not follow from it, and would drop a
  `WaitDieAbort` or `WaitOn` from a live re-read, defeating task 06's retry loop.

Inside a transaction, then, a cross-service dependency is **always** read from
its owner under the shared transaction id. `remote_lookup` pre-registers the
owner in `txn.participants`, so `Commit`/`Abort` releases the read lock it takes.

### 3. Memoisation and invalidation

Without memoisation, every read of a def costs one round trip per remote
dependency. Read locks make a remote read stable against *other* transactions,
so the cache in §1 is sound — with one exception, which is why `remote_lookup`
on the reference branch deliberately never caches: **this transaction's own
composed action** can write, on the owner, a member this transaction already
read.

There is a second, entirely local version of the same problem, and it is the
common case: a transaction reads `def y = x + 1`, then writes `x`, then reads
`y` again. `assign` buffers the write and updates `x`'s own cache entry, so the
memoised `y` from the first read is served unchanged and the transaction sees a
`y` that does not follow from its own `x`. Read-your-own-writes, broken by the
very cache that makes pull affordable.

#### 3a. Invalidate on local write

`assign`, inside a transaction, must **remove from `txn.read_cache` every def
transitively downstream of the member it wrote**, after buffering the write.

Walk `service.listeners` from the written member, transitively, and drop each
def's entry. This is the "mark dirty" half of the pull design, and the
distinction from the eager approach we rejected is the whole point: this walk
**evaluates nothing, takes no locks, issues no network reads and cannot fail**.
It is a `HashMap::remove` per dependent. Do not recompute here; the next read
will, if there is one.

Transitivity is required (`z` derives from `y` derives from `x`), as is
following listeners into other services on this node.

#### 3b. Invalidate on composed action

That is exactly what `touched_services` reports, so bring that plumbing back —
repurposed from "recompute these defs now" to "invalidate these cached reads":

- `ActionResponse.touched_services: Vec<String>` in `meerkat-lib/src/net/types.rs`,
  with `#[serde(default)]` so older peers still decode.
- `codec::validate_touched_services(&[String]) -> Result<()>`.
- `Manager::touched_services_for_txn(&TxnId) -> Vec<String>` — the **transitive**
  set: the union of `txn.remote_writes` and the services named in `txn.written`.
  Transitivity matters because a node the originator never contacted may have
  been written by a nested action, and nothing else reports it.
- `ComposedCall` (task 10) gains `touched_services`, so a **replayed** dispatch
  reports the same set as the original. A replay that reported a narrower set
  would invalidate less and serve a stale value.

On a successful `ActionResponse`, in `remote_action`:

1. **Merge the reported set into `txn.remote_writes`.** Nothing else populates
   it, and `touched_services_for_txn` reads it to report transitively upward —
   so without this merge a middle node reports only its own services and a
   write two hops down is invisible to the originator, whose cached def then
   stays stale. Validate with `validate_touched_services` before storing.
2. **An empty reported set means "unknown", not "nothing".** The field is
   `#[serde(default)]`, so a peer that does not send it decodes as empty.
   Fall back to recording the slug actually dispatched to, as
   `record_touched_services` does on the reference branch. Reading empty as
   "nothing was touched" turns a version skew into silent staleness.
3. **Invalidate**: drop from `txn.read_cache` every entry owned by a touched
   service, and every memoised def whose `graphs.cross_deps` name one.

**Never evict an entry that is also in `txn.written`.** Those are this
transaction's own buffered writes, mirrored into `read_cache` by `assign`; they
are authoritative and nothing a participant reports can invalidate them.
Dropping one sends the next read to the pre-commit `service.vars` value and
loses read-your-own-writes. The overlap needs a service to be both written here
and named in a report — which takes a composition cycle (A composes onto B,
which composes back onto A under the same id) — but the guard is one condition
and the failure is silent.

The **replay** branch from task 10 must do all of this too, from the recorded
`ComposedCall`. A replay that skipped the merge or the invalidation would leave
the transaction with a narrower touched set and a stale cache than the original
run produced.

Record the `ComposedCall` **before** the invalidation runs. The invalidation
path can read, and therefore can park, *inside* the `do` statement — which is
precisely the case deferred from task 10.

### 4. Cycle detection

Push terminated cascades with `recompute_def_in_txn`'s `changed` return value.
Pull recurses instead, so a cyclic def is infinite recursion rather than a
terminating fixpoint.

Add an in-progress set keyed by `(ServiceNetId, Symbol)`, entered before
evaluation and left after. Re-entering a def already in progress is an
`EvalError` naming the def.

**Put the set on `Transaction`, not on `Manager`.** A recompute awaits remote
reads, and `send_and_await_reply` pumps network events while it waits, so a
second transaction can begin evaluating the same def on this node while the
first is suspended. A manager-wide set would report that as a cycle — a
spurious, load-dependent failure on a perfectly acyclic program. `Transaction`
already owns the rest of the per-transaction evaluation state (`read_cache`,
`locked`, `written`), and this belongs with it.

This is a new failure mode, not a ported one; give it its own test.

### 5. Do not reintroduce

- `propagate_in_txn`
- `refresh_remote_cross_deps_in_txn`
- the `propagate_in_txn` call in `assign` — `assign` buffers the write into
  `txn.written` and `txn.read_cache` and does nothing else
- `txn.remote_writes` **as a recompute trigger**. It survives, populated and
  consumed as described in §3b, but only to drive invalidation and transitive
  reporting — never to decide what to recompute.

The non-transactional path is unchanged: defs remain stored values, and
commit-time `propagate` still recomputes and still notifies remote listeners.
Push outside a transaction, pull inside, is a deliberate asymmetry — say so in
the code, because the reference branch's parallel `propagate`/`propagate_in_txn`
naming will otherwise invite someone to restore the symmetry.

## Tests

1. **Un-ignore everything task 11 ignored.** That suite is the acceptance
   criterion. `test_remote_dependency_is_never_served_from_dep_cache` and
   `test_recompute_in_txn_read_locks_same_service_dependencies` are the two that
   most directly constrain this design.
2. **Restore the deferred test** from task 10:
   `test_a_park_inside_the_do_statement_does_not_re_send_the_child_action`.
   §3b gives it a mid-`do` park again.
3. **New — read, write, read again.** With `def y = x + 1`: read `y`, write
   `x`, read `y` again in one transaction; the second read reflects the write.
   This is the test for §3a, and it is the one that fails if memoisation is
   added without local invalidation. Add the transitive form too (`z` over `y`
   over `x`), because a non-transitive walk passes the two-level case.
4. **New — cycle detection:** a cyclic def read inside a transaction returns an
   error naming the def, and does not hang or overflow the stack.
5. **New — memoisation:** a def with a remote dependency, read twice in one
   transaction, issues one remote lookup.
6. **New — invalidation on a composed action:** the same def, read before and
   after a composed action that writes its remote dependency, returns the
   updated value on the second read. This is the test for §3b.
7. **New — a composed action does not evict this transaction's own writes.**
   Write a local member, compose an action, then read that member back; it must
   still be the buffered value. Covers the `txn.written` guard in §3b.
8. **New — transitive touched-services reporting.** In a `client -> mid -> rc`
   chain where only `rc` is written, a `client` def over `rc` must see the new
   value. Fails if §3b's merge into `txn.remote_writes` is missing.
9. `meerkat/tests/s1.mkt` passes.

## Notes

- Expect this PR to be large. It is the one place where that is the right
  answer: the mechanism has to change in one step, and everything it leans on
  has already landed.
- Performance to state in the PR description: first read of a def with N remote
  dependencies costs N round trips; subsequent reads in the same transaction
  cost nothing until a composed action touches one of those services. That is
  the price of fully transactional semantics and it was accepted deliberately.
